---
title: Speed versus memory — six weeks of benchmarks, and the five conclusions I had to take back
date: 2026-09-01
permalink: /speed-versus-memory-report-tuning
---

# Speed versus memory — six weeks of benchmarks, and the five conclusions I had to take back

We have an analytics report at work that aggregates a month of clinical records: for each treatment performed, how many, for which age bracket, under which insurance, worth how many points. One office, one month, optionally grouped by tag or by staff member.

The first working version took **10.39 seconds and issued 25,001 queries** for a hundred thousand items. The instance it runs on has 512 MB of RAM and a quarter of a vCPU.

Five phases later, at *ten million* rows, it does the same job with **4 queries.**

Every claim below is tied to the benchmark that produced it and the conditions it ran under. I'm being pedantic about that on purpose, because the most useful thing I learned here is that **five conclusions I had already written down and acted on were wrong**, and not one of them was wrong because of the code.

Every implementation rebuilt from its own commit and re-run at 10M rows on 1 vCPU:

| implementation | 10M rows | queries @10M | peak heap @10M |
|---|---|---|---|
| C4 (first version) | 169.692 s | 5,001 | 1,571 MB |
| Phase 1 | 71.253 s | 5,001 | **55.7 MB** |
| Phase 3 | 62.105 s | 5,001 | 56.3 MB |
| Phase 4 | 46.207 s | 752 | 392.6 MB |
| **Phase 5** | **20.530 s** | **4** | 506.5 MB |

*(Health warning before you lean on those digits: the cells are single runs, and the fixture changes between the first three rows and the last two. Cross-session drift of up to ±30% turned up on this harness. Trust the ordering and the order of magnitude, not the decimals.)*

Wall time falls monotonically, 8.3× end to end. Peak heap doesn't. It drops 28× at Phase 1, then climbs back up 9×.

Phase 1 is the memory minimum, and it is pure streaming. Every phase after it spent memory to eliminate round trips, and did it deliberately, with the bill written down each time.

So the tradeoff behaved differently depending on how far back you stand. Inside a single read shape it kept dissolving: streaming the office-wide join is faster than loading it *and* 68× smaller. Across the project it was real, and by the last phase memory pressure had started converting back into wall time.

A caveat I'll repeat where it bites. Every number here is a benchmark on one developer machine with MySQL co-resident, against synthetic data, with two cross-database dependencies substituted. They come from **three different environments**: the early bake-off figures predate the harness entirely, the 2026-07-29 comparison tables were taken in pinned containers, and the Phase-5 tables were taken on an M2 Pro outside one. Numbers are comparable *within* an environment and not across them, so I've labelled which is which throughout. None of them are production latencies.

## The measurement harness, before anything else

The benchmarks drive the actual use case against a live MySQL, through the actual repositories and the actual GORM queries, with nothing mocked. They sit behind an env gate so a normal `go test ./...` skips them:

```go
if os.Getenv("RUN_DB_BENCH") != "1" {
    b.Skip("set RUN_DB_BENCH=1 (and start the local MySQL) to run")
}
```

Two dependencies are substituted, which matters when you read the heap tables below. Patient birthdays live in a second database that the timings deliberately exclude, and the master-data client is an in-memory stand-in.

Every DB-backed arm in the report benchmark runs through one shared harness. Each line of it exists because something went wrong without it:

```go
func runBenchReportArm(b *testing.B, name string, op func() error) {
    b.Run(name, func(b *testing.B) {
        // Warm this arm's pages and caches before timing anything.
        if err := op(); err != nil {
            b.Fatalf("%s warm-up: %v", name, err)
        }

        runtime.GC()
        b.ReportAllocs()

        // Reproduce the 512MB instance's ceiling for the timed section only.
        debug.SetMemoryLimit(460 << 20)
        defer debug.SetMemoryLimit(math.MaxInt64)

        sampler := startHeapSampler()
        defer sampler.stopAndPeak()

        dbBenchQueryCount.Store(0)
        b.ResetTimer()

        for range b.N {
            if err := op(); err != nil {
                b.Fatalf("%s: %v", name, err)
            }
        }

        b.StopTimer()

        b.ReportMetric(float64(sampler.stopAndPeak())/1e6, "peakHeapMB")
        b.ReportMetric(float64(dbBenchQueryCount.Load())/float64(b.N), "queries/op")
    })
}
```

**The discarded warm-up run.** An earlier phase of this report was misled by a cold InnoDB buffer pool. Arms run in a *fixed* order with the baseline first, so without a warm-up the baseline absorbs the cold-page cost of every table the tier touches while the later arms run warm. That inflates the ratio in the direction that flatters the change. At `-benchtime 3x`, one cold iteration is a third of the reported mean.

**`SetMemoryLimit(460 << 20)` around the timed section.** This models the 512 MB instance, so a materialization that would OOM in production shows up as GC thrash rather than passing quietly on a 32 GB laptop.

**A query counter**, wired as a GORM callback. Query count turned out to be the most predictive number in the project. More than allocations, more than heap.

**Two memory metrics that do not mean the same thing.** The peak sampler polls on a ticker:

```go
// benchHeapSampler polls runtime.ReadMemStats().HeapAlloc on a ~5ms ticker
// and tracks the maximum live-heap size observed.
```

