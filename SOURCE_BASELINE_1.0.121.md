# GT30 Optimization project — source baseline v1.0.121

## Baseline
Use the user-provided `mhrise-recomp-1.0.121.zip` as the new source baseline. The previous patch `gt30_patch01_dynamic_resolution.diff` was written for v1.0.85 and is obsolete.

## Dynamic resolution findings (verified in v1.0.121)
File: `runtime/src/nnsdk/nvn/window.cpp`

The rate array already includes:
`1.0f, 0.93334f, 0.86667f, 0.80001f, 0.73334f, 0.66668f, 0.60001f, 0.53335f, 0.50001f`

- Standard dynamic resolution uses the first five levels (down to ~73%).
- Wide dynamic resolution (`dynres_wide`) enables all nine levels down to 50%.
- Wide mode starts at the lowest level after load and climbs when GPU headroom allows.
- Settings UI already exposes `dynres_wide` and `no_dynamic_resolution`.

README documents this at the Dynamic resolution settings section; `docs/settings.md` repeats it.

## Decision
Do not reapply the old patch that appended 0.66668: it is already present. Next work should audit actual GPU pipeline stalls, shader cache and compile behavior in v1.0.121 before proposing a new code change.

## Important repository status
This repository currently contains project notes and patch artifacts only; the full upstream source tree has not yet been imported.
