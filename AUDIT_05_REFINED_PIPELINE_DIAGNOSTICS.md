# Audit #05 — refined pipeline routing diagnostics (v1.0.121)

## Source re-opened

The user re-uploaded `mhrise-recomp-1.0.121.zip`. The exact file inspected was:
`runtime/src/nnsdk/nvn/executor.cpp`.

The refined patch is:
`patches/gt30_pipeline_routing_diagnostics_1.0.121.diff`

## Corrections made

1. `async queued` is now incremented only after insertion into `async_pending_` and `async_queue_`; it measures successful queue insertion rather than candidates.
2. Added `sync total` at entry to the synchronous pipeline-creation path. This counts synchronous path entries/requests, including requests whose driver creation may later fail; it is not a count of successful VkPipeline objects.
3. Added mutually exclusive top-level synchronous routing reasons: disabled, per-stage, skip-GS, non-recurring, retry, known-sync, and unrecordable.
4. Split unrecordable diagnostics into count-check, missing base cache key, and missing variant cache key. These subreason counters may overlap when more than one condition occurs on the same request.
5. Diagnostic counters use relaxed atomics; report emission uses atomic exchange to reset each reporting interval.

## Validation

- Re-generated the unified diff against the exact v1.0.121 source.
- Ran `patch --dry-run -p1` against the extracted source tree; all hunks applied cleanly.
- No Android build or runtime test has been performed. Do not treat this as a compiled or validated APK.

## Interpretation caution

The counters measure routing decisions. `sync total` means entries into the synchronous creation path, not successful driver pipeline creation. The reason counters are designed to account for those path entries; unrecordable subreason counters are diagnostic subsets and can overlap.
