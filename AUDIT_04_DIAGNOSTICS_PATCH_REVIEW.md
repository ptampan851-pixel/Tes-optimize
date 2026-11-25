# Audit #04 — review of pipeline routing diagnostics patch

## Scope

Reviewed the committed draft:
`patches/gt30_pipeline_routing_diagnostics_1.0.121.diff`

This is a static review of the diff, not a build or runtime validation.

## Findings

### 1. async_queued is currently an attempted-queue counter

The counter is incremented before entering the existing `if (async && !retry && recordable)` enqueue block. It therefore measures candidates reaching the enqueue branch, not confirmed successful queue insertion. Rename it to `async_queue_candidates`, or move the increment to the exact successful insertion point after inspecting the complete source.

### 2. No sync_total counter

The report has reason counters but no total count for actual synchronous pipeline creations. Without a total measured at the actual synchronous creation site, the reason counters cannot be reconciled against real sync creations.

Add `sync_total` at the actual sync pipeline creation point, not at the earlier routing decision.

### 3. unrecordable is a combined bucket

The current counter groups all recordability failures together. The source audit identified at least:
- Vertex stage `count_check`
- Missing base stage cache key
- Missing generated variant cache key

Separate these only after confirming exact local variables and control flow in the full v1.0.121 source.

### 4. Counters need concurrency review

Before relying on plain `uint64_t` increments, confirm whether `Executor::pipeline` can execute concurrently on multiple threads. If it can, use an appropriate low-overhead atomic or thread-local aggregation strategy. Do not assume thread safety from the current diff.

## Next implementation requirements

1. Inspect the complete current v1.0.121 `executor.cpp` and relevant report-emission code.
2. Count confirmed async insertions at the successful queue insertion point.
3. Count actual synchronous creations at the synchronous creation point.
4. Record mutually exclusive routing reasons, and distinguish the three unrecordable causes if source flow supports it.
5. Confirm counter reset/report boundaries and thread-safety.
6. Run a source-level diff review and build if the Android toolchain is available.

## Decision

Do not implement a performance behavior change yet. The diagnostic patch should be corrected and validated first. The original source ZIP is not currently attached to this conversation, so exact insertion points cannot be safely re-verified in this session.
