# Audit #03 — synchronous fallback reasons (v1.0.121)

Inspected the exact source file from user-uploaded ZIP:
`runtime/src/nnsdk/nvn/executor.cpp`, function `Executor::pipeline` around lines 7290–7438.

## Existing routing conditions

- `async_on = async_pipelines_ && !per_stage_keys_ && !key.skip_gs`
- `async = async_on && rp_recurring_`
- When pipeline hash is pending and `async` is true, the draw is skipped while background job continues.
- When pipeline hash is pending but `async` is false, hash is marked abandoned and `retry=true`.
- Known pipeline is allowed to be synchronous within the known-pipeline time budget.
- Pipeline recording eligibility starts false when Vertex stage has `count_check`.
- `recordable` also requires every relevant base cache key and every generated variant cache key to be nonzero.
- Async queueing occurs only when `async && !retry && recordable`.

## Diagnostic categories to instrument (not yet changed)

1. `async_pipelines_ == false` (global disabled / serial setting)
2. `per_stage_keys_`
3. `key.skip_gs`
4. `!rp_recurring_`
5. `retry` / abandoned pending hash
6. Vertex `count_check` makes recordability false
7. Stage base cache key missing
8. Variant cache key missing (variant generation / stage-specific path)
9. Known-pipeline budget path (synchronous on purpose)
10. Fallback synchronous pipeline create

## Recommended next patch

Add lightweight counters grouped by reason, increment only at decision points; report them in the next profiler/log report. Avoid per-draw log spam, dynamic allocations, or formatting inside hot path. Counters should be reset at report boundaries or shown as totals, and should include a total sync-created count so the categories can be cross-checked.

## Safety / performance

No rendering behavior or scheduling changes in this audit. This is instrumentation-only, designed to reveal which gates account for sync pipeline creation on Mali-G615 MC2 before any optimization is attempted.
