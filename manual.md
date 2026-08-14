---
layout: default
---

# Marchline manual

Deterministic, massively scalable pathfinding for games with armies.
This is the complete offline manual for the Unity package.

- [Migration from Unity NavMesh](migration-from-navmesh.md)
- [Migration from A* Pathfinding Project](migration-from-app.md)
- [Determinism & lockstep guide](determinism.md)

**Tested with:** Unity 6000.5 on macOS (arm64/x64). Targets Unity 2022.3+;
plugin binaries build for Windows, Linux, macOS, iOS, and Android in CI.
Consoles are available through direct licensing.

---

## 1. Sixty-second start

1. Empty GameObject → **Add Component → Marchline → Nav World**. Position
   it at your map's minimum corner; set `Grid Size` to cover the map.
2. On any unit: **Add Component → Marchline → Marchline Agent**.
3. Move it:

```csharp
GetComponent<Marchline.MarchlineAgent>().MoveTo(target.position);
```

Press Play. The NavWorld scans obstacles automatically; the agent plans
and walks. To see everything at once instead:
**GameObject → Marchline → Mini RTS Demo** → Play.

## 2. Concepts

**One world, many agents.** A single `NavWorld` owns the navigation state:
a cell grid scanned from your physics scene, a native deterministic
simulation, and every registered `MarchlineAgent`. Agents' transforms are
VIEWS of the simulation — Marchline moves them; you issue orders.

**Orders, not steering.** `MoveTo` / `MoveGroup` submit orders. The
planner amortizes work across ticks under a fixed budget: agents keep
following their previous plan until the new one is delivered, so orders
never freeze an army mid-march. Group orders to one point share ONE flow
field regardless of group size; scattered goals inside one 32×32 region
share one multi-source field; loners get individual hierarchical paths.
You never choose — classification is automatic.

**Crowds are solid.** Soft separation shapes crowds; anticipatory steering
bends streams around each other; a hard collision constraint guarantees
agents never interpenetrate. Crowds pack around a shared goal instead of
stacking ("group arrival"), and formation moves re-form the group's shape
at the destination with per-slot assignment.

**The world is live.** Geometry changes need no Scan button: add
`MarchlineObstacle` to anything that moves/spawns/despawns and its cells
refresh instantly; a rolling reconcile sweep converges everything else a
few cells per tick. Followers of outdated plans replan automatically.

## 3. NavWorld reference

| Field | Meaning |
|---|---|
| `preset` | Tunes planning budget + default agent speed. RTS 40k work-units/tick, speed 0.35 cells/tick · TowerDefense 20k / 0.35 · ColonySim 10k / 0.25 · Horde 60k / 0.45 |
| `cellSize` | World units per grid cell. Pick roughly your smallest unit's diameter. |
| `gridSize` | Grid dimensions in cells, starting at this transform's position (X/Z plane in 3D, X/Y in 2D). |
| `obstacleMask` | Physics layers that count as obstacles when scanning. |
| `scanMode` | `Physics3D` samples a band 0.1–0.9 × cellSize ABOVE the ground plane (the ground itself never blocks); `Physics2D` samples overlap boxes. |
| `ticksPerSecond` | Simulation rate (FixedUpdate-driven accumulator). 30 is right for most games. |
| `backgroundPlanning` | Default on: the native simulation runs on a worker thread; the main thread only enqueues orders and copies a snapshot. Turn OFF for lockstep games (see the determinism guide). |
| `reconcileCellsPerTick` | Rolling physics re-check, cells per tick (0 disables). Catches geometry changes not tracked by `MarchlineObstacle`. |

Key methods: `MoveTo(agentId, worldPos)`, `MoveGroup(agents, worldPos,
keepFormation)`, `RefreshRegion(bounds, margin)` (write-through re-scan of
an area), `RefreshAll()`, `AgentStateOf(id)`, `TryExplain(id, out ex)`
(synchronous mode only), `WorldToCell` / `CellToWorld`.

## 4. MarchlineAgent reference

| Member | Meaning |
|---|---|
| `initialDestination` | Optional transform; the agent walks there on Start. |
| `OverrideSpeed` | Cells per tick; 0 uses the preset default. |
| `MoveTo(worldPos)` | Order this agent. |
| `State` | `Idle`, `Planning` (order submitted, plan pending), `MovingOnField` / `MovingOnPath`, `Arrived`, `Unreachable`. |
| `Arrived` | Convenience for `State == AgentState.Arrived`. |

`Unreachable` is an answer, not a hang: the goal cannot be reached from
the agent's position. It re-resolves automatically when the world changes.

## 5. Dynamic worlds

- **Moving/spawning blockers:** add `MarchlineObstacle`. On any transform
  change it refreshes its old AND new footprints (physics is the source of
  truth, so overlapping obstacles resolve correctly). `margin` pads the
  refreshed area.
- **Everything else:** the reconcile sweep re-checks
  `reconcileCellsPerTick` cells per tick round-robin — untracked changes
  converge within `gridArea / rate` ticks without frame spikes.
