# Audit #06 — async pipeline worker policy and test plan (v1.0.121)

## Source inspected

- `runtime/src/nnsdk/nvn/executor.cpp`, `Executor::pipeline` around lines 7290–7455.
- `Executor::async_loop` around lines 11479–11542.
- Worker configuration and profiling code in the same source.

## Verified existing behavior

1. Async pipeline workers default to **2**.
2. Startup flags select **4**, **3**, or **1** workers (`pipeline_threads_4`, `pipeline_threads_3`, `pipeline_threads_1`); otherwise 2.
3. Workers are detached and run `async_loop()`.
4. Each worker sets a low scheduling priority and calls `pin_background_thread()`.
5. Each worker takes one queued job, reconstructs shader modules from cached SPIR-V, then calls `make_graphics_pipeline`.
6. The worker holds a shared lock on `pipeline_cache_lock_` while creating the graphics pipeline.
7. If the async result is abandoned, the created pipeline is discarded; failed or unavailable translations are routed to `async_retry_` for synchronous handling.
8. The existing source already logs the selected worker count when async workers start.

## Optimization decision

Do not change the default worker count in this patch.

Why:
- More workers may finish queued pipelines sooner, but can increase CPU scheduling pressure and heat.
- The phone's background workers are intentionally assigned lower priority and pinned by `pin_background_thread()`; overriding that policy without evidence risks competing with the game/render thread.
- Pipeline creation uses a shared pipeline-cache lock in this path. The actual driver behavior and throughput must be measured before increasing concurrency.
- One worker may reduce background load but leave more pipeline jobs queued and extend the period during which affected draws are skipped.

## Controlled test plan

Use the same game build, save, location, graphics settings, and route for each run. Clear neither shader/pipeline caches nor game data between runs unless a separate cold-cache test is intentionally being performed.

1. Baseline: default 2 workers.
2. Test 1 worker with `pipeline_threads_1`.
3. Test 3 workers with `pipeline_threads_3`.
4. Test 4 workers with `pipeline_threads_4`.

For each run, capture:
- pipeline routing counters from the refined diagnostics patch;
- async worker count line;
- synchronous pipeline creation time and total pipeline count;
- slow-submit spikes and their breakdown;
- average frame pacing / FPS over the same route;
- device temperature and whether performance degrades later in the session.

Change only the worker-count flag between runs. Repeat each run at least twice if results vary.

## Decision criteria

Prefer the worker count that reduces visible hitching and slow-submit stalls without causing sustained frame-time regression or excessive heat. Do not choose based on peak FPS alone.

## Status

This is an analysis and test plan only. No worker scheduling, priority, pinning, or default-count behavior has been changed. No APK has been built or tested.
