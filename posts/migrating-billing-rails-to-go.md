---
title: Migrating a billing system from Rails to Go — process, phases, and what nearly broke us
date: 2026-05-12
permalink: /migrating-billing-rails-to-go
---

# Migrating a billing system from Rails to Go — process, phases, and what nearly broke us

I keep meeting engineers in the middle of a Rails-to-Go billing migration, and they all say the same thing: they thought it was a rewrite of a CRUD app, and it isn't.

A billing migration is its own kind of project. The code is the easy part. The hard part is preserving correctness across a moving target (invoices in flight, subscriptions mid-renewal, refunds half-processed, webhooks queued for retry) while two systems run side by side for weeks or months. This post is the field guide I wish someone had handed me.

It's deliberately generic. Any resemblance to a system I or you might have worked on is incidental, and the techniques apply regardless of stack flavor.

## Why migrate at all?

Before any process discussion, the project needs an answer to *why*. "Ruby is slow" is not an answer. Specific, measurable answers I've seen earn the migration:

- **Concurrency primitives.** A Rails monolith with a Sidekiq fleet can absolutely handle scale, but expressing "fan out, do twelve things in parallel, collect results, all under a deadline" is awkward. Go's goroutines + contexts make this a five-line function.
- **Decimal arithmetic and type safety on money.** Ruby's [`BigDecimal`](https://docs.ruby-lang.org/en/master/BigDecimal.html) works, but Ruby's *type system* lets a float sneak into a money calculation and you'll find it in a customer-facing invoice. A Go codebase with a `money.Amount` type that has no float constructor catches the bug at compile time.
- **Boot time and memory.** A Rails worker takes hundreds of MB and seconds to boot. A Go worker takes tens of MB and milliseconds. For a billing system that scales horizontally during end-of-month spikes, the difference compounds in your cloud bill.
- **Operational consistency with the rest of your stack.** If everything else is Go and the billing system is the lone Rails outlier, on-call engineers pay a tax every time they touch it.

If you can't write the *why* on one whiteboard line, you probably should not do this migration yet.

## The phase map

Every migration I've worked on or watched has roughly the same five phases. The phase you're tempted to skip is the one that bites you.

```
Phase 0: Understand
Phase 1: Carve a boundary
Phase 2: Shadow (build in parallel, run silent)
Phase 3: Cut over (one slice at a time)
Phase 4: Decommission
```

The naming is mine. The order is not negotiable.

### Phase 0: Understand the existing system

You do not rewrite a system you do not understand. You especially do not rewrite a *billing* system you do not understand, because billing is where every weird historical decision your company ever made comes home to live.

The deliverable from Phase 0 is a single document (call it the *domain audit*) that answers:

- What are the entities? (customer, subscription, plan, invoice, line item, payment, refund, credit note...)
- What are the state machines on each one? (`subscription: trialing → active → past_due → cancelled → ...`)
- What are the invariants? (an invoice's `total` must equal the sum of its line items, post-tax; a subscription cannot be `active` without a `current_period_end` in the future; a refund cannot exceed the original payment's amount)
- What are the integrations? (payment provider, tax provider, accounting export, dunning emails, webhooks in, webhooks out)
- What are the recurring jobs and what do they do? (renewal cron, dunning cron, currency conversion, reconciliation)
- Where does the system *talk to itself*? ([ActiveRecord callbacks](https://guides.rubyonrails.org/active_record_callbacks.html), model observers, after_commit hooks, anything that fires implicitly)

That last one is the killer in a Rails-to-Go migration. Rails callbacks are the great hidden coupling of every Rails app. Every `after_save :recalculate_balance` is a side effect the Go rewrite will need to replicate explicitly, and you won't notice the missing one until a customer's balance is wrong.

Spend a week on this. Maybe two. The cost of skipping it is shipping a bug that issues a refund twice.

### Phase 1: Carve a boundary

You do not rewrite the whole billing system in one shot. You rewrite *one capability* at a time.

The criteria for the first capability to carve out:

- **Read-heavy, not write-heavy.** Reads are easier to migrate than writes because the rollback is "just keep reading from Rails". Writes mean dual-write logic and conflict resolution.
- **Well-bounded.** "Fetch a customer's invoice history" is a great first slice. "Process a webhook from the payment provider" is a terrible first slice.
- **Low blast radius.** Pick something that, if it goes wrong, you can serve from Rails for a day while you fix the Go service. Don't start with the renewal cron.

A typical first carve: a read API that returns a customer's invoices, formatted for the customer-facing UI. The data still lives in Rails's database (we'll get to data decoupling). The Go service reads from the same tables, formats responses, and that's it. No writes.

Why start here? Because it forces you to build, on day one:

- The Go service skeleton (HTTP, logging, metrics, deploy pipeline).
- A way to share or mirror the database.
- A way to route traffic from the frontend to the new service.
- The observability you'll lean on for everything that comes after.

If those four things don't exist by the end of Phase 1, you don't have infrastructure to do the harder work.

### Phase 2: Shadow

Before you cut a single user over, the Go service runs *silently in parallel* with Rails for at least a few weeks.

The shape: every request that hits Rails is also dispatched to Go, with a different request ID. Both responses are recorded. A diff job runs over the recorded pairs and produces a report:

```
2026-06-12  GET /v1/invoices/123
  status: rails=200 go=200
  body  : equal (after canonicalization)

2026-06-12  GET /v1/invoices/124
  status: rails=200 go=200
  body  : DIFFERENT
  diff  :
    -  "total_cents": 12000
    +  "total_cents": 11999
```

Every divergence is either a bug in the Go service, or (terrifyingly often) a bug in the Rails service that nobody knew about because the output was never compared against an independent implementation. (You will find both. Budget for it.)

**Canonicalize before comparing.** JSON field order, floating-point formatting, decimal trailing zeros and timestamp timezones all need to be normalized before a diff. Otherwise every response looks different and you'll dismiss real bugs as noise.

You hold here, in shadow mode, until the diff is clean for *every* request shape for at least a week. Then you can think about cutting traffic over.

### Phase 3: Cut over, one slice at a time

The cutover is the moment you start serving real production responses from the Go service.

The technique I keep reaching for: a **feature flag, scoped by customer**. Not "10% of requests to Go", which is noisy and undebuggable. Instead: "this list of explicitly-enabled customers go to Go; everyone else goes to Rails." Start the list with one customer, yourself. Then a handful of internal accounts. Then opt-in beta customers. Then a percentage, by customer hash, growing weekly.

Why per-customer and not per-request? Because a single customer must see *consistent* behavior across requests. If their invoice list comes from Go but their invoice detail comes from Rails, and the two have drifted by a cent, the customer sees an arithmetic error.

The thing that makes per-customer cutover work is a routing layer somewhere in front of both services that knows the feature flag:

```go
if billing.IsOnGo(ctx, customerID) {
    return goBilling.Handle(req)
}
return railsBilling.Handle(req)
```

That layer is the cheapest insurance you'll buy. When something goes wrong on Go, you flip the flag for the affected customer and they're back on Rails before you've finished your second sip of coffee.

### Phase 4: Decommission

Eventually, every customer is on Go. The Rails code path is dead. Now you have to *prove it's dead* before you delete it.

Tactics:

- **Log every entry into the Rails code path with severity WARN.** If the logs are clean for two weeks, the path is genuinely dead.
- **Delete the code, not just stop calling it.** Stale dead code lies in wait. Delete it. The git history remembers if you ever need it.
- **Drop the dual-write logic.** As long as Rails was still running, the Go service was probably double-writing to keep Rails's state fresh. That logic comes out too.
- **Decommission the database tables only after a longer wait.** If the Rails app is gone but the tables are still being read by Go, leave the tables. If they're truly orphaned, drop them only after a backup and a delay.

Treat this phase as a project with its own scope. "We finished cutover, we're done" is how you end up paying for two systems for a year.

## Data decoupling: the hard part

The single hardest question in a Rails-to-Go billing migration: *who owns the data?*

The naive answer is "share the database". It works on day one and becomes a tar pit by month three. Two services writing to the same tables, with subtly different assumptions about callbacks, validations, and indexes, will diverge. The clean answer is "split the database eventually". Both answers are right; the migration is the messy middle.

The strategies I keep using, roughly in order of how I introduce them:

### Strategy 1: Shared database, read-only Go

In Phase 1, the Go service reads directly from the Rails database. It does *not* write. This is the simplest possible starting point and gives you the entire read API without any data-sync work.

Constraints:

- The Go service reads from a *replica*, not the primary, so it can't accidentally lock anything Rails needs.
- The Go service treats the schema as someone else's contract. Every Rails migration is now also a Go deployment risk. Coordinate.
- Any column-level meaning encoded in Ruby (e.g. an `enum` that maps integers to names, or a `serialize :data` that pickles YAML) needs to be replicated *exactly* in the Go reader. Misinterpreting a status enum is how you tell a customer their `cancelled` subscription is `active`.

This phase will last longer than you expect.

### Strategy 2: Dual write through an anti-corruption layer

When Go starts writing, you don't let it write directly to the shared Rails schema. You introduce an **[anti-corruption layer](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)** (ACL) on the Rails side: a small, explicit Ruby module whose only job is to accept calls from Go and translate them into the Rails idioms. (The pattern itself was coined by Eric Evans in *Domain-Driven Design* and is often paired with [Martin Fowler's Strangler Fig](https://martinfowler.com/bliki/StranglerFigApplication.html) approach.)

```ruby
module Acl
  module Billing
    def self.create_invoice(attrs)
      ActiveRecord::Base.transaction do
        invoice = Invoice.create!(
          customer: Customer.find(attrs[:customer_id]),
          amount_cents: attrs[:amount_cents],
          # ... mapping continues
        )
        AfterCommitJobs.enqueue_for(invoice) # explicit, not implicit
        invoice
      end
    end
  end
end
```

The ACL is the seam. Go calls a thin HTTP or gRPC endpoint backed by this Ruby code. The Rails internals (callbacks, observers, validations) are preserved on the Rails side, where they belong. Go doesn't have to know about them.

This is uglier than "direct DB writes" but dramatically safer. The implicit becomes explicit, and the failure modes are surfaceable.

### Strategy 3: Outbox + projection

Once the Go service has its own write path, both sides need to stay in sync. The pattern I [wrote about for the saga / outbox post](/saga-and-outbox) applies directly here.

Each side writes to its own database in a single local transaction, and writes an outbox row describing the change. A relay reads the outbox and publishes events. The other side subscribes and updates its projection of the data.

```
┌─────────── Rails ──────────┐    ┌──────────── Go ────────────┐
│  BEGIN                     │    │  BEGIN                     │
│    UPDATE subscriptions    │    │    UPDATE invoices         │
│    INSERT outbox(...)      │    │    INSERT outbox(...)      │
│  COMMIT                    │    │  COMMIT                    │
└─────────┬──────────────────┘    └─────────┬──────────────────┘
          │                                  │
          └──────────┬───────────────────────┘
                     ▼
                ┌──────────┐
                │  Broker  │
                └─────┬────┘
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
        ┌─────────┐         ┌─────────┐
        │ Rails   │         │   Go    │
        │ updates │         │ updates │
        │ its     │         │ its     │
        │ view    │         │ view    │
        └─────────┘         └─────────┘
```

The benefit is that neither side reads the other's transactional state directly. Each side has its own consistent local view, refreshed from events. The cost is **eventual consistency**: there's a window, usually a few seconds, where the two sides disagree.

For billing, the question becomes: *which fields are okay to be eventually consistent, and which are not?*

- **A customer's display name?** Eventually consistent is fine.
- **A subscription's `current_period_end`?** Eventually consistent is fine, mostly.
- **The amount about to be charged on the customer's card?** Not eventually consistent. That one needs to be authoritative on one side, with the other side reading a fresh value at decision time.

The hard work of Phase 2 is mapping each field to that question and acting accordingly.

### Strategy 4: Single-writer migration per entity

Eventually each entity belongs to one service. Subscriptions live in Go; legacy refund records live in Rails; the customer record is owned by Go but mirrored to Rails as a read-only projection.

The transition for each entity is its own mini-migration:

1. **Both services write.** Go writes via its own path, Rails writes via the ACL.
2. **Backfill historical data into the new owner's schema.** Cron-based, batched, restartable. Never lock the source table for the whole backfill.
3. **Compare reads from both sides** until they match for some agreed window.
4. **Flip the writer.** Now only Go writes. Rails reads from a projection updated by Go's outbox.
5. **Eventually, retire the Rails-side projection.** Anyone who needs the data calls Go.

Each entity moves through these stages independently. You will be in the middle of stage 4 for one entity and stage 1 for another, at the same time. That's normal.

## Money is not a number

This is the single most expensive mistake I've seen.

In Ruby, money commonly lives as a `BigDecimal`. In Go, the temptation is to use `float64`. Don't.

The patterns that have worked:

- **Store money as integers** (`amount_cents int64`, `currency string`). Never store money as floats. Never store money without a currency. The integer is the source of truth.
- **Wrap it in a type that hides the integer.** `money.Amount` with no float constructor. Arithmetic operations only between same-currency amounts. Multiplication only by a unitless scalar. Division returns a quotient *and* a remainder, so you can't lose a cent.
- **Round explicitly, with a stated rule.** Banker's rounding vs. round-half-up vs. truncate: these will give different answers, and they will give different answers from the Rails side if you weren't paying attention to which one ActiveRecord was doing.
- **Forbid implicit conversions to display.** Formatting money for a UI is an explicit call, `m.Format(locale)`, not a `String()` method that hides the locale assumption.

The first time you bring up the Go service and the integration tests pass, run a reconciliation report against Rails over a month of real data. If even one cent disagrees, find it before you go further.

## Deployment hurdles

The architectural patterns get the most ink, but the deployment hurdles are what make these projects miss their dates.

### Schema migrations during cutover

While both Rails and Go run, schema changes need to be *additive* and *backwards-compatible*. No dropping columns. No renaming columns. No tightening NOT NULL constraints without a `default`. Every migration goes through the dance:

1. Add the new column (nullable, or with a default).
2. Backfill values.
3. Deploy both services to read/write the new column.
4. Only after both are deployed and stable: tighten constraints or drop the old column.

This is true for any rolling deployment, but it's *especially* true when "both services" run in different languages and one of them might be three weeks behind the other on the new shape.

### Background jobs are their own migration

[Sidekiq](https://github.com/sidekiq/sidekiq) jobs don't magically become Go workers. A typical Rails billing system has dozens of jobs: renewal, dunning, reconciliation, currency conversion, webhook retries, report generation. Each one needs a Go equivalent, and the transition has the same shadow / cutover / decommission shape as the HTTP API.

A trap: jobs that *enqueue other jobs*. If your Go renewal worker enqueues a Sidekiq dunning job by name, you've coupled the two systems' queues. Untangling that mid-migration is painful. Decide early: do you bridge the queues (Go enqueues via Sidekiq's Redis protocol), or do you migrate both producer and consumer together?

### Cron jobs and locks

Anything that runs on a schedule needs a careful handover. You do not want both the Rails renewal cron and the Go renewal cron running on the same day. Pick one. Cut over with a flag. Confirm the old one is disabled before you let the new one run.

Distributed locks help (a row in a `cron_locks` table with `(job_name, run_date)` as the primary key, both services acquire-or-skip), but the real solution is "exactly one writer per scheduled job during transition".

### Connection pool sizing

Ruby and Go have radically different concurrency models. A Rails app with a pool of 25 DB connections per Puma worker is reasonable. A Go app with the same per-process pool size will exhaust the database under the same load, because Go can have *thousands* of goroutines waiting on those 25 connections.

Calculate the database's max connections, divide by the number of Go pods, divide again by a safety factor, and that's your [`db.SetMaxOpenConns`](https://pkg.go.dev/database/sql#DB.SetMaxOpenConns). Don't copy the Rails number.

### Webhooks in and out

Webhooks deserve special attention because they are *the* place where a billing migration leaks correctness if you're not careful.

- **Inbound webhooks** (from your payment provider): exactly one consumer. During the migration, point them at a small router that decides whether to forward to Rails or Go *per event type* (or per customer, depending on your cutover dimension). Never let both services try to process the same webhook.
- **Outbound webhooks** (to your customers): exactly one publisher. While both services exist, *only one* should send the `invoice.paid` webhook for a given invoice. The other side, if it's tracking the same event, suppresses the webhook send. Use the outbox to centralize this: only the system that wrote the row publishes its event.

A duplicated outbound webhook means your customer's system receives a "paid" notification twice. They might charge their downstream customer twice. You will get the email.

### Rollback strategy

The thing that lets you sleep at night during the cutover is a rollback that takes seconds, not days. Concretely:

- **The feature flag flips per customer.** Affected customer goes back to Rails immediately.
- **Both databases stay in sync via the outbox.** A customer flipped back to Rails finds their data is current.
- **Schema changes are reversible.** If you added a column that Go writes, Rails ignores the unfamiliar column on read. If you tighten a constraint, deploy it only after both sides are guaranteed to satisfy it.

You probably won't need to roll back often, but you need to be able to. Without that, every cutover is a heart-attack moment.

### Observability parity

Whatever you have on Rails (request logs, error tracking, metrics, tracing) you must have on Go at parity *before* the first real customer moves. The migration's first weeks will produce surprises, and the only way to debug a surprise on Go without panicking is to have the same instrumentation you'd have on Rails.

Specific things to wire up day one:

- **Trace IDs that cross the language boundary.** A request that enters Rails and is forwarded to Go (or vice versa) carries the same trace ID. Otherwise you can't follow a single user action across services.
- **Error reporting with the same severity taxonomy.** A 500 from Go should look the same to your error tracker as a 500 from Rails.
- **Latency percentiles on each endpoint.** You will be asked, more than once, "is the Go version faster?". Have the answer.

## Thinking patterns

A few mental moves I keep coming back to.

**Migrations are not rewrites.** A rewrite is "I'm going to build a better version of this." A migration is "I'm going to move this thing from system A to system B, preserving its current behavior, including the bugs I don't know about." The temptation to "fix the data model along the way" is the temptation that kills the project. Move first. Improve later. They are different projects.

**The migration is the project.** Two teams I've seen treated the migration as a "while we're here" alongside the year's product roadmap. Both teams failed. A migration of this size is a 6–18 month project that needs ownership, a roadmap, and an end date.

**Test invariants, not implementations.** Your assertions should be statements like "the sum of an invoice's line items equals the invoice's total, post-tax, in every currency, on every day of every customer's history." That assertion holds for both Rails and Go and it's *checkable*. "The `calculate_total` method returns 12.00 for input X" is an implementation test that needs rewriting on the other side and tells you nothing about correctness.

**One slice. One customer. One day.** When you cut over, cut one slice of one capability for one customer for one day. Watch what happens. Then expand. The instinct to "just enable everyone next Monday" is the same instinct that ships the bug that issues twelve refunds in twelve seconds.

**The boring decisions are the load-bearing ones.** Money as integers. One writer per entity. Outbox for cross-service events. Feature flags scoped by customer. None of these will impress in a tech talk. All of them will determine whether this migration succeeds.

## Lessons that are worth repeating

A handful that are easy to write down and surprisingly hard to internalize:

1. **The migration is its own project.** Resource it that way.
2. **Spend a real amount of time on Phase 0.** A week is not enough. Two might be.
3. **Shadow before you cut.** A diff log is the cheapest insurance you'll ever pay for.
4. **Per-customer feature flags beat per-request percentages.** Always.
5. **Money is integers + currency. Never floats. Never floats. Never floats.**
6. **One writer per entity, eventually.** Get there as fast as the project will allow.
7. **Apply the outbox pattern early.** The cost is small. The correctness gain is enormous.
8. **Treat webhooks like a critical section.** Exactly one publisher per event. Exactly one consumer per event.
9. **Build observability parity before the first cutover.** Not after.
10. **Have a rollback that takes seconds.** Or you will be the person paged when it doesn't.

I have left out a lot: currency conversion, tax engines, accounting exports, regulatory holds, the joy of trying to explain dunning to a non-billing engineer. Maybe future posts. For now: if you're about to start one of these, take the phase map, take Phase 0 seriously, and remember that the migration is the project.

Good luck. And mind your cents.

## References

**Patterns**

- [Strangler Fig Application — Martin Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Anti-Corruption Layer pattern — Microsoft / Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)
- [Saga pattern (microservices.io)](https://microservices.io/patterns/data/saga.html)
- [Transactional outbox pattern (microservices.io)](https://microservices.io/patterns/data/transactional-outbox.html)

**Rails / Ruby**

- [Active Record Callbacks — Rails Guides](https://guides.rubyonrails.org/active_record_callbacks.html)
- [Sidekiq](https://github.com/sidekiq/sidekiq)
- [`BigDecimal` — Ruby standard library](https://docs.ruby-lang.org/en/master/BigDecimal.html)

**Go**

- [`database/sql`](https://pkg.go.dev/database/sql), including [`SetMaxOpenConns`](https://pkg.go.dev/database/sql#DB.SetMaxOpenConns)

**Related posts**

- [The Go transaction manager bug that ate my rollback](/go-transaction-manager-commit-errors)
- [MySQL transactions — same Go pattern, different ways to lose](/mysql-transactions-silent-rollback)
- [Idempotency keys](/idempotency-keys-deduped-writes)
- [Saga and the outbox](/saga-and-outbox)
