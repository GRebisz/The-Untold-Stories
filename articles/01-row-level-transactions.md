# How Row-Level Transactions Saved Our Grid Performance (and Our Sanity)

*When your "save" button takes 18 seconds, something has gone fundamentally wrong. Here's how we fixed it.*

---

I'm going to tell you about a bug that wasn't a bug. It was a design decision that made perfect sense three years ago, quietly became a bottleneck, and then one day decided to bring a production system to its knees during a client demo.

If you've ever worked on enterprise SaaS — the kind where thousands of users hammer the same features during compensation cycles, performance reviews, or end-of-quarter crunches — you know exactly the kind of surprise I'm talking about.

## The Setup

Our platform has a feature that most enterprise applications have in some form: editable data grids. Think spreadsheets, but backed by a database, with validation rules, role-based access, and audit trails. Users open a grid, edit values across dozens of rows, and hit save.

Simple enough, right?

Under the hood, the original save operation worked like this: for every single cell the user modified, the system opened a database transaction, validated the value, wrote it, and committed. One cell, one transaction. Repeat for every modified cell in the grid.

When the feature was built, grids had maybe 10-15 editable fields. One transaction per value was fine. Nobody noticed. Nobody complained.

Fast forward to a client deployment with 200+ editable fields per grid, multiplied by hundreds of concurrent users during their annual compensation cycle. Suddenly, that "save" button was triggering hundreds of individual database transactions in sequence. Response times looked like this:

| What We Measured | What We Saw |
|-----------------|-------------|
| Mean response time | 1.32 seconds |
| Median | 964ms |
| 95th percentile | 1.51 seconds |
| Worst case | **18.3 seconds** |

An 18-second save operation. During peak usage. On a screen where managers are making salary decisions for their entire department.

That's not a performance issue. That's a trust issue. When your application hangs for 18 seconds, users don't think "the database is under load." They think "this software is broken."

## The Obvious Fix (That We Didn't Do)

The instinct is to batch everything into a single transaction. Wrap all the values in one big commit. It's faster, it's simpler, and it's wrong — at least for our use case.

Here's why: if you batch 200 values into one transaction and value #187 fails validation, you've just rolled back 186 perfectly good saves. The user sees an error, their entire grid reverts, and they have no idea which field caused the problem. In a compensation management context, that means a manager who just spent 20 minutes adjusting salaries across their team loses all of it because one field had an integer overflow.

We needed something in between — not one transaction per cell, and not one transaction for the entire grid. We needed row-level transactions.

## The Fix

The concept is straightforward: group all the cells in a single row into one transaction. If a row saves successfully, it's committed and done. If a row fails, only that row rolls back. Every other row's changes are preserved.

This gives you three things that matter in enterprise software:

**Partial success.** Twenty rows saved, one failed? The user keeps their twenty good rows and gets a clear error pointing at the one that didn't make it. They fix it and move on. No lost work.

**Meaningful errors.** Because we're validating at the row level, we can tell the user exactly what went wrong: "Row 14, column 'Annual Bonus' — value exceeds maximum allowed integer." Not "save failed, please try again."

**An audit trail that makes sense.** Every validation error gets logged to a dedicated error table with the process context, the field that failed, the value that was attempted, and who attempted it. When a system administrator asks "why couldn't this manager save their grid last Tuesday?" — we have the answer in seconds.

## The Results

We ran the optimized implementation through the same load tests, same dataset, same concurrency levels:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Mean | 1.32s | 588ms | **55% faster** |
| Median | 964ms | 464ms | **52% faster** |
| 90th percentile | 1.36s | 669ms | **51% faster** |
| 95th percentile | 1.51s | 806ms | **47% faster** |
| Worst case | 18.3s | 7.3s | **60% faster** |

The mean response time dropped by more than half. The worst-case scenario went from "user thinks the app crashed" to "user notices a brief pause." And that's before we even looked at further optimization — this was purely the architectural change from per-cell to per-row transactions.

## What We Shipped (Besides the Fix)

The performance win was the headline, but the supporting work is what made this a proper enterprise feature rather than just a speed patch:

**Feature flagging.** The entire optimization sits behind a configuration parameter. Flip it to `true` to opt in, `false` to revert to the original behavior. No code deployment needed to switch. This matters when you're running a multi-tenant platform where different clients are on different release cycles.

**Error logging infrastructure.** A dedicated database table for grid save errors with indexed queries for process, step, and date range. Batch-inserted asynchronously so the error logging itself doesn't become a performance bottleneck. We borrowed the pattern from our analytics pipeline — collect errors in a thread-safe buffer, flush every 5 seconds or every 100 entries, whichever comes first.

**Validation that actually validates.** Integer bounds checking, numeric precision validation, date format verification — all happening before the database transaction starts, not after it fails with a cryptic SQL exception.

## The Takeaway

The lesson here isn't "batch your transactions" — any mid-level developer knows that. The lesson is about the granularity of your failure boundary.

When you design a save operation, you're really making a decision about how much work a user loses when something goes wrong. One transaction per cell means no user ever loses more than one value — but the performance cost is brutal at scale. One transaction for everything means great performance — but a single bad value nukes the entire operation.

Row-level transactions hit the sweet spot for grid-based data entry: the failure boundary matches the user's mental model. Users think in rows. They fill out a row, move to the next one. If a row fails, losing that row's changes makes sense to them. Losing their entire grid doesn't.

The best performance optimization isn't always the fastest one. It's the one that fails gracefully when things go wrong — because in enterprise software, things always go wrong.

---

*Gregory Rebisz is a software developer and technical content writer specializing in enterprise SaaS, backend performance, and the kind of problems you only discover at scale. He's based in South Africa and available for freelance technical writing.*
