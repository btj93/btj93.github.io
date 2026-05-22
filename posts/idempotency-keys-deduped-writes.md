---
title: Idempotency keys — the request you can replay
date: 2025-12-21
permalink: /idempotency-keys-deduped-writes
---

# Idempotency keys — the request you can replay

It's the end of the year, I've been ramping down on building, and I've been doing more reading than typing. One topic I keep coming back to is **idempotency** — specifically, the pattern of accepting an `Idempotency-Key` header on write endpoints so a client can safely retry without producing duplicate side effects.

It's a deceptively simple idea. The first time I sketched it on a whiteboard I thought I'd finished in five minutes. Two months later I was still patching edge cases. This post is the version of the explanation I wish I'd had then.

## The problem, plainly

You have an HTTP endpoint that does something with a side effect — charge a card, send an email, create an order. A client calls it. Then:

- the request times out before the response comes back,
- or the connection drops,
- or the load balancer hiccups,
- or the client's process crashes and a watchdog retries.

The client doesn't know whether the request succeeded. It retries. If your server is naïve, you've now done the side effect twice.

The fix from the client's side is to attach a unique key per *logical request*. The fix from the server's side is to use that key to recognize a retry and return the *original* response instead of executing again.

That key is the idempotency key.

## What "the same response" really means

The naïve mental model: "if a key matches a previous request, return what the previous request returned."

The actual model needs more nuance:

1. **The same key with the same request body** → return the cached response.
2. **The same key with a *different* request body** → reject with `422`. The client is using the key wrong; do not silently coalesce.
3. **The same key while the original is still in flight** → don't both execute. The second one should either wait for the first or be rejected with `409 Conflict`.

Number three is the one people skip. It's the one that matters.

## A simple table schema

The minimum viable backing store is a single table:

```sql
CREATE TABLE idempotency_records (
    key             TEXT        PRIMARY KEY,
    request_hash    TEXT        NOT NULL,
    state           TEXT        NOT NULL CHECK (state IN ('in_flight', 'completed')),
    response_status INTEGER,
    response_body   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX ON idempotency_records (created_at);
```

`request_hash` is a hash of the request body (or the request body plus selected headers). It's how you detect "same key, different body". `state` is how you detect in-flight duplicates.

## The flow

The flow at request time:

```
1. INSERT (key, hash(body), state='in_flight') ON CONFLICT DO NOTHING.
2. If the insert happened (i.e. no conflict), we own the key. Execute the operation.
   On success: UPDATE state='completed', response_status, response_body.
   On failure: DELETE the row, so the next retry can attempt fresh.
3. If the insert conflicted, fetch the existing row.
   - If hash matches and state='completed': return the cached response.
   - If hash matches and state='in_flight': return 409, or wait, depending on your contract.
   - If hash mismatches: return 422 (key reuse on a different request).
```

Three of those branches are subtle:

**"Delete the row on failure" feels scary.** It is. The first time I implemented this, I treated failed operations as terminal — once a key sees a 500, that key is poisoned forever. Users hated it. The compromise I landed on: distinguish *retriable* failures (network blip to upstream, transient DB error) from *terminal* failures (validation error, business rule rejection). Retriable → delete the row, let the client retry. Terminal → mark `completed` with the error response, because the next call will get the same error and at least it'll be fast.

**"Return 409 for in-flight" feels wrong.** Doesn't this defeat the point of idempotency? No — it prevents the *server* from running the same job concurrently. The client can retry the 409 a few hundred milliseconds later, by which time the original will have completed (or timed out and been cleaned up). The point is to dedupe *effects*, not to magically replay in-flight requests.

**"Hash the request body" needs care.** Two requests with identical *semantic* bodies but different JSON field ordering should be treated as the same. So canonicalize before hashing: sort keys, normalize whitespace, drop fields you don't care about (like `request_id`).

## The race the database has to win

