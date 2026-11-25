# AUDIT 07 — Async queue wait and worker throughput diagnostics (v1.0.121)

## Purpose

Extend the pipeline diagnostics from the routing audit with measurements of the asynchronous worker queue. This is an observability-only change: it does not alter worker count, thread priority, CPU affinity, pipeline routing, or the graphics pipeline creation path.

## Source findings

- Async workers are created lazily when the first eligible graphics pipeline is queued.
- The default worker count is 2; runtime flags can select 1, 3, or 4.
- Each worker calls `setpriority(..., 10)` and `pin_background_thread()`.
- In `runtime/src/nnsdk/os.cpp`, `pin_background_thread()` builds an affinity mask from usable CPUs excluding the detected big-core mask, then applies that mask when affinity is enabled. This is a best-effort policy; the helper can do nothing when the CPU mask is unavailable or affinity is disabled.
- The queue is FIFO (`async_queue_`); a worker removes a job under `async_mutex_`, then translates cached shader modules and creates the Vulkan graphics pipeline.

## Patch contents

File: `patches/gt30_async_queue_diagnostics_1.0.121.diff`

The patch is intended to be applied **after** `gt30_pipeline_routing_diagnostics_1.0.121.diff`.

It adds:
- An enqueue timestamp to `AsyncJob`.
- Average queue wait time, measured from enqueue until a worker pops the job.
- Number of jobs completed during the profiling interval.
- Number of driver pipeline-creation calls and their average measured duration.
- Successful, failed, and abandoned job counts.

Counter semantics:
- Success means the job was not abandoned, the driver returned `VK_SUCCESS`, and a non-null pipeline was produced.
- Failed means the job was not abandoned and did not meet the success condition. It can include shader-cache/module failures or driver/pipeline failures.
- Abandoned is reported separately and is mutually exclusive with success/failed in this report; it means the foreground submit path took over while the worker was processing the job.
- Average queue wait is averaged across completed jobs. Jobs still waiting or processing at report time are not included.
- Driver average is averaged only across jobs that reached `make_graphics_pipeline`. The timer starts after acquiring `pipeline_cache_lock_`, so it excludes queueing and cache-lock wait.

## Validation performed

- Patch #04 was applied to a fresh extraction of `mhrise-recomp-1.0.121.zip`.
- The local working extraction was inspected and contains the expected instrumentation. The exact GitHub patch files have not yet been independently re-applied in a clean-room test; re-run sequential patch application before treating them as release-ready.
- SHA-256 of patch #05: `baf04a027b98907026275140b81c050640f3ed0d5fc9f37dfc4a082dd89f7dd9`.

## Runtime collection

Enable `perf_stats` so `profile_frame()` emits the profiling report with less extra logging overhead than `verbose_log`. Add `profile_30` for a 30-frame reporting interval; otherwise the normal interval is 300 frames. `profile_30` alone does not enable the report. Reproduce the same route and scene for each worker setting (1, 2, 3, 4), then compare:
- Average queue wait.
- Average driver creation time.
- Success/failure/abandoned counts.
- Existing synchronous pipeline routing counts and slow-submit spikes.
- Frame pacing and device temperature.

Interpretation:
- High queue wait with modest driver time suggests worker throughput or scheduling is limiting.
- Low queue wait with high driver time suggests driver pipeline creation itself is expensive.
- Low values in both but continued stalls suggest another synchronous path or non-pipeline work should be investigated.

## Limitations / status

This is source instrumentation only. No Android NDK build, APK build, device runtime test, or thermal test has been performed. The report is emitted only when the existing verbose/performance profiling path is enabled. Do not change the default worker count based on this audit alone.
