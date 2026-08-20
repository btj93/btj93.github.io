---
title: Saga and the Outbox — orchestrating multi-step writes without 2PC
date: 2026-03-15
permalink: /saga-and-outbox
---

# Saga and the Outbox — orchestrating multi-step writes without 2PC

A while back I wrote about [idempotency keys](/idempotency-keys-deduped-writes), which solve one specific problem: a client retried a write because they didn't know whether the first one landed. That post punted on the harder cousin of the problem, **doing several writes, across several systems, atomically.**

This is that post. It's about why "just use two-phase commit" is rarely the answer, what people actually do instead (the Outbox pattern and the Saga pattern), and how the two compose. I've been thinking about this a lot recently because I keep running into the same shape: a request that needs to update my database *and* send something to an external system (a queue, an email service, a webhook). I want both, or neither, and I want the system to recover sanely if one of them fails.

## The shape of the problem

Concretely: you accept an HTTP request to create an order. To do this you need to:

1. Insert a row into the `orders` table.
2. Decrement stock in the `inventory` table.
3. Publish an `order_created` event to a message queue (so the warehouse service can ship it).
4. Send a confirmation email to the customer.

If step (1) and (2) live in the same database, you can wrap them in a `BEGIN ... COMMIT` block and they're atomic with respect to each other. Fine.

But (3) and (4) talk to systems *outside* your database. The queue doesn't know about your transaction. The email service definitely doesn't.

- The database commits, then the process crashes before publishing. Your `orders` table has a row, but the warehouse never hears about it.
- You publish to the queue first, *then* try to commit the database, and the commit fails. Now the warehouse is preparing to ship an order that doesn't exist.
- You succeed at the database and the queue but the email send returns a 500. You retry the whole flow and now there are two orders.

Each of these is a real outage waiting to happen, and the moment your business has any real volume, all three will happen monthly.

## Why people say "just use 2PC" and why I disagree

The classical computer-science answer is [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol): have a transaction coordinator prepare all the resource managers, then either commit them all or roll them all back.

Why it's a tempting answer:

- It's conceptually clean. One protocol, atomic across heterogeneous systems.
- The XA spec exists. JTA implements it. Postgres has prepared transactions. You can do this.

Why it's almost never the right answer in practice:

- **Most of the systems you want to participate in 2PC don't support it.** Your email provider's REST API has no concept of "prepare". Your message broker probably doesn't either (Kafka especially does not). 2PC is only available when every participant implements the protocol.
- **It's blocking.** Resources held in the "prepared" state lock rows. If the coordinator goes down between prepare and commit, those locks can hang for an embarrassingly long time. (Search for "in-doubt transactions" if you want to ruin your weekend.)
- **It assumes a synchronous, trusted network.** Latency and partitions are the operating conditions.

So 2PC goes to the bottom of the toolbox.

## The Outbox pattern: solve the database/queue boundary

The [Outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html) fixes one specific subset of the problem: **"I need to write to my database AND publish a message, atomically."**

The trick is simple. Instead of writing to the database *and* publishing, you write *both into the database*:

```
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (
      id, aggregate_type, aggregate_id, event_type, payload, created_at
  ) VALUES (
      gen_random_uuid(), 'order', :order_id, 'order_created', :payload, now()
  );
COMMIT;
```

A separate process (call it the outbox relay) reads from the `outbox` table and publishes each row to the queue. After a successful publish, it marks the row as sent (or deletes it).

Now your atomic unit is the database transaction, and the database is something you already trust to be atomic. The queue receives a message *if and only if* the order was written.

A few things this pattern needs to get right:

1. **At-least-once delivery.** The relay can crash after publishing but before marking the row sent. It'll publish the same row again. Consumers must be idempotent. (Hi, idempotency keys.)
2. **Order preservation.** If you care about the order in which events are published per aggregate, the relay must read in `created_at` order *per* `aggregate_id`. Don't trust insertion order across aggregates.
3. **Backpressure.** A burst of orders writes a burst of outbox rows. The relay needs to keep up; if it falls behind, lag grows. Monitor outbox table size.
4. **Garbage collection.** Don't grow the outbox table forever. Either delete sent rows or move them to a cold archive.

