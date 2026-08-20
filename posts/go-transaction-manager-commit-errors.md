---
title: The Go transaction manager bug that ate my rollback
date: 2025-06-15
permalink: /go-transaction-manager-commit-errors
---

# The Go transaction manager bug that ate my rollback

I've been writing Go for a while now, and like every other Go shop on earth, somewhere in the codebase there is a function called something like `WithTx`, `RunInTx`, or `TransactionManager.Execute`. It opens a transaction, runs a callback, and commits at the end.

This post is about a subtle bug I shipped in my own implementation of the pattern. Everything looks correct, every test passes, and yet under specific failure modes the transaction silently leaks.

## The shape of the pattern

The shape I'm going to talk about looks like this:

```go
type TxManager interface {
    Do(ctx context.Context, fn func(ctx context.Context) error) error
}

type txManager struct {
    db *sql.DB
}

func (m *txManager) Do(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := m.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }

    ctx = ctxWithTx(ctx, tx)

    if err := fn(ctx); err != nil {
        _ = tx.Rollback()
        return err
    }

    return tx.Commit()
}
```

The idea is straightforward. A repository looks up the transaction from `ctx`, and if one is there, it uses it. If not, it falls back to the bare `*sql.DB`. The caller orchestrates a usecase:

```go
err := tx.Do(ctx, func(ctx context.Context) error {
    if err := repo.SaveOrder(ctx, order); err != nil {
        return err
    }
    if err := repo.DeductStock(ctx, order.Items); err != nil {
        return err
    }
    return nil
})
```

Now where's the bug?

## The bug: a commit can fail too

```go
return tx.Commit()
```

If `tx.Commit()` returns an error, what happens? The function returns the error. Good. But what state is the transaction left in?

In Postgres, a `COMMIT` that fails (for example, due to a [serialization failure under SERIALIZABLE](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE), a [deferred constraint](https://www.postgresql.org/docs/current/sql-set-constraints.html), or a network blip mid-flush) leaves the transaction *closed* on the database side. The client-side `*sql.Tx` is also done, so calling `Rollback()` on it after a failed commit just returns [`sql.ErrTxDone`](https://pkg.go.dev/database/sql#pkg-variables). So far, nothing leaks.

The bug is at the *caller*.

The caller wraps `tx.Do` like this:

```go
err := txManager.Do(ctx, func(ctx context.Context) error {
    // ... some work ...
    return nil
})
if err != nil {
    log.Error("transaction failed", "err", err)
    // assume the side effects didn't happen
    return err
}

// commit succeeded, do follow-up side effects
publishEvent(...)
```

Now imagine `fn` returned `nil`, but `tx.Commit()` returned an error. From the caller's point of view, they get an error back from `txManager.Do`. That's correct, but they don't know **whether the failure was inside `fn` (no rows written) or inside `Commit` (rows possibly written, or possibly not, depending on where in the commit pipeline the error happened)**.

That ambiguity is the bug.

## How the bug bit me

My original implementation looked roughly like this:

```go
func (m *txManager) Do(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := m.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if r := recover(); r != nil {
            _ = tx.Rollback()
            panic(r)
        }
    }()

    if err := fn(ctxWithTx(ctx, tx)); err != nil {
        _ = tx.Rollback()
        return err
    }

    if err := tx.Commit(); err != nil {
        _ = tx.Rollback() // (1) no-op, returns ErrTxDone
        return err        // (2) caller can't tell why this errored
    }
    return nil
}
```

1. The `tx.Rollback()` after a failed `Commit()` does nothing useful, and the swallowed `sql.ErrTxDone` made it look like I was handling things when I wasn't.
2. The error returned is just `err`, so there's no way for the caller to tell whether the application code failed or the commit did.

## The fix: classify the error

The fix is to wrap commit errors in a typed error, so the caller can act differently:

```go
var ErrCommit = errors.New("commit failed")

func (m *txManager) Do(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := m.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }

    var done bool
    defer func() {
        if !done {
            _ = tx.Rollback() // only rolls back if fn panicked
        }
    }()

    if err := fn(ctxWithTx(ctx, tx)); err != nil {
        done = true
        _ = tx.Rollback()
        return err
    }

    if err := tx.Commit(); err != nil {
        done = true
        return fmt.Errorf("%w: %w", ErrCommit, err)
    }

    done = true
    return nil
}
```

Now the caller can check:

```go
if err := txManager.Do(ctx, fn); err != nil {
    if errors.Is(err, ErrCommit) {
        // commit failed — state on the DB is "rolled back" but
        // *we don't know* whether peers (cache, search index)
        // observed any of our intermediate writes.
        // Treat the operation as failed-but-uncertain.
        return retryableErr(err)
    }
    return err
}
```

I find this distinction useful because it forces me to name the thing. "Commit failed" is not the same as "the business logic rejected this request" and lumping them together makes the recovery story muddier than it needs to be.

## A second bug that hid for weeks

There's a related bug that's easier to ship and harder to spot: silently swallowing the rollback error.

```go
defer func() {
    _ = tx.Rollback() // <-- swallowed!
}()
```

Why does it matter? Most of the time it doesn't. [`Rollback()`](https://pkg.go.dev/database/sql#Tx.Rollback) on an already-committed transaction returns `sql.ErrTxDone`, and that's fine. But the `database/sql` driver also returns errors from `Rollback()` if the connection has been killed mid-transaction. If the linter is happy and you're never logging it, you'll never know.

The shape that finally satisfied me:

```go
defer func() {
    if rbErr := tx.Rollback(); rbErr != nil && !errors.Is(rbErr, sql.ErrTxDone) {
        logger.Warn("rollback returned non-trivial error", "err", rbErr)
    }
}()
```

Loud enough to find in logs, quiet enough to not page anyone in the night.

## Lessons I keep relearning

1. **`Commit()` is a write.** Treat it like one. It can fail, and the failure mode is different from "business logic failed".
2. **Don't `Rollback()` after `Commit()` errored.** It just returns `ErrTxDone` and pollutes the call site with a useless line of code.
3. **Wrap commit errors in a sentinel.** It makes recovery code possible to write.
4. **Don't silently swallow `Rollback()` errors.** Most of them are fine. The ones that aren't are the ones you want to see.
5. **Tests rarely exercise commit failure.** If you care about the path, build a fake driver that can be told to fail at commit time. A `httptest.Server`-style escape hatch for `database/sql/driver` does wonders.

Going forward, I default to a transaction manager that exposes a clear `ErrCommit` and logs `Rollback()` non-trivially.

Happy committing, and rolling back!

## References

- Go `database/sql` package: [`pkg.go.dev/database/sql`](https://pkg.go.dev/database/sql)
- `sql.ErrTxDone`: [`pkg.go.dev/database/sql#pkg-variables`](https://pkg.go.dev/database/sql#pkg-variables)
- `Tx.Commit`, `Tx.Rollback`: [`pkg.go.dev/database/sql#Tx`](https://pkg.go.dev/database/sql#Tx)
- `errors.Is` / `errors.As` / `%w` wrapping: [`pkg.go.dev/errors`](https://pkg.go.dev/errors), [`fmt.Errorf`](https://pkg.go.dev/fmt#Errorf)
- PostgreSQL `SERIALIZABLE` and SSI: [`postgresql.org/docs/current/transaction-iso.html`](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE)
- PostgreSQL deferred constraints (`SET CONSTRAINTS`): [`postgresql.org/docs/current/sql-set-constraints.html`](https://www.postgresql.org/docs/current/sql-set-constraints.html)
