# AUDIT 10 — Practical GPU profiling capture protocol (v1.0.121)

## Purpose

This document turns the existing profiling code into a controlled capture procedure. It adds no runtime patch and does not change rendering, worker scheduling, or GPU policy.

## Exact source behavior

### Periodic report

In `Executor::profile_frame()`:

- Default report period is 300 frames.
- Creating the `profile_30` flag changes the period to 30 frames.
- A report is emitted only when `perf_stats` or `verbose_log` is enabled.
- `perf_stats` enables the profile without enabling the slow-run log.
- `verbose_log` enables the detailed/slow-run log and also activates KGSL monitoring in the submit path.
- If neither flag is enabled, the function saves pipeline cache/records and returns without the performance report.

### One-shot GPU pass report

In `Executor::prof_gpu_mark()` and `Executor::prof_gpu_report()`:

- The profiler requires a timestamp query pool.
- The pool is created only when the selected Vulkan queue family has non-zero `timestampValidBits` and query-pool creation succeeds.
- The file `<NIER_DATA>/profile_gpu` is checked when the first eligible mark of a frame is reached.
- If the file exists, it is deleted immediately and the current frame is profiled.
- Marks are written before render passes and compute dispatches. The final mark closes the sample.
- The report waits until the submitted frame is completed, then sorts groups by accumulated GPU time and prints at most 40 groups.
- Labels include render-target dimensions/format and optional pass suffixes; repeated labels are accumulated together.
- The profiler has a 1024-mark cap. It is a single-frame sample, not a continuous trace.

### Per-draw mode

`profile_gpu_draws` adds marks for eligible draws in passes with no color target (the source comment specifically mentions shadow passes). It should remain disabled for the first pass-level capture because it increases query count and instrumentation overhead.

## Capture sequence

Use the same clean v1.0.121 source with the canonical combined Patch #08 applied. Do not layer the older separate routing/async patches on top.

### A. Baseline periodic report

1. In the data directory identified by the app's `NIER_DATA` environment, enable `perf_stats`.
2. Also enable `profile_30` to shorten the report interval from 300 to 30 frames.
3. Do not enable `verbose_log` yet. It adds slow-run logging and KGSL monitoring; keep the first capture as light as practical.
4. Launch the game and let the target scene settle. Traverse the same route and camera direction for each comparison.
5. Save several consecutive reports, not just one. Record GPU frame ms, executor ms/frame, GPU-wait ms/frame, load breakdown, and Patch #08 async/sync pipeline counters.
6. Note whether the scene is cold or warmed, and record actual device temperature externally.

### B. One-shot GPU pass breakdown

1. Keep the same scene and repeatable camera position.
2. Create an empty file named exactly `profile_gpu` in the same `NIER_DATA` directory.
3. Trigger a frame that reaches a render-pass or compute mark.
4. The file is consumed/deleted when profiling starts. Wait for the profiled submission to complete.
5. Save the log lines beginning with `perfil GPU del frame` and all following `perfil GPU:` lines.
6. Repeat the capture once if the result is atypical, but do not compare captures from different scene states as if they were identical.

### C. Slow-run diagnosis (only if needed)

If the periodic report indicates large executor time or the game has obvious stalls, enable `verbose_log` for a short targeted capture. Inspect `envío lento` lines for wall-clock run time, NvnSubmit thread CPU time, voluntary/involuntary context switches, and load breakdown. Disable it after capture to avoid unnecessary logging and KGSL polling during ordinary play.

Do not enable `profile_gpu_draws` unless pass-level groups leave a specific question unanswered.

## Interpretation rules

- A high GPU pass duration identifies expensive GPU work in the profiled frame; it does not prove whether the cause is shader complexity, bandwidth, fill rate, or memory pressure.
- A pass-level report is not directly interchangeable with the periodic whole-frame GPU timestamp. They use different scopes and may not cover exactly the same work.
- A large executor time with large GPU-wait time points toward waiting; a large executor time with low GPU wait needs CPU/load breakdown.
- A slow wall-clock NvnSubmit run with low thread CPU time suggests waiting or descheduling rather than pure CPU computation.
- A long pipeline driver call and high synchronous fallback counters point to a pipeline/driver stall, not necessarily a GPU rendering bottleneck.
- Do not change worker count, thread priority, CPU affinity, dynamic-resolution levels, or GPU settings during the baseline captures. Change one variable only after a repeatable baseline exists.

## Minimal data to send for analysis

- Several consecutive `perf_stats` reports from the same warmed scene.
- One `profile_gpu` pass report from that scene.
- If stalls occur, the corresponding `envío lento` lines from a short `verbose_log` capture.
- Device temperature, graphics preset/resolution, and whether the run was cold or warmed.
- The exact worker flag used (default, `pipeline_threads_1`, `pipeline_threads_3`, or `pipeline_threads_4`).

## Validation boundary

This is a source-based capture guide for v1.0.121. No source patch was applied, no Android NDK/APK build was run, and no device capture or thermal test was performed.
