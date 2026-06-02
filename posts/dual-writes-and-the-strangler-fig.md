---
title: Dual writes and the strangler fig — how to move a live system without losing data
date: 2026-08-09
permalink: /dual-writes-and-the-strangler-fig
---

# Dual writes and the strangler fig — how to move a live system without losing data

[Last month I wrote about migrating a billing system from Rails to Go](/migrating-billing-rails-to-go). In that post I name-dropped two patterns I leaned on heavily — the [strangler fig](https://martinfowler.com/bliki/StranglerFigApplication.html) and dual writes — without actually unpacking either of them.

A few people asked me what I meant. Fair. So this is the companion post that drills into both. They're the two load-bearing patterns for moving a *live* system — one that's serving real traffic while you're rebuilding it underneath — and getting them wrong is the difference between a smooth migration and a Sunday night that no one wants to remember.

## The strangler fig, named properly

The pattern's name comes from a [2004 essay by Martin Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html). The metaphor: a strangler fig is a vine that wraps around a host tree, slowly grows down to the soil, and eventually replaces the host while the host is still standing. From the outside, the canopy never disappears. Inside, it's a totally different organism.

In software, the host is the old system. The vine is the new one. The facade — the stable surface clients keep pointing at — is what makes the substitution invisible.

The shape of a strangler fig migration:

```
Pre:                 Mid:                       Post:
┌─────┐              ┌──── facade ────┐         ┌──── facade ────┐
│ old │              │                │         │                │
└─────┘              ▼                ▼         ▼                ▼
                ┌─────┐          ┌─────┐    ┌─────┐         ┌─────┐
                │ old │          │ new │    │ new │         │ new │
                └─────┘          └─────┘    └─────┘         └─────┘
                  (still does       (some
                   most things)      things)
```

The facade is the part that's worth obsessing over. It is the part that lets you change the implementation per-route, per-user, per-percent. Without a facade, every cutover is a big-bang cutover — and a big-bang cutover is precisely what the pattern exists to avoid.

The facade can be a lot of different things: an API gateway, a smart reverse proxy, a feature-flag-aware service router, or a few branches in the application code itself. The exact mechanism matters less than the property: **clients keep talking to one stable thing, while the implementation underneath shifts**.

## What a strangler fig migration looks like, step by step

1. **Insert the facade in front of the old system.** Clients now go through the facade. The facade does nothing but forward to the old system. Verify in production that the facade is invisible — same latency, same responses, no behavioral change.
2. **Build a new service for one capability.** Pick the smallest, lowest-risk thing. Read-heavy. Well-bounded. (I covered the criteria in the [billing post](/migrating-billing-rails-to-go#phase-1--carve-a-boundary).)
3. **Run the new service in parallel, silently.** It receives traffic but doesn't serve it yet — see the dual-write section below.
4. **Cut over one slice at a time.** The facade starts routing that capability to the new service for *one customer*, then a handful, then a percentage by customer hash, growing weekly.
5. **Decommission the old code path.** When the facade routes 100% of that capability to the new service, the old code path is dead code. Log entries into it for two weeks to confirm. Then delete it.
6. **Repeat for the next capability.** Until the old system is empty. Then retire the old system.

The order is not negotiable. Step 1 happens before step 2 — you do not start building the new service until the facade exists and is invisible.

## Dual writes — what they are and what they aren't

When the new service exists but isn't being read from yet, it needs data. Otherwise it has nothing to serve and nothing to verify against.

That's where dual writes come in. A "dual write" is the umbrella term for **keeping the new system's state in sync with the old system while the old system is still authoritative**.

The naive picture:

```
client → facade ──┬──→ old (authoritative)
                  └──→ new (shadow)
```

The facade writes to both. Reads come from old (during shadow phase) or from new (during cutover). Old stays authoritative until you're convinced new is right.

The naive picture is also the trap. Let's see why.

## The dual-write atomicity problem

You cannot atomically write to two heterogeneous systems without [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol). And 2PC is, as I argued in the [outbox/saga post](/saga-and-outbox), the wrong tool for almost every real situation.

Without 2PC, "write to old, then write to new" has four outcomes:

| Old | New | Result |
|---|---|---|
| ✓ | ✓ | Fine. |
| ✗ | (skipped) | Facade returns error; nothing committed. Fine. |
| ✓ | ✗ | **Diverged.** Old has the row, new doesn't. |
| ✓ | ✓-then-✗ | Worse: new has it, then partially rolled back. State is ambiguous. |

The third row is the killer. It's the same shape of problem I described in the [transaction manager post](/go-transaction-manager-commit-errors) — a partial commit leaves you not knowing the world's state. Multiplied across thousands of writes a day during a migration, the divergence is silent and accumulates.

You cannot fix this with retries alone, because the new system might *have* received the write but failed to acknowledge it (network blip on the response). A retry from the facade then writes the same data twice.

The fix, structurally, is the same as the outbox fix: **don't synchronously dual-write across system boundaries.** Make one write atomic — to a system you control — and replicate from there asynchronously.

## The four topologies you'll actually use

In practice, "dual write" is one of four shapes. They have very different correctness properties.

### Topology 1: synchronous dual write at the facade

The naive picture above. Facade writes to old, then to new, returns success only when both succeed.

- **Latency** is the sum of both.
- **Atomicity** is broken — see the previous section.
- **Failure modes** are nasty.

Don't ship this. It looks tempting because it's three lines of code. The three lines of code are the bug.

### Topology 2: old writes to its DB + outbox; new consumes events

The shape I reach for whenever the old system is something I can modify:

```
client → facade → old (writes DB + outbox in one txn)
                            │
                  ┌─────────┘ (relay)
                  ▼
              broker
                  │
                  ▼
                new (applies event idempotently)
```

The old system writes its database row *and* an outbox row in a single local transaction. A relay process publishes the outbox rows to a broker. The new system consumes from the broker and applies the change to its own store, idempotently.

Why this works:

- **Atomicity is contained.** The old service has a single local transaction. Either both rows commit or neither does.
- **New is eventually consistent.** There's a window — usually seconds — where old has the new state and new doesn't. That's fine, because reads still come from old during the shadow phase.
- **Replays are safe.** The relay can publish a message twice; the new system absorbs the duplicate because its writes are idempotent. (This is where [idempotency keys](/idempotency-keys-deduped-writes) earn their keep at the consumer side.)
- **The old service doesn't need to know the new service exists.** The relay does.

This is what "dual writes" actually means in production, even when people say things that sound synchronous.

### Topology 3: CDC-based shadowing

When you can't modify the old service — vendor system, legacy code no one wants to touch, an external partner — you can read the old system's database write-ahead log via [Debezium](https://github.com/debezium/debezium) (or similar) and project the changes onto the new system.

```
old DB ──(WAL)──→ Debezium ──→ broker ──→ new
```

- **No application changes on old.** The cleanest property of this approach.
- **Schema coupling.** Debezium sees what changed at the table/column level, not at the semantic intent level. If your old service does a bulk `UPDATE` that means "rebill these customers", the CDC stream shows a flurry of column updates. The new service has to know how to interpret them.
- **DDL is dangerous.** A column rename on the old side, undeclared to the consumer, breaks the projection. Coordinate.

CDC is the right tool when you don't own the source. When you do own the source, an outbox is usually less coupled and easier to reason about.

### Topology 4: application-layer dual write

The variant I keep seeing people *try*:

```ruby
def create_invoice(attrs)
  ActiveRecord::Base.transaction do
    invoice = Invoice.create!(attrs)
    NewBillingService.create_invoice(invoice)  # ← HTTP call, inside the txn
    invoice
  end
end
```

Two reasons not to:

1. **The HTTP call is inside the transaction.** If the new service is slow, the old service's transaction holds row locks for as long as the call takes. Pathological under load.
2. **The atomicity is fake.** If the HTTP call succeeds and the local commit then fails, the new service has been told about an invoice the old service never finalized.

A pragmatic middle ground exists: write the local row in the transaction; enqueue a job that publishes to the new service; the job is idempotent and retried. That's just an outbox without the table. It works at small scale. It gets fragile fast.

If you find yourself reaching for topology 4, walk to topology 2 instead. It's not much more code and it's correct.

## Reads during dual write

The other half of the picture: where do reads go?

Three phases, in order:

1. **Shadow** — reads from old. New is receiving writes but its state is purely for verification.
2. **Verify** — reads from old, *and also* from new in the background, with the responses compared and the difference logged.
3. **Cutover** — reads from new, per-customer behind a feature flag.

The verify phase is where the bugs surface. Canonicalize responses before comparing (JSON field order, decimal formatting, timestamp timezones — all the boring stuff). I described the diff log format in the [billing migration post](/migrating-billing-rails-to-go#phase-2--shadow); it applies here verbatim.

If the diff is dirty, you do **not** progress to cutover. The whole point of running dual is that you can see disagreement before customers do.

## The instant-rollback property

Here's the thing people forget: dual writes during cutover aren't just for verification. They are the rollback story.

While you're dual-writing, **the old system is still receiving every write**. Its state is authoritative and current. If something goes wrong on the new side, you flip the feature flag, reads go back to old, and not a single customer's data is lost.

This is the property that makes a strangler fig migration *safe*. The vine doesn't kill the host until you're sure the vine can hold itself up.

Stop dual-writing too early — say, you "decommission" the old write path before you have confidence in the new one — and you've lost the rollback. Now any new-side bug is a real outage.

## When to stop dual-writing

Three signals, all required:

1. **You've been reading from new exclusively for N days, no rollbacks.** N is at least a week; for billing or other money-touching paths, longer.
2. **The verification diff has been clean for N weeks.** Including during peak traffic and at month boundaries (cron jobs, monthly closings, all the seasonal weirdness).
3. **No code path still reads from the old system for that capability.** Including background jobs, reports, internal dashboards, the support tooling, *everything*. If anything still reads from old, you can't turn off writes to old.

Only then: drop the dual write, retire the old write path, delete the code.

This phase has an expiry date attached. **Schedule the removal of the dual-write code as part of the plan that introduces it.** Otherwise it stays in your codebase for years. Dead code rots. Alarms fire for systems no one uses anymore. The dual-write became a footgun.

## Anti-patterns

A few patterns I've seen go wrong, with the failure mode:

**Dual writes without verification.** You write to both systems and pray they agree. They don't. The divergence is silent because no one is comparing. By the time you notice (usually months later, often via a customer support ticket), there's no way to know which side is "right" and you have to manually reconcile. Verification is not optional.

**Per-percent traffic split before per-customer split.** Each request is independently routed. A user sees inconsistent results across requests in the same session — their invoice list comes from new, their detail comes from old, the totals don't add up. Always slice by *customer*, never by *request*. Once you cut a customer over, every request from that customer goes to the same side. I keep saying this and it keeps mattering.

**Synchronous dual writes that double latency.** Even when atomicity is somehow handled, doing two HTTP calls per write doubles your write-path latency. At any meaningful scale this shows up in p99s and starts paging on-call. Async or outbox topology, always.

**Forgetting deletion semantics.** A delete on old needs to propagate to new. Does it? What about soft delete vs hard delete? What about tombstones in the event stream? Deletes are the most under-thought operation in dual-write designs. They're also the operation that silently drifts the systems farthest apart over time.

**No plan to remove the dual write.** The PR that adds the dual write should have a tracking ticket for its removal. The ticket should have a date attached. Otherwise the dual write becomes load-bearing for things no one remembers, and removing it years later is its own migration.

**Reverse-engineering the new system to match a bug in the old one.** During verification, you'll find diffs caused by *bugs in the old system*. The old system is "right" only in the sense that it's what customers see today. Sometimes the right answer is to fix the bug in both. Sometimes it's to bake the bug into the new system intentionally because customers depend on the bug-shaped behavior. Both are legitimate; what's not legitimate is finding the diff and not making a decision.

## How the pieces compose

Worth saying explicitly:

- **The strangler fig** gives you the topology — facade in front, capabilities replaced one at a time, old system retired at the end.
- **Dual writes** give you the data — new system stays hot while old is authoritative.
- **The outbox pattern** gives you the *correctness* of the dual write — atomic local commit + asynchronous replication.
- **Idempotency keys** give you the *safety* on the consumer — the new system can absorb duplicates without diverging.
- **Per-customer feature flags** give you the *cutover* — one slice at a time, instant rollback.
- **Verification (diff logs)** gives you the *confidence* — you can prove old and new agree before betting on new.

Take any one of those out and the migration gets dramatically more dangerous. Take all of them out and you're doing a big-bang rewrite and praying.

## Lessons I keep coming back to

A handful:

1. **The facade is the load-bearing piece, not the new code.** Build it first. Make it invisible. Earn the right to slot the new system in behind it.
2. **"Dual write" is not a primitive — outbox is the primitive.** Anyone who tells you to "just write to both" is glossing over the atomicity problem.
3. **Dual writes are also the rollback.** Don't stop dual-writing until you're sure you won't need to read from old again.
4. **Reads slice per-customer. Never per-percent.** Consistency within a session matters.
5. **Verification is what makes dual writes safe.** Without a diff log they're an expensive pretence.
6. **Schedule the removal at the same time as the introduction.** Otherwise the dual-write outlives its usefulness by years.

The strangler fig is a beautiful pattern in the same way a long Sunday hike is beautiful — slow, deliberate, more about the route than any single step. Take it seriously, build the facade first, dual-write through an outbox, verify obsessively, slice per-customer, and the host quietly disappears one capability at a time.

Mind your reads. Watch the diff. And take the dual-write code out when you're done.

## References

**Patterns**

- [Strangler Fig Application — Martin Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Strangler pattern (microservices.io)](https://microservices.io/patterns/refactoring/strangler-application.html)
- [Transactional outbox pattern (microservices.io)](https://microservices.io/patterns/data/transactional-outbox.html)
- [Saga pattern (microservices.io)](https://microservices.io/patterns/data/saga.html)
- [Two-phase commit protocol — Wikipedia](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)

**Tools mentioned**

- [Debezium](https://github.com/debezium/debezium) — change-data-capture for relational databases

**Related posts**

- [Migrating a billing system from Rails to Go](/migrating-billing-rails-to-go)
- [Saga and the outbox](/saga-and-outbox)
- [Idempotency keys — the request you can replay](/idempotency-keys-deduped-writes)
- [The Go transaction manager bug that ate my rollback](/go-transaction-manager-commit-errors)
