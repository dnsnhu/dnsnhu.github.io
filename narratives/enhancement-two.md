---
layout: default
title: "Enhancement Two: Algorithms and Data Structures"
description: "Added an analytics layer: a single pass moving average, a monotonic deque for the sliding minimum and maximum, streak detection, and a least squares goal projection."
permalink: /narratives/enhancement-two/
---


# Enhancement Two: Algorithms and Data Structures

[Back to the portfolio]({{ site.baseurl }}/) · [Original source]({{ site.baseurl }}/artifact/original/) · [Enhanced source]({{ site.baseurl }}/artifact/enhanced/)

Added an analytics layer: a single pass moving average, a monotonic deque for the sliding minimum and maximum, streak detection, and a least squares goal projection.

## Files in this enhancement

**Added (10)**

`AnalyticsBenchmark.java`, `AnalyticsEngine.java`, `AnalyticsEngineTest.java`, `AnalyticsSnapshot.java`, `GoalProjection.java`, `MonotonicWindow.java`, `MonotonicWindowTest.java`, `StreakSummary.java`, `WeightAnalytics.java`, `WeightAnalyticsTest.java`

**Changed (7)**

`AddWeightActivity.java`, `SqliteWeightDao.java`, `WeightAdapter.java`, `WeightItem.java`, `WeightListActivity.java`, `WeightListViewModel.java`, `WeightRepository.java`

Full source for every file is on the [enhanced source]({{ site.baseurl }}/artifact/enhanced/) page.

## Narrative

### Description of the Artifact

The artifact is an Android weight tracking application written in Java. I built it as Project Three in CS 360 Mobile Architecture and Programming. It is the same artifact I examined in my Module Two code review and the same one I restructured for the previous enhancement, so the version I started from here already has a repository, a ViewModel per screen, and database work off the main thread.

The application registers and signs in a user, lists logged weights in a RecyclerView, adds and edits entries, stores a goal weight, and sends an SMS when the goal is reached. Persistence is SQLite. The original was 546 lines across seven classes.

What matters for this enhancement is what the application did not do. It stored numbers and rendered them as text. There was no average, no trend, no minimum or maximum, no sorting, and no projection. The main query was a bare SELECT with no WHERE, no ORDER BY, and no LIMIT, and the list was rebuilt from scratch after every change. For an application whose entire purpose is tracking a number over time, the absence of any computation on that number is the real gap.

### Justification for Inclusion and Summary of Improvements

I selected this artifact for the algorithms category because the missing calculations are the ones where the choice of algorithm actually decides whether the feature holds up. Averaging, tracking extremes over a window, and fitting a trend all have an obvious implementation that works fine on a short log and falls apart on a long one. That makes it a real design problem rather than an exercise.

The enhancement adds an analytics layer. The pieces a reviewer should look at are these:

- WeightAnalytics.movingAverage keeps a running sum and adjusts it by the entering and leaving element, so the whole series comes out in one pass. Recomputing each window separately means summing k values n times. On 100,000 entries with a 30 entry window the single pass version measured 0.51 milliseconds against 4.98 for the recomputing version, about ten times faster.

- MonotonicWindow tracks the sliding minimum and maximum with two deques of indices. When a value arrives, every index at the back holding a worse value is discarded, because a value that is both worse and older can never win again. Each index is pushed once and popped at most once, which bounds the total work regardless of how wide the window is.

- WeightAnalytics.projectToGoal fits a least squares line to the recent entries from running sums of x, y, x squared, and x times y, so the fit and the projection come out of a single pass with no intermediate storage.

- WeightAnalytics.detectStreaks finds runs of consecutive entries moving the same direction in one comparison per entry.

- AnalyticsEngine keeps all of this current, with a fast path for appends and a rebuild for edits and deletes. More on that below, because the split is the main decision in this enhancement.

- WeightItem now carries a float rather than a String. The value used to change type three times on a round trip, and any calculation would have had to parse every element before it could start. That type choice is the reason the original computed nothing.

- WeightAdapter now uses DiffUtil. Deleting one entry dispatches one row removal instead of rebinding the whole list, and scroll position survives.

- The main query carries ORDER BY. The original had no ordering anywhere, so rows came back in whatever order SQLite chose and only looked chronological by accident. Every calculation here depends on the series being in recording order, so leaving that to chance stopped being acceptable the moment anything was computed from it.

