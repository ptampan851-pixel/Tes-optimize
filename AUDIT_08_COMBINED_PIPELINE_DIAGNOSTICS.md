# AUDIT 08 — Combined pipeline diagnostics patch (v1.0.121)

## Why this combined patch exists

The separate routing and async diagnostic patch files were useful review artifacts, but their independently maintained hunk offsets and ordering were not a reliable single application path. This combined patch is regenerated directly from the unmodified v1.0.121 source and should be treated as the canonical patch for this diagnostic set.

## Canonical patch

`patches/gt30_combined_pipeline_diagnostics_1.0.121.diff`

It combines:
- Synchronous/async pipeline routing counters and fallback reason counters.
- Async job enqueue timestamp and queue wait measurement.
- Driver pipeline-creation duration measured only after acquiring `pipeline_cache_lock_`.
- Completed, successful, failed, and abandoned job counts.
- Periodic reporting through the existing `profile_frame()` report.

## Validation

- Regenerated from the uploaded `mhrise-recomp-1.0.121.zip` source.
- Applied to a fresh extraction with `patch --dry-run -p1`: passed.
- Applied to a second fresh extraction: passed.
- Confirmed the resulting source contains the async and routing report statements and the queue timestamp/counter fields.
- No Android NDK compilation, APK build, device runtime test, or thermal test has been performed.

## Usage

Apply this combined patch once to clean v1.0.121 source. Do not apply the older separate routing and async patch files on top of it; they describe the same changes and would duplicate/conflict.

Enable `perf_stats` to emit the profiling report. Add `profile_30` to shorten the report interval to 30 frames; `profile_30` alone does not enable reporting.

Compare identical scenes and routes across worker settings 1, 2, 3, and 4. Record queue wait, driver-call duration, success/failure/abandoned counts, synchronous routing reasons, frame pacing, and temperature. Keep the default worker count unchanged until device data supports a change.
