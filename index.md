---
layout: default
---

# Marchline

**Deterministic pathfinding and crowd navigation for Unity, from one
character to thousands.**

Marchline moves the characters in your game: an enemy chasing the player,
guards on patrol, NPCs, companions, tower defense creeps, colonists and
whole armies. They find their way, avoid each other and arrive sensibly,
in a few milliseconds a frame.

[Get it on the Unity Asset Store](https://assetstore.unity.com/packages/tools/behavior-ai/marchline-rts-pathfinding-crowd-navigation-400596)

## Why Marchline

- **One-line API.** `agent.MoveTo(target)`. World scanning, planning,
  avoidance and arrival all happen behind it.
- **Good with one character.** Set a speed in world units per second and
  change it mid-route, `Stop`, `Warp`, and ask `CanReach` or
  `PathLengthTo` before you send anyone anywhere.
- **Good with thousands.** Group orders share flow fields automatically:
  1,000 units to one point costs barely more than 50.
- **Multi-storey.** Floors joined by stairs, escalators, doors and
  teleporters, with queues at the doors.
- **Bit-deterministic.** Fixed-point core, identical across Windows,
  macOS, Linux and WebAssembly: lockstep multiplayer and replays without
  desyncs, checked by golden-hash tests on every commit.
- **Live worlds.** Build a wall and every character reroutes; a jammed
  chokepoint gets priced and the crowd swings to another way through.
- **Honest failure.** No route means `Unreachable`: reported at once,
  explained in the inspector, and retried automatically when the world
  opens up.

## Sixty-second start

1. Install the package.
2. Empty GameObject → **Add Component → Marchline → Nav World**. It scans
   your obstacles on Play.
3. On any character: **Add Component → Marchline → Marchline Agent**.
4. Move it:

```csharp
GetComponent<Marchline.MarchlineAgent>().MoveTo(target.position);
```

A whole group: `navWorld.MoveGroup(selectedUnits, clickPoint)`. Add
`keepFormation: true` to keep the shape.

## Performance

Measured on an Apple M2 Max, release builds, from the repository bench
suite: 10,000 actively repathing agents ≈ 0.5 ms/tick of planning; a full
10,000-agent simulation tick including separation, anticipatory steering
and hard collision ≈ 1.8 ms; a 256×256 flow-field build ≈ 1.5 ms; 50,000
cached-follow agents ≈ 1.4 ms. Budgets are counted in deterministic work
units, never wall-clock, so lockstep peers stay bit-identical.

## Documentation

- [Quickstart & recipes](quickstart.md): a player and two enemies in five
  steps, then patterns for stealth AI, RTS squads, tower defense, hordes,
  colony sims, multi-storey worlds and lockstep
- [Manual](manual.md): setup, components, reachability, groups and
  formations, floors and links, dynamic worlds, debugging, performance
- [Changelog](changelog.md): what changed in each version
- [Migrating from Unity NavMesh](migration-from-navmesh.md)
- [Migrating from A* Pathfinding Project](migration-from-app.md)
- [Determinism & lockstep guide](determinism.md)

## Version

These pages describe Marchline 2.0. It works in world units and seconds;
1.x worked in cells and ticks. If your copy is 1.x, see
[Upgrading from 1.x](manual.md#upgrading-from-1x) and the
[changelog](changelog.md).

Questions or studio licensing: **support@marchline.dev**