So `peakHeapMB` is a *sample* of `HeapAlloc`: a lower bound on the true peak (a spike between two ticks is invisible), and it counts uncollected garbage as though it were live. Whereas:

```go
// benchRetainedHeapBytes returns the live-heap growth caused by holding build()'s result.
// allocs/op and B/op count everything a build TOUCHES; this counts only what SURVIVES.
// Two GCs on each side because the first can leave the second's finalisable garbage behind.
func benchRetainedHeapBytes(build func() any) uint64 {
    runtime.GC()
    runtime.GC()
    var before, after runtime.MemStats
    runtime.ReadMemStats(&before)

    held := build()

    runtime.GC()
    runtime.GC()
    runtime.ReadMemStats(&after)
    runtime.KeepAlive(held)

    if after.HeapAlloc < before.HeapAlloc {
        return 0
    }

    return after.HeapAlloc - before.HeapAlloc
}
```

Touched versus survives. I confused the two and put a 4× error into a design doc; that comes up later.

## Where we started: 25,001 queries

The original design was a per-patient loop. For each patient, fetch records, then blocks, then items, then resolve which insurance was billing. Collect everything, then aggregate.

The design doc raised performance *before a line was written*: the per-patient loop plus the billable-group fetcher's own documented N+1 "can reach tens of thousands of sequential queries for a large office." It set a target of **p95 ≤ 3 s**.

It set no memory target at all: no heap budget, no RAM cap. Half of what follows comes out of that omission.

`BenchmarkCollectTreatmentReport`, 100,000 items across 5,000 patients, collect *and* aggregate:

| metric | v1 per-patient |
|---|---|
| wall | **10,394 ms** |
| queries | **25,001** (= 5 × patients + 1) |

Then the seeder was extended to actually populate insurance rows. It hadn't been, so every billable-group lookup returned empty and skipped the N+1 underneath it. Query count went from 25,001 to **28,751**. The benchmark had been measuring a cheaper report than the real one for its first three tiers.

Measured properly, on the seeded data, the collect step alone churns through **605.9 MB across 11.36 M allocations** per report at 100k items. That's allocation *volume*, not resident memory. Peak heap for the same run is 38.1 MB.

How much of the 10 seconds is the aggregation itself? `BenchmarkAggregateTreatmentReport` runs the pure in-memory fold, no database:

| records | wall | bytes | allocations |
|---|---|---|---|
| 100,000 | 16.3 ms | 3.12 MB | 104,280 |
| 1,000,000 | 161.8 ms | 24.7 MB | 1,004,280 |

**16 ms of aggregation inside a 10,394 ms report**, 0.16% of the wall, and perfectly linear. Knowing that on day one is why nobody spent a week optimising the fold.

## The bake-off: five collectors, and a lesson about noise

Five collection strategies, each gated behind a differential test asserting data-equal output against v1 before it was allowed to compete. Group-row order gets normalized by key first, because first-seen ordering legitimately differs between strategies.

At 100,000 items, `groupBy=none`, collect-only, `GOMAXPROCS=1`:

| strategy | wall | peak heap | queries | allocations |
|---|---|---|---|---|
| v1 per-patient | 8,579 ms | 38.1 MB | 28,751 | 11.36 M |
| C1 load-all | 4,492 ms | **179.3 MB** | 13,754 | 8.93 M |
| C3 chunk-50 | 4,474 ms | 51.0 MB | 14,051 | 8.99 M |
| C3 chunk-200 | **4,359 ms** | **46.9 MB** | 13,826 | 8.95 M |
| C3 chunk-500 | 4,349 ms | 52.1 MB | 13,781 | 8.94 M |
| C3 chunk-1000 | 5,345 ms | 59.6 MB | 13,766 | 8.94 M |

**Load-all costs 3.8× the memory and is *slower*.** C1 sits at 179.3 MB against chunk-200's 46.9 MB, and loses on wall time too. At this size that wasn't a budget failure. 179 MB cleared the ~400 MB target comfortably. We rejected it on shape. Chunked heap is independent of office size; load-all's grows with it. What killed it was the slope.

**More queries can be faster.** chunk-200 issues 72 *more* queries than load-all and beats it by 133 ms. Smaller result sets mean less allocation churn and less GC pressure under a 460 MiB limit, which outweighed the extra round-trips.

**Chunk size is not monotonic.** chunk-1000 is slower than both chunk-500 and chunk-200 while also using more memory. "Fewer round-trips is always faster" breaks down somewhere between 500 and 1000 patients per `IN` list.

The big lever was the insurance loop. The residual per-patient calls were ~97% of what remained (13,750 of ~14,000 queries). Batching them, at the same chunk size, took 100k from 5,072 ms / 13,826 queries to **905 ms / 126 queries**; at chunk 500 it reached **863 ms / 51 queries**.

### The number that should have stopped me

Those two sweeps also contain the number that invalidates most of the following month.

**The same v1 arm, same machine, about 41 minutes apart: 8,579 ms and 9,990 ms.** A 16% spread.

Meanwhile the decisions I was making off that harness were between candidates separated by **0.2%** (chunk-500 vs chunk-200) and **3%** (chunk-500 vs load-all).