- **Scripted edits:** `RefreshRegion(bounds)` after you change colliders,
  for immediate pickup.
- Followers whose plan predates a world change replan automatically while
  continuing on the old plan; sealed-off goals surface as `Unreachable` —
  and `Unreachable` units retry on their own the next time the world
  changes (demolish the wall and they resume their original order).

### Stuck units: the recovery ladder

A unit that stops making progress escalates automatically — you never
need to babysit it:

1. **Local avoidance** — anticipatory steering samples alternate headings
   around neighbors (always on).
2. **Push-through** — after ~1s of no movement, soft spacing relaxes so
   the unit can squeeze through gaps (hard collision always stays on).
3. **Congestion pricing + reroute** — units wedged mid-route for several
   seconds raise deterministic congestion costs on the jammed cells, then
   request a fresh plan built against those costs. The new flow field
   routes the *whole crowd* through a different gate or lane — replanning
   against unchanged costs would just return the same jammed path, so the
   cost map is what actually learns. Costs decay once the jam clears.
4. **Honest failure** — if no route exists at any price, the unit reports
   `Unreachable` and quietly retries when the world changes.

Escalation is rate-limited per unit with deterministically staggered
cooldowns, so thousands of jammed units never replan on the same tick.
All of it is integer math on fixed tick boundaries — lockstep-safe.

## 6. Group movement & formations

```csharp
navWorld.MoveGroup(selectedAgents, clickPoint);                    // crowd
navWorld.MoveGroup(selectedAgents, clickPoint, keepFormation: true); // formation
```

Crowd orders share one flow field and arrive PACKED around the point
(chain arrival — no stacking, no endless jostling at a full goal).
Formation orders preserve each agent's offset from the group centroid
(clamped to 8 cells, snapped to walkable): the group re-forms its shape,
and each agent homes onto its own slot for the final approach.

## 7. Performance guidance

- The planning budget (preset) is in deterministic WORK UNITS, not
  milliseconds — the same work happens on every machine.
- Reference numbers (Apple M2 Max, release, from the repo bench): 10k
  actively-repathing agents ≈ 0.5 ms/tick planning; full simulation tick
  with 10k agents incl. separation/steering/collision ≈ 1.8 ms; 256×256
  field build ≈ 1.5 ms; 50k cached-follow agents ≈ 1.4 ms.
- With `backgroundPlanning` on, all of that runs off the main thread.
- Memory: ~5 bytes/cell per cached flow field, 4 B/cell walkability,
  4 B/cell spatial hash.

## 8. Debugging & troubleshooting

Debugger v1, all in the editor (sync mode — with `backgroundPlanning`
off — because the main thread may only query the native world then):

- **Explain panel** — select any agent during Play: state, goal,
  cost-to-goal, path progress.
- **Flow-field overlay** — with an agent selected, the Scene view draws
  cyan arrows showing the field it follows: "why is it going THAT way?"
  at a glance.
- **Congestion heat** — select the NavWorld (`drawCongestion`): jammed
  lanes the planner is currently pricing glow orange, and fade as crowds
  clear.
- **Nav budget profiler** — the NavWorld inspector graphs planner work
  units spent per tick against the preset budget (deterministic work
  units, not milliseconds; a full bar means builds are being amortized —
  by design). "Congested cells" counts actively priced lanes.

| Symptom | Look at |
|---|---|
| Agent not moving | Its inspector during Play — the Explain panel shows state, goal, and (sync mode) what it's queued behind. `Planning` = order in flight; `Unreachable` = no route exists. |
| Everything unreachable | NavWorld gizmo (select it): red cells are blocked. Wrong `obstacleMask`? Ground on an obstacle layer in 2D mode? |
| Units ignore a new wall | Is the wall's collider on `obstacleMask`? Add `MarchlineObstacle` for instant pickup, or wait for the reconcile sweep. |
| Agents stop arriving at a character (chase/follow AI) | The target probably has a collider (CharacterController counts!) on a layer inside `obstacleMask` — the scan marks its own cell as a wall. Put characters on a layer excluded from the mask (the console warns about this at agent registration). See the Chase Demo for the correct setup. |
| "ABI … does not match" exception | The native plugin and C# scripts come from different package versions — reimport the package cleanly. |
| Console warning about a contained panic | `MarchlineWorld.TakePanicFlag()` returned true: an internal error was contained. Please report it — it is a Marchline bug, never fatal to your game. |

## 9. Advanced: direct world access

`NavWorld.World` exposes the managed wrapper (`MarchlineWorld`): raw
fixed-point conversion (`ToFixed`/`FromFixed`), bulk `ReadPositions`,
`Stats()`. In `backgroundPlanning` mode, do NOT call it from the main
thread except through NavWorld's snapshot-safe methods. For engine-less or
server-side use, the C ABI (`include/marchline.h` in the repository) is
the supported surface — see the determinism guide.
