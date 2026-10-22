# Decisions & Findings

Log non-obvious choices and anything learned the hard way. Newest first.

## Decisions
- Remotion Player is the single clock; Cavalry is driven per frame via setFrame/render. (Initial architecture)
- Critical test (Phase 2) runs on a hard-coded timeline before any UI gets built.

## Cavalry findings
_(load times, WASM size, determinism, context limits, quirks)_