I was resolving 0.2% differences with an instrument that moved 16% between sweeps. Everything in the next three phases inherited that.

And the 16% isn't even clean drift. The billable-insurance service that every arm calls was refactored between the two sweeps, so the gap is drift plus an uncontrolled change, and I can't tell you how much of each. I couldn't take my own instrument's error apart, and I used it to settle 0.2% differences anyway.

## Phase 1: fold instead of collect

C4 still accumulated every record before aggregating, so peak heap was `O(rows)`. The scaling run measured **402 MB at 1M rows**, 78% of the instance, and projected a hard OOM just above 1.15M.

The fix is not clever. A sum doesn't need every row at once, and the output is a few hundred rows whether the input is 100k items or 10M. So restructure from *accumulate-all → resolve → aggregate* into a per-chunk fold:

```go
acc := newTreatmentReportAccumulator(groupBy, period)
for chunk := range chunks {
    records := collect(chunk)
    acc.fold(records, resolve(chunk))   // fold, then drop
}
return acc.assemble()
```

Phase 1 also materialized a `count` column. The per-item billing count had been living inside a JSON `attributes` blob, so the collector parsed JSON **one million times per report** on the hot path, and fetched the largest column in the table to do it. A slim projection returning `(BlockID, ReferenceID, Count)` meant the blob was never fetched, transferred, or parsed.

At 1M rows in the controlled environment described later:

| 1M rows | C4 | Phase 1 |
|---|---|---|
| wall | 10,422 ms | **6,638 ms** |
| peak heap | 326.3 MB | **10.7 MB** |
| allocations | 62.8 M | 29.9 M |

A 1.57× speedup and a **30× memory reduction** out of one change. At 10M rows, Phase 1's peak heap is 55.7 MB against C4's 1,571 MB: **28× smaller.**

The accumulator is now sized by `O(distinct output groups)` instead of `O(input rows)`. Phase 1 stayed the memory floor for the whole project; nothing later beat it.

**It also introduced a correctness bug that is still open.** `ADD COLUMN count NOT NULL DEFAULT 0` sets every pre-existing row to 0, and the read path filters `count > 0`. Every item written before the migration is silently excluded, so historical months under-count. The backfill was consciously deferred (the product is unreleased) and it's written in bold in the design doc. The JSON parse we deleted was also the thing that made old rows readable.

## Phase 2: an allocation win that was not a latency win

Many items in a block share a treatment code, so group in SQL, `SUM(count) ... GROUP BY block_id, reference_id`, and ship one row per `(block, code)` instead of one per item.

The reasoning was explicitly about **asymmetric hardware**: a 0.25-CPU app next to a capable managed MySQL means DB CPU is cheap and app CPU is expensive, so move work *toward* the database. The doc hedged it ("must be watched in the bench"), which was the right instinct.

The A/B, same data with and without the roll-up, mean of three:

| tier | ungrouped | grouped | Δ wall | Δ allocations | Δ bytes |
|---|---|---|---|---|---|
| 100k | 0.563 s | 0.508 s | −9.7% | −24.3% | −38.6% |
| 500k | 2.740 s | 2.546 s | −7.1% | −24.3% | −38.9% |
| 1M | 5.582 s | 5.228 s | −6.3% | −24.3% | −39.0% |
| 2M | 10.633 s | 10.408 s | **−2.1%** | −24.3% | −39.0% |

The allocation saving is **exactly −24.3% at every tier, to the decimal**, a structural property of the row shape, completely scale-invariant. The wall saving decays monotonically and by 2M sits at −2.1%, inside the noise band.

So Phase 2 buys allocations and GC pressure. At the size it was designed for, **it does not buy latency.**

I had recorded it as "−7% wall", a figure taken at 1M, in a run that already showed 2M coming out roughly flat. The flat tier was right there in the same table, and I quoted the one that made the change look good. Allocation counts are deterministic and beautifully reproducible, which makes them easy to measure and dangerously easy to mistake for a latency result. A −24% allocation cut is worth having on a CPU-starved box, because GC work is CPU you don't have. It is still not a −24% report.

(The roll-up got re-proposed twice more. In the Phase-5 architecture it delivered its Go-side promise exactly (**−67% allocations, −66% bytes**) while **wall rose 248%** and peak heap didn't move. The doc's note about it is my favourite line in the repo: *"This section exists so it is not re-proposed a fourth time."*)

## Phase 3: the profiler was lying

Phase 3 started with a profile, which overturned two things I had been assuming for weeks.

**"The report is 91% DB-bound."** That came from a timing callback wrapping the whole query call, which includes GORM's row scan and struct materialization. `EXPLAIN ANALYZE` put actual database execution at **~1.5 s of a ~5.5 s pass at 1M**. The report was app-CPU-bound on turning rows into objects, and I had spent weeks believing the opposite.

**"The fix has to be a smarter query."** It wasn't. Each block entity unmarshalled two `serializer:json` columns, `diseases` and `dental_formula_detail`, and built value objects from them. The report reads exactly two fields off a block: its ID and its record ID. Every one of those JSON parses was discarded.

Replacing the entity fetch with a two-column projection:

| 1M rows | Phase 1 | Phase 3 |
|---|---|---|
| wall | 6,638 ms | **6,150 ms** |
| allocations | 29.9 M | **24.3 M** |