Verification is 35 JUnit tests across the three analytics classes, on top of the 31 already covering the previous enhancement. The moving average is checked against a recomputation of every window, and the sliding minimum and maximum are checked against a brute force scan on two thousand random entries, so the fast implementations are compared against the slow ones they replace rather than against my expectations of them.

### Course Outcome Coverage

I planned this enhancement to cover Outcome Three, on designing and evaluating computing solutions using algorithmic principles while managing the trade-offs involved in design choices. I met it, and the trade-offs were real rather than decorative.

The first is where the computation happens. SQLite supports window functions, so the moving average could have been pushed into the query. I kept it in the application because that makes the analytics a pure function over a list, testable on the JVM with no device, and it keeps the query a plain scan. The cost is pulling rows into memory that the database could have aggregated. For a personal weight log, where the series is small enough to hold anyway, the testability is worth more.

The second is the append and rebuild split. The monotonic window earns its bound by discarding indices it has proved can never matter again. Removing an entry from the middle can make a discarded index relevant once more, and the structure no longer holds what it would need to bring it back. Supporting that would mean keeping the discarded candidates, which is the exact cost the structure exists to avoid. So appends update incrementally and edits and deletes rebuild. Appends are the common case in a weight log and stay cheap as history grows; edits are rare and pay a linear cost.

The third is whether the structure is worth it at the size this application actually runs at, and this is the one I got wrong at first. I assumed the deque would win everywhere. Timing it across window widths on 100,000 entries showed the deque flat at roughly seven milliseconds from a window of 7 up to a window of 10,000, while the rescan grew from 2.96 milliseconds to 1,441. The two cross near a window of 30. Below that the rescan is genuinely faster, because scanning a couple of dozen floats in a row is cache friendly and the deque pays for boxing its indices. My default window is 30, which is almost exactly the crossover. I kept the deque anyway, and the reason is the shape of the two curves rather than the value at one point: the rescan cost rises with the window and the deque cost does not, so the deque is the choice that does not need revisiting if the window widens.

I also want to be careful about how that claim is evidenced. Wall clock timing depends on the machine, so the test that actually asserts the complexity counts deque operations instead. It comes out at exactly 4.00 operations per entry at both 10,000 and 100,000 entries, and stays there when the window is widened to 5,000. A rescan would do work proportional to the window on every entry, so that assertion would fail for any window past six.

One update to my outcome-coverage plan. My code review had this enhancement covering Outcome Four as well, on well-founded techniques and tools. I have moved Outcome Four to the database enhancement, where replacing a hand written SQLite helper with Room is a much more direct demonstration of adopting the current standard tool for the platform. This enhancement claims Outcome Three alone. That matches the coverage I recorded in my Module Four journal.

### Reflection on the Enhancement Process

The hardest part was the one I expected to be routine. Keeping the deque correct once entries could be edited and deleted took more thought than writing it in the first place, because the structure assumes the series only ever grows at the end. I spent a while trying to patch the deques in place on a deletion before accepting that the information needed to do it correctly had already been thrown away, and that throwing it away is the whole point. Rebuilding is the less clever answer and it is the right one. What convinced me was writing down what a patch would have to store and realising it was the same set of candidates the discards eliminate.

Having two update paths created a risk I had not thought about until the code existed: they can quietly disagree. An incremental append and a full rebuild are supposed to produce identical figures, and nothing enforces that. The test I am most glad I wrote replays the same 1,500 entry series both ways and compares every field of the result. It also let me make the rebuild path call the append path rather than duplicating the logic, which removes the drift at the source instead of just detecting it.

Two things surprised me. The first was the benchmark contradicting me, which I described above. I had already written down that the deque was faster before I ever measured it, and at the window size I had chosen the measurement said otherwise. Keeping the structure but changing the argument for it was more useful than picking whichever number looked better. The second was a test catching a logic error in the projection. My first version asked whether the trend was moving toward the goal, which reports a user who has already dropped below their goal weight as moving away from it. The application had already answered this: the goal notification fires when the logged weight is at or below the goal, so the goal is an upper bound, not a target to hit exactly. The analytics now use the same rule the rest of the application does.

The limitation I want to name is that all of this indexes entries by position rather than by date. The date column holds the output of Date.toString, which is locale dependent and does not sort, so there is no trustworthy way to turn a per-entry slope into a per-day one. The projection therefore reports entries rather than days, which is the honest unit for the data that exists. Fixing it is a column type change rather than an algorithmic one.


[Back to the portfolio]({{ site.baseurl }}/)