The implementation can be simple. The first one I wrote in production was a goroutine that did [`SELECT ... FOR UPDATE SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html) `LIMIT 100 WHERE sent_at IS NULL`, published each row, then `UPDATE ... SET sent_at = now()`. With a single instance running you get a stream of events with the same atomicity as the database itself.

For higher throughput you can move to log-based delivery: let a CDC tool ([Debezium](https://github.com/debezium/debezium), etc.) read the database WAL and publish for you. Same idea, more plumbing.

## The Saga pattern: solve the cross-service boundary

The Outbox is great when the second step is "publish a message". It's silent on the harder case: **"I need to do two business operations across two services, and if the second one fails I need to undo the first."**

For that, you reach for [sagas](https://microservices.io/patterns/data/saga.html).

A saga is a sequence of *local* transactions, each in its own service or database, with a *compensating action* defined for each step. If step N fails, you run the compensations for N-1, N-2, ... back to 1. There's no global transaction; instead there's a defined recovery procedure.

For the order example, a saga might look like:

```
Step 1: Reserve inventory.
  Forward:    UPDATE inventory SET reserved = reserved + qty WHERE ...
  Compensate: UPDATE inventory SET reserved = reserved - qty WHERE ...

Step 2: Charge payment.
  Forward:    POST /payments {amount, customer_id}
  Compensate: POST /payments/{id}/refund

Step 3: Create order.
  Forward:    INSERT INTO orders (...) VALUES (...)
  Compensate: UPDATE orders SET status = 'cancelled' WHERE id = ...

Step 4: Send confirmation email.
  Forward:    POST /emails {to, template, data}
  Compensate: POST /emails {to, template: 'order_cancelled', data}
```

There are two orchestration styles:

- **Orchestration.** A central service knows the whole sequence and drives it step by step. Easy to read, easy to test, easy to debug. Bad when the central service becomes a bottleneck or a single point of failure.
- **Choreography.** Each service publishes events; other services listen and react. No central coordinator. More resilient to a single service going down. *Way* harder to reason about, because the saga's structure is only visible if you map who listens to what.

I default to orchestration unless I have a specific reason not to. Reading a saga as a sequence of `do/undo` pairs in one file is much easier than tracing it across seven event handlers.

## Compensations are not rollbacks

Rollbacks pretend the operation never happened. Compensations *acknowledge that it did* and apply a counter-operation. The difference matters:

- A refund is not the absence of a charge. The customer's statement will show *both* the charge and the refund. (This is fine, it's how the world works.)
- A "cancel email" is sent *after* the confirmation email. The customer sees both.
- An unreserve of inventory after a reserve is fine *if no one else reserved the same units in between*. If they did, you may have temporarily oversold, and you have to deal with it at the application level.

Designing compensations is harder than it sounds, and a lot of sagas get this wrong by assuming the world stayed still between step N and the compensation of step N.

## How the two compose

The Outbox and the Saga sit at different layers of the stack:

- **Outbox** solves "this service's database write and this service's published event must be atomic." It's a single-service pattern.
- **Saga** solves "these operations across multiple services must either all succeed or be compensated." It's a multi-service pattern.

In a system that does both, the picture looks like this:

```
   ┌──── Order Service ─────────────┐
   │  BEGIN                         │
   │    INSERT orders ...           │
   │    INSERT outbox (saga_start)  │
   │  COMMIT                        │
   └─────────────┬──────────────────┘
                 │ (outbox relay)
                 ▼
            ┌──────────┐
            │ Message  │
            │  Broker  │
            └────┬─────┘
                 │
         ┌───────┴────────┐
         ▼                ▼
   ┌───────────┐    ┌──────────────┐
   │ Inventory │    │   Payment    │
   │  Service  │    │   Service    │
   └─────┬─────┘    └──────┬───────┘
         │                 │
         │  (each writes   │
         │   to its own    │
         │   outbox on     │
         │   completion)   │
         ▼                 ▼
```

Each service uses the outbox pattern internally to publish its step's result. The saga's structure lives in the orchestrator (or, in choreography, in the choice of who listens to what). Compensations follow the same publish path as forward steps.

## Why I keep coming back to these

In practice the *distributed* part of distributed systems is a small minority of the code, but it's almost all of the operational pain. The code that does the actual business logic (placing the order, charging the card) is straightforward. The code that handles "what if the email service was down for 12 seconds" is the code that wakes you up at 3am.

Outbox and saga, together, give you a vocabulary and a structure for that part of the system. You don't need 2PC. You need:

- Local transactions that are *truly* atomic (i.e. one database).
- Idempotent consumers (so retries are safe).
- A clear, testable list of compensations for each step.
- Observability into the saga state (where in the sequence are we? are any sagas stuck?).

Get those right and most of the multi-system writes you ever do are tractable.

The next time I hit "I need to update my DB and call an external API atomically" I'll be writing an outbox table instead of looking for an XA driver.

Happy committing.

## References

- [Saga pattern (microservices.io)](https://microservices.io/patterns/data/saga.html)
- [Transactional outbox pattern (microservices.io)](https://microservices.io/patterns/data/transactional-outbox.html)
- [Two-phase commit protocol — Wikipedia](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
- [Debezium](https://github.com/debezium/debezium), change-data-capture for relational databases
- [PostgreSQL `SELECT ... FOR UPDATE SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html), the locking clause
- Related: [Idempotency keys](/idempotency-keys-deduped-writes)
