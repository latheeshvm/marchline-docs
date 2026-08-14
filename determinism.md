---
layout: default
---

# Determinism & lockstep guide

Marchline's core simulation is **bit-deterministic across platforms**:
all math is Q32.32 fixed point (floats are lint-banned from the core), all
iteration orders are explicit and total, planning budgets are counted in
work units (never wall-clock time), and internal parallelism uses
order-fixed reduction. Cross-platform bit-exactness is enforced in CI by
golden trajectory hashes on Linux x64, Windows x64, macOS arm64 and
macOS x64, in debug and release.

**The guarantee's boundary is the ABI.** Identical sequences of ABI calls
with identical arguments produce identical simulations everywhere. What
you feed it is your responsibility — and that's where the two modes come
in.

## Mode 1: background planning (default)

`NavWorld.backgroundPlanning = true`. All native work runs on a worker
thread; the main thread enqueues TICK-STAMPED commands and renders from a
snapshot. The simulation is still a pure function of the command sequence
— thread scheduling only affects how fresh the rendered snapshot is — but
the command sequence itself derives from Unity-side float math (transform
positions, `Time.fixedDeltaTime` accumulation, physics scans), which
Unity does not guarantee to be identical across machines.

**Use for:** single-player, co-op with server authority, anything that
does not replay or lockstep-compare simulations across machines.

## Mode 2: synchronous (lockstep)

`NavWorld.backgroundPlanning = false`, or drive the C ABI directly from
your own fixed game loop. For true lockstep:

1. Feed identical inputs on every peer: build the walkability grid from
   shared data (not per-machine physics scans), issue orders as raw cell
   coordinates from your synchronized command stream, and tick with the
   same budget on the same frame.
2. Prefer the raw fixed-point entry points (`ml_agent_add` takes Q32.32
   `long`s: `raw = (long)(value * 4294967296.0)` computed from
   integer/synchronized data, not from local float transforms).
3. The same ticks with the same inputs are then bit-identical on every
   peer and platform — desync-free by construction, verified by the same
   golden-hash machinery that gates our CI.

## Server-side / headless

The native library has no engine dependencies. Link `marchline.h` +
the platform library from any C-compatible host (a dedicated server, a
replay verifier, a bot harness) and run the exact simulation your clients
run. `ml_abi_version()` must match the header you compiled against; every
call is panic-contained (`ml_take_panic_flag`).

## What can break determinism (checklist)

- Feeding positions/goals derived from local float math into a lockstep
  sim (quantize to cells or fixed point first)
- Different budgets or tick counts per peer (budget is part of the input)
- Building the grid from per-machine physics scans in lockstep mode
- Mixing plugin/package versions across peers (the C# wrapper refuses to
  start on an ABI mismatch; native hosts should check `ml_abi_version`)