The whole thing hinges on step 1 being atomic. Two concurrent requests with the same key must result in exactly one winner of the insert. That's why it's [`INSERT ... ON CONFLICT DO NOTHING`](https://www.postgresql.org/docs/current/sql-insert.html) (Postgres) and not "SELECT then INSERT".

A useful sanity check: if your database doesn't support a single-statement "insert if not exists with row-level locking", you can't implement this correctly with that database. Don't reach for `SELECT ... FOR UPDATE` followed by `INSERT` — there's a window between the two where another transaction can insert under you.

In Postgres specifically, `INSERT ... ON CONFLICT (key) DO NOTHING RETURNING *` is the magic. If the returning set is empty, you lost the race; fetch the existing row. If it has a row, you won; you own the key.

## Scope and expiration

Idempotency keys can't live forever.

- **Scope per account / API client.** Two different customers can legitimately use the same key string. The primary key should be `(account_id, key)`, not `key`.
- **Expire after some window.** 24 hours is a reasonable default for most APIs. [Stripe uses 24 hours](https://docs.stripe.com/api/idempotent_requests); that's not a coincidence. Long enough to cover any reasonable retry window; short enough that the table doesn't grow without bound.
- **Garbage-collect with a periodic job, not a `DELETE` on every request.** Per-request cleanup adds latency to every write.

```sql
DELETE FROM idempotency_records
WHERE created_at < now() - interval '24 hours';
```

Run it every hour. Done.

## What it does NOT solve

Important — this pattern handles **duplicate writes to one service**. It does not handle:

- **Distributed atomicity across services.** Charging a card *and* sending a confirmation email atomically is a different problem. (Look up "outbox pattern" or "saga". I'll write about those soon.)
- **Concurrent updates to a single resource.** "Two users editing the same record at the same time" is an optimistic-concurrency problem, not an idempotency one.
- **Replay attacks.** If an attacker observes a request, they can re-use the key — but that's not a feature, that's the attacker forging a request. Idempotency assumes the key is a *correctness* tool, not a *security* tool.

People conflate these regularly. Idempotency keys solve one specific problem: a client retried because it didn't know whether the original landed. Don't try to make them do anything else.

## The "what if the response is huge" trap

A trap I fell into early: storing the full response body in the idempotency record.

That works fine when responses are small. It breaks when one of your endpoints returns a paginated list of 500 items, and now every retry is duplicating those bytes into the idempotency table. The table grows fast. Backups grow faster.

Two ways out:

1. **Cap the cached response size.** If the response is over some limit, mark the record as "completed but body not cached" and have the retry re-execute. As long as the operation is genuinely idempotent server-side, the second execution returns the same answer.
2. **Reference, don't copy.** Store an ID that points to the result (e.g. "order_id"), and let the retry re-fetch the body from the order table.

(1) is simpler. (2) is more correct. I usually start with (1) and graduate to (2) if the endpoint earns it.

## Lessons

A few things that I keep relearning, and that this post exists to keep me from forgetting:

1. **Idempotency is about deduping side effects, not magically replaying responses.** The "return the cached response" part is convenient. The "don't double-execute" part is the point.
2. **The atomic insert is the entire correctness argument.** If you don't have it, you don't have idempotency. You have a race.
3. **In-flight conflicts deserve a real status code.** `409` is for that. Don't let two copies of the same job run concurrently because you couldn't be bothered to handle the case.
4. **Distinguish retriable from terminal failures.** Poisoning a key on a network blip is a user-hostile bug.
5. **Hash a *canonicalized* request body.** JSON field order is not stable.
6. **Cap cached response size.** Or learn to love big tables.

I keep coming back to this pattern because the moment you have a write endpoint that can be called from a flaky network — which is every write endpoint, ever — you need it. And the moment you need it, you really need it.

Happy retrying.

## References

- Stripe — [Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- PostgreSQL — [`INSERT ... ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html)
- HTTP 409 Conflict — [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/409)
- HTTP 422 Unprocessable Content — [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/422)
- Related — [Saga and the outbox](/saga-and-outbox)