−7.4% wall, −19% allocations, purely from not doing work.

Two negative results from the same phase:

- **Pushing the whole aggregation into SQL** measured **~3% slower**; `EXPLAIN ANALYZE` showed the cross-join `GROUP BY` forcing a ~3 s "Aggregate using temporary table."
- **Applying the same projection to the records** was worth ~2%, because records carry no JSON columns, confirming the cost was the unmarshal, not row count.

And the verdict I wrote at the time was **"about 2% of the full pass — basically noise,"** from comparing two single runs at different moments, 5.974 s against 6.051 s. A real 7% gain discarded as jitter, by an instrument with 16% drift.

## Phase 4: where the tradeoff dissolves, locally

After Phase 3 the pipeline still issued **501 queries** at 1M rows, and wall time was dominated by round-trips. The fix is to let the database do the join. The interesting question is how to *consume* it.

Five read shapes, prototyped against the same seeded million-row office on Phase-3 code:

| shape | wall | peak heap | allocations | verdict |
|---|---|---|---|---|
| staged 3 reads, chunk 500 | 5.208 s | 5.1 MB | 19.9 M | baseline |
| staged 3 reads, chunk 4000 | 5.561 s | 15.6 MB | 19.8 M | worse on both |
| join, chunked per patient | 2.992 s | 6.6 MB | 20.3 M | |
| join, office-wide, load-all | 1.290 s | **278.6 MB** | 20.0 M | memory trap |
| join, office-wide, cursor-streamed | **1.169 s** | **4.1 MB** | **7.0 M** | adopted |

The load-everything join is what you reach for when you've decided to buy speed with memory. It costs 278.6 MB, more than half the instance, and it is **slower** than streaming the same join off a cursor. The stream wins on wall, on memory by **68×**, and on allocations by nearly 3×.

There was no tradeoff here at all, just a materialization step costing 274 MB that bought nothing, because nothing downstream ever needed two rows at once. Allocations collapse from 20.0 M to 7.0 M for the same reason the heap does: the slice that would have held a million joined rows never exists, so neither does the churn of growing it nor the GC work of tracing it.

That got promoted to a hard constraint:

> Never materialize the joined result set. The stream MUST use `Rows()` + `rows.Next()`, folding a small buffer at a time. `.Scan(&slice)` over the join measured 277 MB — forbidden.

...with a tripwire in the benchmark: *"If peak heap exceeds ~50 MB at any tier, STOP and report — that means a forbidden materialization crept in."*

### The phase as a whole still cost memory

This is where I had to correct my own spec.

A single office-wide cursor **cannot pause mid-stream** to look things up. So to make one streamed pass possible, two things have to be resolved *up front* and held for the duration: a record→insurance bucket map, and patient birthdays (which live in a different database entirely and can never be a join column).

Those pre-resolves are what cost the memory. At 1M rows, peak heap went 11.0 MB → 23.8 MB. At 10M rows, **56.3 MB → 392.6 MB.** Phase 1's flat heap is gone, because the bucket map is `O(records)` and the birthdays are `O(patients)`.

The spec had claimed "~10–20 MB" and had to be corrected in place:

> the retained figure (7.6 MB) is in that range, but it is **no longer flat** across tiers as Phase 1's was

So the trade is: **one office-wide streaming pass buys 41% of the wall and eliminates 424 of 501 queries, paid for with pre-resolve maps that grow with office size.** Good deal. But a deal, and I originally wrote it up as a free win.

(Two smaller costs. Phase 4 allocates *more* than Phase 3 (27.0 M vs 24.3 M) because a joined row carries seven fields per item where the staged read carried three. And streaming broke an unspecified API contract: group rows had been emitted in first-seen order, which depended on stream order. Adding `ORDER BY` would reintroduce a filesort and kill the streaming, so the fix was sorting group keys at assemble time, a **visible behavioural change**. The ordering test deliberately uses staff IDs 9 and 10, because a naive string sort puts "10" before "9".)

### What the peak heap number contains

Running `benchRetainedHeapBytes` against each stage, forcing collection and measuring what survives, at 1M rows, 150k records, 50k patients:

| stage | retained | why it exists |
|---|---|---|
| baseline | 0.76 MB | — |
| patient IDs | +0.43 MB | the window's distinct patients |
| record→insurance bucket | +4.82 MB | derived over other tables; a cursor can't pause to compute it |
| birthdays | +1.97 MB | ages live in a different database |
| stream + assemble | +0.36 MB | a fixed 2,000-row fold buffer, plus output cells |
| **total retained** | **7.58 MB** | against a **23.78 MB** sampled peak |

**Two-thirds of the scary number was collectable churn.** With `GOGC=100` a collection doesn't trigger until the heap reaches roughly twice the live set, so an ~8 MB live heap naturally *samples* at 16–23 MB.

The architecture diagram had estimated the bucket map at ~2.4 MB; measured, it's 4.8 MB, the largest retained term. And an earlier spec revision had extrapolated the *sampled* figure to claim ~300 MB live at a 10M-item office. Extrapolating the *retained* figure gives **~76 MB**. Wrong by 4×, from scaling the wrong metric.

