---
title: MySQL transactions — same Go pattern, different ways to lose
date: 2025-06-29
permalink: /mysql-transactions-silent-rollback
---

# MySQL transactions — same Go pattern, different ways to lose

Two weeks ago I wrote about [the Go transaction manager bug that ate my rollback](/go-transaction-manager-commit-errors). The whole post was Postgres-flavored: deferred constraints, serialization failures at commit, the `ErrCommit` sentinel pattern.

A few readers came back with a reasonable question: *does MySQL behave the same?*

The short answer is mostly yes at the Go level, and very much no in the failure modes that actually fire. There's also one MySQL-specific footgun the Postgres-centric story doesn't cover, and it's the one that has bitten me the hardest.

This is the follow-up.

## At the `database/sql` level, the pattern is identical

The fix from the previous post (wrap commit errors in a typed sentinel, don't bother rolling back after a failed commit) works unchanged on MySQL. The Go layer doesn't distinguish drivers:

- `tx.Commit()` returning an error marks the `*sql.Tx` as done.
- A subsequent `tx.Rollback()` returns `sql.ErrTxDone`, useless cleanup, same as Postgres.
- The `ErrCommit` wrapper still lets the caller distinguish "the business logic rejected this" from "the commit failed and the world is ambiguous".

So if you have the pattern from last post, you don't have to delete it for MySQL. Good.

```go
if err := tx.Commit(); err != nil {
    done = true
    return fmt.Errorf("%w: %w", ErrCommit, err)
}
```

This line works the same in both worlds.

## The causes of a failed COMMIT diverge

Where the two databases actually differ is in *why* `Commit()` would fail. The Postgres list I gave was:

> a serialization failure, a deferred constraint, or a network blip mid-flush

In MySQL, two of those three barely apply.

**Deferred constraints don't exist.** [InnoDB checks foreign-key and uniqueness constraints per-statement](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html). You'll never get the Postgres "I built up state for the whole transaction and then a constraint violation surfaced at COMMIT" surprise. If a constraint is going to fail, it fails when the offending `INSERT` or `UPDATE` runs, not at the end.

This is actually a relief, one fewer thing to handle at commit time.

**Serialization failures at COMMIT don't really happen.** [Postgres SERIALIZABLE is optimistic](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE): the database lets transactions proceed, tracks read/write sets, and surfaces conflicts at commit (`SQLSTATE 40001`, "could not serialize access"). [MySQL SERIALIZABLE is *lock-based*](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable): it converts plain reads into `SELECT ... LOCK IN SHARE MODE`, and conflicts manifest as **lock waits or deadlocks during statements**, not as a COMMIT error.

So the "I tried to commit and the DB refused because someone else's writes invalidated mine" failure mode that occasionally surfaces on Postgres doesn't surface on MySQL the same way. MySQL surfaces those conflicts earlier and more bluntly.

**Network or server failures mid-commit.** This is the one that's symmetric. A connection dropped between sending COMMIT and reading the OK packet leaves you in the same ambiguous state on both engines: the write may or may not have landed; the client cannot tell. The `ErrCommit` sentinel earns its keep here.

So in practice, on MySQL, `tx.Commit()` errors are dominated by connection-level failures rather than DB-side correctness checks. Rare, but real, and the handling is the same.

## The MySQL footgun that *isn't* in the Postgres playbook

This one took me weeks to track down the first time.

**MySQL silently rolls back the entire transaction on certain mid-statement errors, not just the failing statement.**

The offenders:

- [`ER_LOCK_DEADLOCK`](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html#error_er_lock_deadlock) (error 1213): [InnoDB detected a deadlock](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html) and unilaterally killed one of the transactions. *Yours*, in this story.
- [`ER_LOCK_WAIT_TIMEOUT`](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html#error_er_lock_wait_timeout) (error 1205): your transaction waited longer than [`innodb_lock_wait_timeout`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout) for a lock. Depending on [`innodb_rollback_on_timeout`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_rollback_on_timeout), this may roll back just the statement or *the whole transaction*. The default depends on the MySQL version, and you should not rely on either.

When one of these fires, the transaction *is already gone server-side* before your Go code even sees the error. Your next statement on the same `tx` runs against… what, exactly? In the worst case, it runs in autocommit mode and writes data that was supposed to be part of the rolled-back transaction.

The shape that bites:

```go
err := tx.Do(ctx, func(ctx context.Context) error {
    if err := repo.UpdateBalance(ctx, src, -100); err != nil {
        return err // ← deadlock; server has rolled back the tx already
    }
    return repo.UpdateBalance(ctx, dst, +100) // ← runs in a fresh implicit transaction
})
```

If `UpdateBalance` on `src` deadlocked and the error path naively returned, you'd be fine, because the rollback in the `Do` deferral is a no-op against an already-dead transaction. But if your `fn` swallowed the error and continued, you'd write to `dst` *outside* the original atomic boundary. Money would move one way and not the other.

(I have not personally done this with money. I have done it with something only slightly less embarrassing.)

## The fix: detect and retry the *whole* transaction

The pattern on MySQL has to include a retry, and the retry has to be at the transaction boundary rather than the statement.

A starter implementation:

```go
import (
    "errors"
    "time"

    mysqldriver "github.com/go-sql-driver/mysql"
)

func isMySQLRetriable(err error) bool {
    var mErr *mysqldriver.MySQLError
    if !errors.As(err, &mErr) {
        return false
    }
    switch mErr.Number {
    case 1213, // ER_LOCK_DEADLOCK
         1205: // ER_LOCK_WAIT_TIMEOUT
        return true
    }
    return false
}

func (m *txManager) DoWithRetry(
    ctx context.Context,
    maxAttempts int,
    fn func(ctx context.Context) error,
) error {
    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        lastErr = m.Do(ctx, fn)
        if lastErr == nil {
            return nil
        }
        if !isMySQLRetriable(lastErr) {
            return lastErr
        }
        // Exponential backoff with a small jitter cap.
        sleep := time.Duration(attempt) * 10 * time.Millisecond
        select {
        case <-time.After(sleep):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return lastErr
}
```

**The retry happens at the `Do` boundary, not inside `fn`.** A fresh transaction means a fresh `*sql.Tx`, a fresh connection from the pool, and `fn` runs cleanly from the start. Retrying *inside* `fn` against the same `tx` won't work, because the tx is dead.

**`fn` must be idempotent across retries.** This is the part that catches people. If `fn` writes a row whose ID is allocated externally (a UUID generated in Go before the call), retrying produces the same row twice, but the first write was rolled back, so the second write succeeds, and you're fine. If `fn` has an external side effect like "send an email" or "call an external API" embedded *inside* the transaction, retrying duplicates that side effect.

Rule of thumb I follow: **side effects that aren't reversible don't go inside a `DoWithRetry`.** They go after a successful commit, behind something idempotent. The [outbox pattern from the saga post](/saga-and-outbox), where you write the side effect as a row inside the transaction and publish it from a relay outside, is the cleanest way to keep `fn` retriable.

## What's symmetric between Postgres and MySQL

For completeness, the retry pattern isn't MySQL-only. Postgres has its own retriable codes (full table in the [Postgres error codes appendix](https://www.postgresql.org/docs/current/errcodes-appendix.html)):

| Code | Meaning |
|---|---|
| `40001` | serialization_failure (true OCC conflict) |
| `40P01` | deadlock_detected |

Postgres deadlocks are also auto-rolled-back. The retry-the-whole-transaction principle is identical. The difference is just the menu of error codes you look for.

A generic `isRetriable` that handles both can be as simple as:

```go
func isRetriable(err error) bool {
    if isMySQLRetriable(err) {
        return true
    }
    if isPostgresRetriable(err) { // pgconn.PgError on SQLSTATEs 40001/40P01
        return true
    }
    return false
}
```

And then your `DoWithRetry` is the same function on both engines.

## A tangential MySQL durability note

This one surprised me the first time. MySQL has [`innodb_flush_log_at_trx_commit`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit):

- `=1` (default): fsync per commit. Strong durability, slower.
- `=2`: write to OS cache per commit, fsync once per second. A `Commit()` can return "success" but lose the transaction on a power failure within that second.
- `=0`: write and fsync once per second. Even more relaxed.

Postgres has a parallel knob ([`synchronous_commit`](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT)), but in MySQL it's a frequent performance tuning target for write-heavy workloads, and it changes the meaning of "the COMMIT succeeded". If your app reports an order as confirmed after `tx.Commit()` returns nil, and the DB is running at `=2`, that confirmation can disappear on a server crash. Not a bug you can fix in Go. Worth knowing exists.

## Lessons that carry over

A handful of additions to the original post's list, MySQL-specific:

1. **A failed statement can take the whole transaction with it.** On MySQL, deadlocks and lock-wait-timeouts roll back the *entire transaction* server-side. Treat them as transaction-level errors.
2. **Retry at the transaction boundary, not the statement.** Always with a fresh `Do`.
3. **Make `fn` idempotent or it doesn't get retried.** This forces side effects out of the transaction and into the outbox.
4. **Detect the error codes explicitly.** Don't retry on every error, only the ones you know are retriable. Anything else gets returned unmodified.
5. **Know your `innodb_flush_log_at_trx_commit`.** "COMMIT succeeded" means something slightly different at each setting.

The shape of the fix from the original post is unchanged: `ErrCommit`, no useless rollback after commit, loud-but-not-paging on non-trivial rollback errors. What changes is the retry layer you build around it. On Postgres you can sometimes get away without one (serialization failures are rare for most workloads). On MySQL, anything with realistic contention will deadlock occasionally, and the retry layer is part of the contract.

Same pattern. Different failure modes. Same lesson: name the thing, classify the error, and don't pretend the database is just a key-value store with extra steps.

Happy committing, on either engine.

## References

**Go**

- `database/sql` package: [`pkg.go.dev/database/sql`](https://pkg.go.dev/database/sql)
- `sql.ErrTxDone`: [`pkg.go.dev/database/sql#pkg-variables`](https://pkg.go.dev/database/sql#pkg-variables)
- `go-sql-driver/mysql` `MySQLError` type: [`pkg.go.dev/github.com/go-sql-driver/mysql#MySQLError`](https://pkg.go.dev/github.com/go-sql-driver/mysql#MySQLError)
- `pgx` / `pgconn` `PgError` type: [`pkg.go.dev/github.com/jackc/pgx/v5/pgconn#PgError`](https://pkg.go.dev/github.com/jackc/pgx/v5/pgconn#PgError)

**MySQL / InnoDB**

- InnoDB deadlocks overview: [`dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html`](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
- Server error reference (1205 / 1213): [`dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html`](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html)
- `innodb_lock_wait_timeout`: [InnoDB parameters](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- `innodb_rollback_on_timeout`: [InnoDB parameters](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_rollback_on_timeout)
- `innodb_flush_log_at_trx_commit`: [InnoDB parameters](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)
- InnoDB transaction isolation levels: [`dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

**PostgreSQL**

- Transaction isolation (incl. SERIALIZABLE / SSI): [`postgresql.org/docs/current/transaction-iso.html`](https://www.postgresql.org/docs/current/transaction-iso.html)
- Error codes appendix (SQLSTATEs incl. 40001, 40P01): [`postgresql.org/docs/current/errcodes-appendix.html`](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- `synchronous_commit` WAL setting: [`postgresql.org/docs/current/runtime-config-wal.html`](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT)
- Deferred constraints (`SET CONSTRAINTS`): [`postgresql.org/docs/current/sql-set-constraints.html`](https://www.postgresql.org/docs/current/sql-set-constraints.html)
