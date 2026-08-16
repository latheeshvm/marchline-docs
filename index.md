---
layout: default
---

# Marchline

**Deterministic, massively scalable pathfinding for Unity games with armies.**

Marchline is navigation middleware built for RTS, tower defense, colony
sims, and horde games — thousands of units finding their way, avoiding
each other, and arriving sensibly, under a few milliseconds a frame.

## Why Marchline

- **One-line API.** `agent.MoveTo(target)` — world scanning, planning,
  crowd avoidance, and arrival all happen behind it.
- **Armies are the default.** Group orders coalesce into shared flow
  fields automatically: 1,000 units to one point costs barely more than 50.
- **Bit-deterministic.** Fixed-point core, identical across Windows,
  macOS, and Linux, x64 and arm64 — lockstep multiplayer without desyncs,
  enforced by golden-hash tests in CI on every commit.
- **Live worlds.** Build a wall mid-wave and every unit reroutes; jammed
  chokepoints get priced and crowds swing to another gate on their own.
- **Honest failure.** No route means `Unreachable` — reported, explained
  in the inspector, and automatically retried when the world opens up.

## Sixty-second start

1. Install the package.
2. Empty GameObject → **Add Component → Marchline → Nav World**. It scans
   your obstacles on Play.
3. On any unit: **Add Component → Marchline → Marchline Agent**.
4. Move it:

```csharp
GetComponent<Marchline.MarchlineAgent>().MoveTo(target.position);
```

Whole army: `navWorld.MoveGroup(selectedUnits, clickPoint)` — add
`keepFormation: true` to keep the shape.

## Performance (measured, not projected)

On an Apple M2 Max, release builds, from the repository bench suite:
10,000 actively repathing agents ≈ 0.5 ms/tick of planning; a full
10,000-agent simulation tick including separation, anticipatory steering,
and hard collision ≈ 2 ms; a 256×256 flow-field build ≈ 1.5 ms; 50,000
cached-follow agents ≈ 1.4 ms. Budgets are counted in deterministic work
units, never wall-clock, so lockstep peers stay bit-identical.

## Documentation

- [Quickstart & recipes](quickstart.md) — a player and two enemies in
  five steps, then patterns for RTS squads, tower defense, hordes,
  colony sims, stealth AI, multi-storey worlds, and lockstep
- [Manual](manual.md) — setup, components, presets, groups & formations,
  dynamic worlds, debugging, performance guidance
- [Migrating from Unity NavMesh](migration-from-navmesh.md)
- [Migrating from A* Pathfinding Project](migration-from-app.md)
- [Determinism & lockstep guide](determinism.md)

## Status

Marchline is in closed beta ahead of its Unity Asset Store release.
Questions, beta interest, or studio licensing:
**support@marchline.dev**