I later tried to lean on that ~2× rule and it broke. Across five tiers, the ratio of "peak dropped" to "live set removed" ranged **0.98× to 3.53×**; across all fifteen arms, **0.44× to 3.98×**. GC pacing explains why removing X removes more than X. It does not predict any individual number.

## The re-measurement, and five conclusions I had to take back

By this point I had a month of conclusions gathered piecemeal: single runs, over several days, on a machine also running the database under test.

So I rebuilt every phase from its own commit via git worktrees, cross-compiled each to a static Linux binary, and ran them in identical pinned containers (`--cpus=1 --memory=8g`, `GOMAXPROCS=1`) against one freshly created MySQL 8.0 on a private Docker network. Three passes each. Application logging suppressed, because the insurance service emits one INFO line per billable group, **15 MB of log in 60 s**, and that stderr I/O had been inside the original timings.

Then, before measuring anything, I ran one identical configuration twice to find out what the instrument could resolve: **1–3%.** That became the yardstick. Anything smaller is not a result.

Five things I had written down turned out to be false.

**1. "The chunked join is a 77.6 s index disaster — 14× slower — the shape is fatal."**
Re-measured: **2.99 s.** The collapse was a missing composite index, not the read shape. With the index present the chunked join actually *beats* the three staged reads, 2.99 s against 5.21 s. I had condemned an architecture in writing, for a schema problem, and been wrong by 26×.

**2. "Phase 3 was about 2% — basically noise."**
Re-measured: **−7.4% wall and −19% allocations at 1M**, and a consistent 7–12% wall gain at every tier.

**3. "Phase 2 bought −7% wall."**
Re-measured: **−2.1% at 2M**, inside the noise band. The −7% was real at 1M and I quoted it as the result, when the tier trend across the whole table was the result.

**4. "More CPU is a dead end — 4× the CPU bought only 1.35×."**
Re-measured on Phase-4 code: **2.18×.**

| CPU quota | wall @ 1M | vs 1.0 vCPU |
|---|---|---|
| 1.0 vCPU | 3.606 s | — |
| 0.5 vCPU | 4.064 s | +13% |
| 0.25 vCPU (production) | 7.857 s | +118% |

This one reversed rather than sharpened, and for a structural reason. The original was measured by `getrusage` at 34% CPU / 66% DB-wait. Removing 424 of 501 queries took most of the DB-wait out, leaving roughly 40% compute / 60% wait at 1 vCPU and CPU-dominated at 0.25. **Fixing the I/O problem is what made buying CPU worthwhile.** "Scaling up won't help" was a true statement about a system that no longer existed.

**5. "C4 out-of-memories past about 1.15M rows."**
It doesn't. C4's benchmark only ever had three tiers, and the OOM was fitted through them: 402 MB at 1M, extrapolated into the 460 MiB limit. Adding a 2M tier to a variant of the benchmark and actually running it (a single pass, not the mean of three): it completes in 19.76 s at **394.1 MB peak**, under the limit. The GC contains it by working harder, so C4 *degrades* rather than dying.

They aren't one mistake repeated. Each went wrong in its own way:

- **Below the noise floor** (#2 and #3). I measured a difference smaller than my drift, without knowing what my drift was.
- **A confounded variable** (#1). Phases 1–3 had historically run *without* the composite index, Phase 4 *with* it. Until every arm ran on the same schema I was moving code and schema together and attributing all of it to code. This is the worst of the four, because more repetitions would not have caught it.
- **A statement that expired** (#4). "More CPU won't help" was *true* when measured, at 34% CPU / 66% DB-wait. Deleting 424 queries changed the ratio and quietly falsified it. Nobody re-ran it because nobody thought of a conclusion as something with a shelf life.
- **An extrapolation never checked** (#5). A line through three points, promoted to a fact about a fourth.

## Phase 5: buying speed with memory, on purpose

After Phase 4, two-thirds of the remaining wall was one phase: the insurance pre-resolve. It issued 75 of the 77 queries and was the last path constructing full domain entities, reading whole records and insurance objects to collapse them into a four-value enum.

The reformulation that killed it: *a record's insurance bucket is the bucket of the first coverage window, in sort order, whose clipped date range contains the record's date.* Coverage windows come from two slim office-wide reads. **No record read, no entity construction**, which is the entire basis for deleting 75 queries.

The insurance phase in isolation, at 1M rows:

| | Phase 4 | Phase 5 |
|---|---|---|
| wall | 2.274 s | **0.029 s** |
| queries | 75 | **1** |
| allocations | 9.51 M | 0.38 M |
| bytes | 334.7 MB | 14.1 MB |

End to end, both arms in the same run, same office, same harness:

| tier | Phase-4 shape | Phase 5 `groupBy=none` | ratio |
|---|---|---|---|
| 100k | 352.8 ms | 107.9 ms | 3.27× |
| 500k | 2,239.3 ms | 642.6 ms | 3.48× |
| **1M** | **3,825.2 ms** | **1,495.6 ms** | **2.56×** |
| 2M | 8,618.5 ms | 3,368.3 ms | 2.56× |
| 5M | 23,644.8 ms | 12,523.6 ms | 1.89× |

Queries at 1M: **77 → 4** (6 for `tag`, 5 for `staff`). `tag` costs +31% over `none`; `staff` +2%.

*(This ran on a different machine from the earlier tables, on a seed with more insurance data. The absolutes are not interchangeable; the arm-versus-arm **ratios** are what carry, because both columns came from the same run. The design doc freezes this table at a specific commit and forbids recomputing it. The Phase-4 arm was deleted afterwards, and dividing a fresh Phase-5 number by a frozen Phase-4 one would silently credit Phase 5 with a benchmark fix.)*

### And the memory

The acceptance criterion I had written was *"peak heap no worse than Phase 4's 23.8 MB."*

Measured at 1M, `groupBy=none`: **109.9 MB.** After a harness fix removing a benchmark fixture that was inflating the live set, plus five other commits: **~60 MB** (two draws: 60.17 and 64.73).

Missed by 2.5×.

Phase 4 had held a bucket per *record*: `O(records)`, 4.82 MB retained. Phase 5 holds coverage windows per *patient-insurance*: 83,334 windows at 1M rows, 128 bytes each, 10.7 MB retained. Comparable at 1M, and it stays comparable up the tier ladder. At 5M it's 416,667 windows and 55.3 MB, which is the same 5× that the record count grew.

The axis that scales worse isn't rows at all, and it took a separate sweep to find. The office-wide coverage read returns every coverage the office owns, **whether that patient visited in the reported month or not**, so Phase 5's heap is linear in the size of the *roster* where Phase 4's map was linear in the records actually touched. Every tier in the ladder above was seeded with no idle patients, so the ladder looks harmless.

The doc's own summary:

> Phase 5 costs materially more heap than Phase 4 and its target was set before the coverage map existed.

That last clause is the bit I'd underline. The criterion wasn't missed through sloppiness. It was written for an architecture that Phase 5 then replaced, and nobody went back to renegotiate it.

The scan path is a smaller version of the same thing. Hoisting the row-scan destinations out of the loop, interning the code strings, and scanning the ids as plain integers cut the report's allocations **16.00 M → 10.00 M at 1M rows** (−38%) and bytes **232.2 → 144.1 MB**. Wall across all three arms: **1.244 / 1.234 / 1.211 s**, flat, inside the noise band, because the measuring box is I/O-bound. The allocation win is proven; the latency win is *unproven* and only expected at 0.25 vCPU where CPU is scarce. That's recorded as an open question rather than a result.

(About 6 of the ~10 remaining allocations per row aren't ours: the driver boxes every column into a `driver.Value` before `database/sql` ever sees it. That part is a floor nothing in our code can move.)

### The gate that fired

The office-wide coverage read returns every overlapping coverage the office owns, **including patients who never visited**. Call the ratio of roster patients to visiting patients *R*. Phase 5's memory is linear in *R*, and I had pre-written a stop-and-redesign gate at R = 10.

At 1M rows:

| R | new Phase A | its peak heap | the path it replaced | margin |
|---|---|---|---|---|
| 1.0 | 206.5 ms | 50.5 MB | 2,509.4 ms | **12.2× faster** |
| 2.5 | 688.3 ms | 141.2 MB | 2,641.2 ms | 3.8× faster |
| 10.0 | **4,694.0 ms** | **430.4 MB** | 2,606.7 ms | **1.80× slower** |

At R = 10 the new design is worse than what it replaced on both axes, at a peak heap that has reached the instance's soft limit.

The sweep ran at two seed sizes, 200k items and 1M, exactly 5× the data. At R = 1 and R = 2.5, Phase A's wall tracks that faithfully (5.1× and 4.6×) and its heap grows 4.0× and 4.5×: linear in the seed, as designed. At R = 10 the wall grows **9.6×**, nearly double the data increase, while the heap grows only **3.0×**.

R = 10 at 1M is the only cell whose live set reaches the soft limit. When it does, the limit caps the heap and the collector pays the difference in CPU. The sub-linear memory growth and the super-linear time growth are one event seen from two sides.

> at roster ratio 10 the office-wide read reaches 430 MB and becomes 1.8× *slower* than the path it replaced, because it hits the memory limit and the GC pays the difference

A 12× speed win inverted into a 1.8× loss because it ran out of memory. Past that point the two axes aren't independent at all.

The super-linearity belongs to **the 512 MB budget**, not to the read shape. It wouldn't show up on a box with headroom, which is what makes it a production finding rather than a benchmark artifact.

The gate fired. I overrode it rather than clearing it, on one judgment: the product is unreleased, and no office we can currently describe sits anywhere near R = 10. That override is recorded as a risk rather than a pass, with a counter attached (every report now logs its office's R and warns at R ≥ 4.0), and with the caveat that there is no production population yet to point that counter at. An alternative exists. At 1M/R=10 a semi-join matches the office-wide read on wall (3,064 vs 3,166 ms, inside noise) and uses **39.7 MB against 374.8 MB**. At R = 1 and R = 2.5 the office-wide read wins outright (162.8 vs 878.3 ms; 534.4 vs 1,408.2 ms).

If you write one of these gates, watch for what happened to mine. "Office-wide wins at every R" had been *measured*, on a 200,000-item seed. At 1M it inverts. A conclusion drawn at one scale had quietly become a premise at another.

### The dial we didn't turn

The insurance resolution had an exchange rate you could buy at. (These three rows come from the pinned-container re-run of the Phase-4-era pre-resolve, not the M2 Pro run above. Same caveat as always about not mixing them.)

| strategy | wall | peak heap |
|---|---|---|
| all patients in one call | 1.226 s | 174.1 MB |
| chunked, 2000 | 2.355 s | 20.0 MB |
| chunked, 500 | 2.356 s | 14.1 MB |

2× faster for 12× the memory. On a 512 MB box that also has to fit a second concurrent report, that's not a purchase you make.

The bottom two rows are where the money was. The default chunk size was 2000. Dropping it to 500 gives **identical wall time to three decimal places and 30% less peak memory.** The 2000 was buying nothing. Someone picked a round number once, and it had been costing 6 MB of headroom per report ever since.

## What it actually does on the production instance

Everything above is 1 vCPU. The instance is a quarter of that, so at the end we measured the real configuration: `--cpus=0.25 --memory=512m`, `GOMAXPROCS=1`, three runs averaged, on a realistic fixture: 20 items per record, 90% of patients visiting once a month and 10% twice or more, a 70/20/10 insurance mix, with rows from other clinics, past months, non-visiting insured patients and non-billable record types all mixed in as ballast.

| rows | patients (1 month) | `none` | `tag` | `staff` | peak heap |
|---:|---:|---:|---:|---:|---:|
| 100k | 4,386 | 0.204 s | 0.229 s | 0.292 s | 10.2 MB |
| 500k | 21,930 | 1.162 s | 1.065 s | 1.268 s | 31.3 MB |
| 1M | 43,860 | 2.198 s | 2.233 s | 2.702 s | 89.7 MB |
| 2M | 87,719 | 4.602 s | 4.668 s | 5.634 s | 161.1 MB |
| 5M | 219,298 | 12.198 s | 13.069 s | 13.633 s | 360.7 MB |
| **10M** | **438,596** | **28.602 s** | **30.606 s** | **37.138 s** | **502.2 MB** |

The same fixture at 1 vCPU / 8 GB runs 100k in 0.067 s and 10M in 8.696 s, so **the production configuration is consistently 3.0–3.9× slower at every tier.** CPU-bound, uniformly. That's the fifth reversal cashing out: buying CPU is now the cheapest available speedup.

**Memory is not the limit.** At 10M the `staff` mode was measured separately because running all three modes in one process got the third SIGKILLed. Run alone, it completes at 512 MB, 640 MB, 768 MB and 1 GB, with differences inside repetition variance. Production serves one mode per request. After six weeks of treating heap as the binding constraint, it isn't.

**The gateway is the limit.** At 10M, `tag` (30.6 s) and `staff` (37.1 s) exceed the 29-second API Gateway timeout. `none` (28.6 s) squeaks under.

**The GC bill shows up where the model says it should.** At 10M, peak heap crosses the 460 MiB soft limit, and per-row cost degrades **+43% between 1M and 10M**, from 635 to 911 ms per million rows. Same mechanism as the R = 10 finding: once the limit binds, memory pressure is paid in wall time.

The table settled a product question too. Someone wanted to raise the maximum report range from 366 days to 1827 (five years). Five years is 60 months, so at 22.8 items per patient-month that's **1,368 rows per patient**:

| threshold | rows | max active patients at a 5-year range |
|---|---:|---:|
| our 3,000 ms perf guardrail | ~1.4M | **~1,000** |
| API Gateway 29 s | ~10M | ~7,300 |

At a one-year range that same 3,000 ms guardrail holds to ~12,000 patients. At five years it holds to about a thousand. Nothing is broken; that's just how much a synchronous response can carry. Going to five years means async, paging, or a per-office guard, and someone on the product side gets to pick, now with a number in front of them.

## Keeping it correct while doing all this

None of the above is worth anything if the report's numbers changed.

Every candidate in every bake-off was gated by a **differential test** asserting data-equal output against the previous implementation before it was allowed to win. Phase 5 shipped with no feature flag and no shadow pass, so that generative differential was, in the plan's words, *"what stands between a duplicated rule and a wrong bucket in production."* It was hardened accordingly: iterations raised 300 → 500, a self-check added that fails loudly if the seed starves any dimension, and the two paths handed independent shuffled permutations of the same data.

There's also a mutation-testing step written into the plan: deliberately invert a business rule, re-run, confirm the differential *fails*:

> A differential that cannot fail is worse than no test, and with no shadow pass this is the only guard.

That turned out to be justified. Follow-up work found two existing tests that **passed under both mutations tried**, one of which could only have detected a regression on the last day of a month between 15:00 and 24:00 UTC, about 1.2% of possible run times.

## A postscript on throwing the harness away

The benchmark harness no longer exists. It was added on a branch and deleted on the same branch, six files, once the capacity work was done. It needed a dedicated MySQL and tens of minutes to build a 10M-row fixture; it could never run in CI.

What got kept is a document with the numbers in it and a recipe for rebuilding the harness if anyone needs to re-measure. Numbers only mean anything with their conditions attached, which is why that document is mostly conditions.

It carries the ±30% warning from the top of this post, for the same reason: those historical comparison cells are single runs.

There's one detail in that rebuild recipe I keep telling people about. The original fixture seeder buffered every row in memory before inserting, so at 10M rows it exhausted 12 GB and **inserted nothing at all**. It had to be rewritten to stream its inserts.

The benchmark that proved the report shouldn't hold all its rows in memory was, itself, holding all its rows in memory.

## What I'd tell you

1. **Measure your noise floor before you measure anything else.** Run one configuration twice. Whatever spread you get is the smallest effect you're entitled to an opinion about. Mine was 16% while I was making 0.2% decisions, and 1–3% once I fixed it. That alone accounts for two of my five bad conclusions. The other three were a confounded variable, an expired conclusion, and an unchecked extrapolation, none of which more repetitions would have caught.
2. **Streaming beats buffering on both axes, within a read shape.** Office-wide load-all: 1.290 s, 278.6 MB. Cursor-streamed, same join: 1.169 s, 4.1 MB. When you think you're trading, first check whether you're just materializing something nobody needs.
3. **Round-trip elimination, though, is bought with memory.** A single streaming pass can't pause to look things up, so everything it needs must be pre-resolved and held. That's why peak heap went 55.7 → 392.6 → 506.5 MB while wall time kept falling. Know which kind of change you're making.
4. **Make the accumulator `O(output)`, not `O(input)`.** Phase 1 dropped 10M-row peak heap 28×, and nothing in four subsequent phases beat it.
5. **Retained heap and peak heap are different numbers.** Peak samples `HeapAlloc` without collecting; with `GOGC=100` it runs near 2× the live set. Force two GCs and measure what survives. Two-thirds of my scariest number was garbage, and scaling the wrong one made a spec wrong by 4×.
6. **An allocation win is not a latency win.** −24.3% allocations at every tier, to the decimal, next to a wall saving that decayed to −2.1%. Both real, but not the same claim.
7. **Never extrapolate a failure you haven't observed.** "It OOMs past 1.15M" was a line through three points. It degrades at 394 MB instead. Predicted death and measured degradation call for different responses.
8. **Change one variable.** A missing index made me write off an entire read shape as fatal, by 26×, and the error survived a month because code and schema were moving together.
9. **Profile the profiler.** "91% DB-bound" was a timing callback that included the row scan. `EXPLAIN ANALYZE` said 1.5 s of 5.5 s.
10. **Re-derive acceptance criteria when the architecture changes.** Phase 5's heap target was set before the structure that blows it existed.
11. **Check the constants you inherited.** Chunk size 2000 → 500: identical latency, 30% less memory, free.
12. **Measure the configuration you actually deploy.** Everything looked fine at 1 vCPU. Production is 3.0–3.9× slower, and that only turned up because someone finally pinned a container at 0.25.
13. **Write down the miss.** Every number here that embarrasses me is in a design doc with its conditions attached. That's the only reason I could re-measure it and find out I was wrong.

Going in, I expected to spend the project turning a dial between fast and small. Most of the time the dial wasn't connected to anything: the memory was being spent on work nobody needed, and deleting that work made things faster for free. Where it did connect, I turned it toward memory four times on purpose, because killing a round trip was worth more than the heap it cost. And at R = 10 and ten million rows it stopped being a dial at all. The heap filled up, the collector started charging for it, and the pressure came back as latency.

Measure your noise floor first. And go look at whatever chunk size someone typed in two years ago.

## References

**Go**

- [`database/sql`](https://pkg.go.dev/database/sql): [`DB.Query`](https://pkg.go.dev/database/sql#DB.Query) + [`Rows.Next`](https://pkg.go.dev/database/sql#Rows.Next), the cursor iteration a streamed join needs
- [`runtime/debug.SetMemoryLimit`](https://pkg.go.dev/runtime/debug#SetMemoryLimit): the soft limit used to model the production instance inside the timed section
- [`runtime.ReadMemStats`](https://pkg.go.dev/runtime#ReadMemStats) and [`runtime.KeepAlive`](https://pkg.go.dev/runtime#KeepAlive): the retained-heap probe
- [A Guide to the Go Garbage Collector](https://tip.golang.org/doc/gc-guide): the `GOGC` pacing that makes a sampled peak ≈ 2× the live set
- [`testing`: benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks), [`B.ReportMetric`](https://pkg.go.dev/testing#B.ReportMetric), and [`benchstat`](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat): custom metrics, and the "is this bigger than noise" question
- [`runtime/pprof`](https://pkg.go.dev/runtime/pprof): heap profiles, and `-base` for attributing a diff

**MySQL**

- [`EXPLAIN ANALYZE`](https://dev.mysql.com/doc/refman/8.0/en/explain.html#explain-analyze): how "91% DB-bound" and "aggregate using temporary table" were settled
- [`ANALYZE TABLE`](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html): optimizer statistics, which flipped a plan mid-investigation
- [InnoDB buffer pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html): why every benchmark arm needs a discarded warm-up run

**Containers**

- [Docker resource constraints](https://docs.docker.com/engine/containers/resource_constraints/): `--cpus` and `--memory`, for measuring against the instance you actually deploy to

**Related posts**

- [Migrating a billing system from Rails to Go](/migrating-billing-rails-to-go)
- [Dual writes and the strangler fig](/dual-writes-and-the-strangler-fig)
- [MySQL transactions — same Go pattern, different ways to lose](/mysql-transactions-silent-rollback)
