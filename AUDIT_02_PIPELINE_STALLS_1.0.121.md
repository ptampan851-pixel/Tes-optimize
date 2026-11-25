# Audit #02 — pipeline compile stalls on v1.0.121 (GT 30 / Mali-G615 MC2)

Source audited: user-uploaded `mhrise-recomp-1.0.121.zip`, not upstream web checkout.

## Report evidence
From uploaded `mhrise-report.zip` log:
- Pipeline cache read at startup: 5404 KB.
- Warm-up records: 1011 pipelines.
- Warm-up: 1011/1011 created in 0.3 s; 0 driver-compiled; 0 missing SPIR-V; 0 failed.
- Async pipeline workers: 2; `no_async_pipelines` is off; worker count override flags (1/3/4) are off.
- At one worst frame: 13.2 fps interval, worst frame 4631 ms; executor worst 4565.3 ms; synchronous pipelines in NvnSubmit 28 totaling 3751 ms; 4 shader translations totaling 4 ms; preloaded 3 totaling 17 ms.
- Slow-submit breakdown: driver pipeline creation 3751 ms (28); deswizzle 237 ms; new images 164 ms; shader translation 153 ms; mirror copy 32 ms; texture upload 30 ms; other smaller categories.
- Later samples at dynamic-resolution floor x0.733 still show GPU time 39.6, 36.2, and 34.9 ms.

## Code path findings
- `runtime/src/nnsdk/nvn/executor.cpp` contains async pipeline creation.
- Background pipeline workers start when an eligible pipeline first enters the async path; default 2, configurable 1/3/4.
- Async eligibility includes `async_pipelines_`, not per-stage keys, no skip geometry shader, recurring render pass, and recordable pipeline keys. Non-recordable/other paths can still create pipelines synchronously in the executor thread.
- A separate pipeline warm-up exists: reads recorded pipeline state, recreates them in a low-priority background thread, then the executor takes ready pipelines. The report shows all 1011 records warm successfully, with zero driver compilation at warmup.
- Driver pipeline cache is validated against vendor/device/cache UUID, and saved in background with pacing; notes describe reducing save frequency because writes previously caused severe SD stalls.

## Interpretation
This report's 3.751 s stall is specifically 28 pipeline creations on NvnSubmit, not shader translation or texture uploads. Since warm-up successfully covers 1011 recorded pipelines but the observed 28 synchronous builds still happen, likely the stall comes from pipelines not covered/eligible by warm-up (e.g. first-seen or non-recordable variants), but this is an inference pending instrumentation.

## Patch decision
Do not alter worker count, cache behavior, warm-up, or Vulkan synchronization blindly. Increasing workers may improve throughput but can increase CPU/GPU driver contention, thermal load, and concurrent shader compiler pressure on Mali. The next safer source change should be diagnostic instrumentation that records why pipeline creation falls back to synchronous NvnSubmit (eligible/recurring/recordable/per-stage/geometry-skip and warmup/cache path), then use the logs to target a narrowly scoped optimization. No performance patch has been applied in this audit.
