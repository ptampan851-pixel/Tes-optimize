# AUDIT 09 — CPU/GPU bottleneck separation (v1.0.121)

## Scope

This audit inspects the original, clean v1.0.121 source to determine which CPU-side and GPU-side measurements already exist before adding more instrumentation. It does not change runtime behavior and does not add a patch.

## Findings

### 1. The source already has a GPU frame-time measurement

In `runtime/src/nnsdk/nvn/executor.cpp`:

- The executor creates timestamp query pools for in-flight frame stamps when the Vulkan queue family reports non-zero `timestampValidBits`.
- `write_stamps()` reads the completed query results and computes `frame_gpu_ns_` from the start/end GPU timestamps.
- `profile_frame()` reports this as `resolución dinámica: GPU del frame ... ms`.
- If timestamp queries are unavailable, the source can fall back to the CPU clock path (`gpu_stamps_`).

This is the broad GPU frame duration metric. It is not a breakdown of CPU submission time.

### 2. A per-render-pass GPU profiler already exists

The opt-in profiler uses `<NIER_DATA>/profile_gpu` as a one-shot trigger:

- `prof_gpu_mark()` writes Vulkan timestamp queries around render passes and compute dispatches.
- The completed frame is collected by `prof_gpu_report()`.
- The log groups time by pass label and compute, sorts by GPU duration, and prints the largest groups.
- The query pool is unavailable when the device/queue does not expose timestamp bits or query-pool creation fails.
- `profile_gpu_draws` can add per-draw marks in eligible passes, but should not be enabled for the first capture because it increases query volume and changes profiling overhead.

This is the best existing tool for finding which GPU passes dominate a heavy scene. It is a single-frame diagnostic, not a continuous low-overhead frame-time series.

### 3. CPU executor time and GPU wait are already separated

The periodic `perf_stats` / `verbose_log` report prints:

- `prof_run_us_`: total time spent in executor `run()`, including GPU wait.
- `prof_gpu_wait_us_`: the part spent waiting for the GPU.
- Draw, render-pass, dispatch, and load/upload summaries.
- A sampled per-draw CPU breakdown: `prof_timing_` measures one of every eight frames when `perf_stats` or `verbose_log` is active. The fine report separates pipeline work, descriptors, state, vertices, and draw work.

Therefore, executor CPU time excluding its measured GPU wait can be estimated as:
`executor CPU-side remainder ≈ prof_run_us_ - prof_gpu_wait_us_`.
This is an aggregate estimate; it is not a full measurement of all game-thread CPU work.

### 4. Slow submit diagnostics already expose CPU-thread time and scheduling clues

For a slow executor run (at least 50 ms), when `log_state_` is enabled, the log prints:

- Wall-clock run duration.
- CPU time consumed by the NvnSubmit thread during that run.
- Voluntary and involuntary context-switch counts.
- NvnSubmit nice value.
- Load-stage breakdown (texture upload, deswizzle, image creation, pipeline work, GPU wait, queue submit, and others).

This distinguishes a long wall-clock stall from a CPU-heavy run. A large wall-clock duration with much lower thread CPU time points toward waiting or descheduling, not necessarily expensive CPU computation.

### 5. Existing ADPF path measures critical-thread CPU work

`runtime/src/nnsdk/nvn/perf_hint.cpp` samples per-thread CPU clocks for the critical threads and reports the most heavily loaded thread's work duration to Android Performance Hint. This helps Android scheduling, but it is not a GPU-time measurement and does not currently provide a complete user-facing per-thread diagnostic report.

## Interpretation for the GT 30 5G / Mali-G615

The earlier report showed heavy-scene GPU frame times in the tens of milliseconds and a separate very large synchronous pipeline-creation stall. Those are different classes of bottleneck:

- A high GPU frame time that persists after pipeline warm-up indicates GPU/render workload pressure.
- A long NvnSubmit run dominated by driver pipeline creation indicates a CPU/driver-side pipeline stall.
- A long run with low NvnSubmit CPU time and high GPU-wait time indicates waiting for GPU completion or queue progress.
- A large executor CPU-side remainder with expensive draw/pipeline/descriptor sections indicates CPU submission overhead.
- Queue wait and worker completion data from the combined Patch #08 should be interpreted alongside these existing measurements, not as a substitute for them.

Do not infer that the Mali GPU is the only bottleneck from a high GPU frame time, or that adding pipeline workers will fix a GPU-bound scene.

## Recommended capture sequence

Keep the canonical Patch #08 applied to clean v1.0.121. Do not apply the older separate routing and async patches.

1. Enable `perf_stats` and `profile_30`. The former enables the report; the latter only changes the report interval to 30 frames.
2. Capture a repeatable route/scene after shaders and pipelines have warmed up. Record GPU frame ms, executor ms/frame, GPU-wait ms/frame, load summaries, async queue wait, worker completion/failure/abandoned counts, and sync-routing reasons.
3. Repeat the same route and camera/scene conditions. Avoid comparing a cold shader traversal with a warmed traversal.
4. For a specific heavy scene, create the one-shot `profile_gpu` file in the data directory, then capture the resulting per-pass GPU report. Do this separately from the baseline run.
5. Only if the per-pass report is insufficient, consider `profile_gpu_draws` for a short targeted capture.
6. Compare worker settings 1, 2, 3, and 4 only after the baseline has been captured. Keep worker priority, affinity, and default worker count unchanged during the first comparison.
7. Record device temperature and whether the run was thermally settled. The source's thermal/performance hints and GPU timings do not replace an actual device temperature reading.

## Decision

No new CPU/GPU timing patch is justified yet. The source already exposes the measurements needed for the next diagnostic pass. The next useful step is to collect a controlled device log and correlate:

- GPU frame time and per-pass GPU time;
- executor wall time, CPU-side remainder, and GPU wait;
- NvnSubmit CPU time and context switches on slow runs;
- Patch #08 async queue wait, driver-call time, and sync fallback reasons.

Only after those measurements identify a specific blind spot should we add narrowly scoped instrumentation or change runtime policy.

## Validation boundary

This is a source inspection of the uploaded v1.0.121 tree. No source patch was applied, no Android NDK build or APK build was run, and no device runtime or thermal test was performed.
