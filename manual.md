---
layout: default
---

# Marchline manual

Deterministic pathfinding and crowd navigation for Unity, from one
character to thousands. This is the complete manual for the Unity package,
version 2.0.

New here? Start with [Quickstart & recipes](quickstart.md) — a player
and two enemies in five steps, then a pattern per genre.

- [Migration from Unity NavMesh](migration-from-navmesh.md)
- [Migration from A* Pathfinding Project](migration-from-app.md)
- [Determinism & lockstep guide](determinism.md)
- [Upgrading from 1.x](#upgrading-from-1x) (2.0 changes units)

**Units.** Everything the components take or return is in world units and
seconds: speeds in units per second, radii, margins, widths and path
lengths in units, link durations in seconds. Cell size and tick rate are
tuning, not units. Only `MarchlineWorld`, the raw layer under `NavWorld`,
speaks cells and ticks.

**Tested with:** Unity 6000.3 LTS and 6000.6 on macOS (arm64/x64). The
package ships native plugins for Windows, Linux, macOS, iOS, Android and
WebGL. Consoles are available through direct licensing.

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
| `preset` | Tunes planning budget + default agent speed (`DefaultSpeed`, world units per second). RTS 40k work-units/tick, speed 10.5 · TowerDefense 20k / 10.5 · ColonySim 10k / 7.5 · Horde 60k / 13.5. The speeds suit unit-sized cells seen from above; give characters their own `speed` (a walk is about 1.5, a run 4 to 6). |
| `cellSize` | World units per grid cell. Pick roughly your smallest unit's diameter. |
| `gridSize` | Grid dimensions in cells, starting at this transform's position (X/Z plane in 3D, X/Y in 2D). |
| `obstacleMask` | Physics layers that count as obstacles when scanning. |
| `scanMode` | `Physics3D` samples a band 0.1–0.9 × cellSize ABOVE the ground plane (the ground itself never blocks); `Physics2D` samples overlap boxes. |
| `ticksPerSecond` | Simulation rate (FixedUpdate-driven accumulator). 30 is right for most games. |
| `backgroundPlanning` | Default on: the native simulation runs on a worker thread; the main thread only enqueues orders and copies a snapshot. Turn OFF for lockstep games (see the determinism guide). |
| `reconcileCellsPerTick` | Rolling physics re-check, cells per tick (0 disables). Catches geometry changes not tracked by `MarchlineObstacle`. |

Key methods: `MoveTo(agentId, worldPos)`, `MoveGroup(agents, worldPos,
keepFormation)`, `IsReachable(from, to)`, `SnapToReachable(from,
target, maxRadius)`, `TryFindPath(from, to, points, out length)` and
`PathLength(from, to)` (see "Ask before you walk" below),
`SetSpeed(agentId, speed)`, `Warp(agentId, worldPos)` and `Stop(agentId)`
(agent control by handle, for bulk agents), `ToCellsPerTick` /
`ToUnitsPerSecond`,
`RefreshRegion(bounds, margin)` (write-through re-scan of an area;
`margin` is in world units), `RefreshRegionDeferred(bounds, margin)` (the
same at the next simulation step), `RefreshAll()`, `AgentStateOf(id)`,
`TryExplain(id, out ex)` (synchronous mode only), `WorldToCell` /
`CellToWorld`.

## 4. MarchlineAgent reference

| Member | Meaning |
|---|---|
| `initialDestination` | Optional transform; the agent walks there on Start. |
| `speed` | World units per second; 0 uses the preset default. Change it any time, in the inspector or from code (`agent.speed = 4.5f`): the agent keeps its order and carries on at the new pace, also inside a formation. |
| `MoveTo(worldPos)` | Order this agent. Safe the frame it is created: the order is issued when it registers. |
| `MoveTo(worldPos, snapToWalkable)` | The same, with the goal moved to the nearest walkable cell when its own is blocked. |
| `CanReach(worldPos)` | Whether a `MoveTo` there could succeed from where the agent stands, without ordering it. |
| `PathLengthTo(worldPos)` | How far it would walk to get there, in world units; -1 when it cannot. Costs a search: for a decision now and then. |
| `RemainingDistance()` | Walking distance left on the current order (0 at rest, -1 when the goal cannot be reached). Costs a search too. |
| `Stop()` | Drop the current order: the agent stands where it is, `Idle`. Inside a link it finishes the crossing first. |
| `Warp(worldPos)` | Move instantly. The agent arrives `Idle`: give it a new order. A crossing in progress is abandoned. |
| `NativeId` | Id in the native world. It never changes while the agent lives: changing speed, warping and stopping happen in place. |
| `CurrentLink`, `LinkProgress` | The link the agent is crossing (or null) and how far through it is, 0 to 1. |
| `State` | `Idle`, `Planning` (order submitted, plan pending), `MovingOnField` / `MovingOnPath`, `Arrived`, `Unreachable`. |
| `Arrived` | Convenience for `State == AgentState.Arrived`. |

`Unreachable` is an answer, not a hang: the goal cannot be reached from
the agent's position. It arrives on the next simulation tick (the core
knows which cells connect; nothing is searched) and re-resolves
automatically when the world changes.

## 5. Dynamic worlds

- **Moving/spawning blockers:** add `MarchlineObstacle`. When it moves,
  resizes, appears or disappears (destroyed, deactivated, or a collider
  switched off) it refreshes its old AND new footprints (physics is the
  source of truth, so overlapping obstacles resolve correctly). The
  footprint is the combined bounds of the solid colliders on the object
  and its children; call `Rescan()` after adding a collider at runtime.
  `margin` pads the refreshed area, in world units. A vanished obstacle
  frees its cells at the next simulation step, not the same frame.
- **Everything else:** the reconcile sweep re-checks
  `reconcileCellsPerTick` cells per tick round-robin — untracked changes
  converge within `gridArea / rate` ticks without frame spikes.
- **Scripted edits:** `RefreshRegion(bounds)` after you change colliders,
  for immediate pickup. When you remove geometry from inside its own
  `OnDisable` / `OnDestroy`, use `RefreshRegionDeferred(bounds)`: the
  departing collider still answers physics queries there, so an immediate
  refresh would find it and leave the cells blocked.
- Followers whose plan predates a world change replan automatically while
  continuing on the old plan; sealed-off goals surface as `Unreachable` —
  and `Unreachable` units retry on their own the next time the world
  changes (demolish the wall and they resume their original order).

### Ask before you walk: reachability

`Unreachable` arrives a tick after the order. To know beforehand, and to
compare routes without ordering anybody:

```csharp
if (agent.CanReach(doorway.position)) agent.MoveTo(doorway.position);

bool open = NavWorld.Instance.IsReachable(spawn.position, exit.position);

// The nearest point to the target that he can actually get to:
Vector3 spot = NavWorld.Instance.SnapToReachable(agent.transform.position, target);

// Which doorway is nearer on foot (not as the crow flies)?
float a = agent.PathLengthTo(doorA.position);   // -1: cannot get there
float b = agent.PathLengthTo(doorB.position);

// The route itself: corner points in world space.
var points = new List<Vector3>();
if (NavWorld.Instance.TryFindPath(from, to, points, out float length)) { /* draw it */ }
```

- The answer is what the planner would find: the same grid (live scan for
  the ground, baked floors upstairs) and the enabled `MarchlineLink`s,
  one-way links in their direction only. Heights pick the floors, as in
  `MoveTo`. It works in both update modes.
- `to` must be a walkable cell, or the answer is false. `SnapToReachable`
  handles a target inside a wall for you; `SnapToWalkable` only finds the
  nearest open cell, which may be on the far side of that wall. A point
  outside the grid counts as the nearest cell on its edge, as in `MoveTo`.
- `SnapToReachable` looks within `maxRadius` world units (4 by default)
  and returns the target unchanged when nothing reachable is that close;
  `TrySnapToReachable` tells the two cases apart.
- `IsReachable` answers "is there a route" and ignores crowding and link
  queues. Cost: after the ground changes, the next query relabels the
  grid once (about 1 ms per 100,000 cells); queries between changes are
  lookups.
- `TryFindPath` and `PathLength` answer "which way, how far": the route
  the planner would take (cell centres at their floor's height, a crossed
  link's entrance followed by its exit; points in the middle of straight
  runs are left out) and its length in world units. They cost a search
  over the cells in between: use them for a decision now and then, not
  per agent per frame. In background mode the call waits for the
  simulation thread to finish the tick it is on, so the answer includes
  this frame's edits.
- A ground-level goal that can only be reached over links (up one stair,
  along a deck, down another) is routed over them; before 2.0 the planner
  treated ground goals as flat and reported `Unreachable`.

### Stuck units: the recovery ladder

A unit that stops making progress escalates automatically — you never
need to babysit it:

0. **Step out of walls** — a unit standing inside a cell that just became
   blocked (an obstacle rose under it, or it spawned in one) plans from
   the nearest walkable cell and walks out into it, up to 6 cells deep.
   Deeper than that it stays put and reports `Unreachable`.
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

Chase AI that re-issues `MoveTo` at a moving target every few frames
works with the ladder, not against it. "Wedged" is judged two ways: by
plan cost (no progress along the current plan) and by position (the unit
has not left a 3-cell box for as long as walking 12 cells would take it).
The positional clock does not restart when a re-aim delivers a fresh
plan, so a pursuer wedged behind other agents still reaches step 3 while
its target keeps moving; and a re-aim within 4 cells of the goal during a
reroute's cooldown keeps the congestion-priced route instead of walking
back into the jam.

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

## 6b. Vertical worlds: floors, stairs, and doors (1.1)

Multi-storey navigation uses LAYERS (independent walkable floors) joined
by LINKS (stairs, jumps, doors, teleporters). The core owns traversal:
capacity limits, deterministic FIFO queues at full links, and crossing
time all run inside the simulation, so multi-floor crowds stay
bit-deterministic and replayable.

Setup:

1. **Bake the upper floors** into a `LayeredGridAsset` (Create >
   Marchline > Layered Grid), via `LayeredGridBaker.Bake(navWorld,
   new[]{ 3f, 6f }, asset)` with the world-space heights of each floor.
   The baker raycasts every column and sorts hits deterministically, so
   identical scenes bake identical bytes (`topologyHash` lets lockstep
   peers verify that before a match). Assign the asset to the NavWorld.
   Upper floors are static after load — doors and links are the dynamic
   vocabulary. (3D scan mode; the base floor stays live-scanned.)
2. **Author links** with the `MarchlineLink` component: the component's
   transform is the entrance, `target` is the exit; endpoints resolve to
   floors by height. Kind, crossing `duration` (seconds), capacity (0 =
   unlimited; waiters queue FIFO), one-way or bidirectional.
   `costDistance` is how much walking the planner counts the crossing as,
   in world units; 0 = automatic (the distance between its ends), always
   heuristic-safe.
3. **Order normally.** `agent.MoveTo(pointOnUpperFloor)` — the height
   picks the floor, and one shared cross-layer flow field serves any
   number of agents heading there. Agents report
   `AgentState.Traversing` while inside a link and render at their
   floor's elevation. Goal snapping (`MoveTo(pos, snapToWalkable:
   true)`) is floor-aware: it searches the floor the height selects and
   keeps that height, so a deck target above a blocked ground cell stays
   a deck order; a target standing on a stairs link (mid-ramp, or on
   its edge) snaps to the link's nearer end.

**Supported scale (v1, honest):** layered planning is synchronous —
every DISTINCT cross-layer destination builds a full multi-floor flow
field the tick it's needed. Crowds sharing a handful of destinations
are the designed case; tens of unique simultaneous cross-floor goals
will spike a tick (measured: 20 unique goals on a small 4-floor map ≈
100 ms). Keep concurrent layered destinations small, or stagger
orders; budgeted layered planning is roadmap work.

Crossings are drawn moving: while an agent is `Traversing`, its transform
slides from where it entered to the link's exit over the link's `duration`
(`NavWorld.smoothLinkCrossings`, on by default; teleporters still jump).
This is presentation only: the simulation holds the agent at the entrance
until the crossing completes, so determinism is untouched. Read
`agent.CurrentLink` and `agent.LinkProgress` (0 to 1) to play a climb or
door animation; both come from the simulation (which link, how many ticks
left), so they are right however close two links stand. Set `duration` to
how long the crossing should take: a 13 m escalator at walking pace is
about 9 seconds.
When the crossing is not a straight line (an escalator's flat landings, a
stair's half-landing), add its corners to the link's `waypoints` in order
from the entrance: crossings are drawn along that path at constant speed,
and goal snapping finds targets standing anywhere on it.

Doors: toggle `link.LinkEnabled` at runtime. Every field rebuilds
deterministically; queued agents re-route or turn `Unreachable`, an
agent already inside finishes its crossing, and reopening the door
un-strands everyone without fresh orders.

## 6c. Massive crowds: LOD tiers + the bulk adapter (1.3)

Two pieces make 50k honest:

1. **Deterministic crowd LOD** (core, opt-in): agents ride tiers — T0
   full dynamics, T1 cached-VO, T2 flow + occupancy (hard collision
   only), T3 dormant. Tiering reads only synchronized simulation state
   (local density, stuck/queue signals — never a camera, which lockstep
   peers don't share), with integer thresholds, hysteresis, dwell, and
   staggered re-evaluation. Hard collision stays on for **every** tier.
   The **Horde preset enables the 50k profile automatically**; the
   switch is simulation-affecting and saved, so all lockstep peers must
   share it.
2. **`MarchlineCrowd`** — thousands of agents with ZERO per-agent
   GameObjects: native spawns, slot-indexed snapshot reads, and
   `Graphics.RenderMeshInstanced` drawing. `crowd.MoveAll(point)` is
   one native call and one shared flow field.

Reference numbers (M2 Max, single core, full simulation, four
converging 12.5k streams): 50k agents ≈ 18 ms/tick average, P99 ≈ 21 ms
— with `backgroundPlanning` on, all of it off the main thread.

## 6d. WebGL (1.4)

WebGL is supported with the same determinism guarantee: the wasm plugin
is built from the identical fixed-point core, linked by Unity's OWN
bundled Emscripten, and the build script refuses to ship unless a
trajectory replayed in wasm is bit-identical to native. CI additionally
re-proves every golden constant under WebAssembly. Notes:

- The simulation runs synchronously on WebGL (browser pthreads are
  COOP/COEP-gated; there is no worker thread). Budget accordingly —
  planning work shares the main thread.
- Replays and saves are portable: a replay recorded in a browser
  scrubs bit-exactly on desktop, and vice versa.
- Rebuild the plugin per Unity version with
  `scripts/build-web-plugin.sh` (it auto-detects the newest installed
  editor's bundled toolchain).

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
| Agent ordered somewhere it can never get | Ask first: `agent.CanReach(pos)`, or pick the goal with `NavWorld.SnapToReachable(from, pos)`. A link end in a blocked cell logs a warning: that link carries nobody. |
| Units ignore a new wall | Is the wall's collider on `obstacleMask`? Add `MarchlineObstacle` for instant pickup, or wait for the reconcile sweep. |
| Agents stop arriving at a character (chase/follow AI) | The target probably has a collider (CharacterController counts!) on a layer inside `obstacleMask` — the scan marks its own cell as a wall. Put characters on a layer excluded from the mask (the console warns about this at agent registration). See the Chase Demo for the correct setup. |
| "ABI … does not match" exception | The native plugin and C# scripts come from different package versions — reimport the package cleanly. |
| Console warning about a contained panic | `MarchlineWorld.TakePanicFlag()` returned true: an internal error was contained. Please report it — it is a Marchline bug, never fatal to your game. |

## 8b. Time machine: record, replay, scrub, save (1.2)

The simulation is a pure function of its command stream, and 1.2 turns
that into tools (sync mode — `backgroundPlanning` off):

- **Record a session**: enable `recordSession` on the NavWorld (or call
  `NavWorld.Instance.World.StartRecording()` any time — mid-game works;
  the replay embeds the current state). `TakeSessionRecording()` returns
  the replay bytes: every order, edit, and door toggle, plus periodic
  drift probes and scrub checkpoints. Replays are bit-exact on every
  platform — a player's replay file reproduces their exact game on your
  machine.
- **Scrub it**: Window > Marchline > Replay Scrubber — load a
  `.mlreplay` file and drag through time; agents draw top-down, colored
  by state. Embedded checkpoints make seeking fast; a corrupted
  checkpoint silently falls back to a full replay (slower, never
  wrong), and a replay from a mismatched build is *detected* at the
  first drift probe, not silently divergent.
- **Save/load worlds**: `SaveWorldBytes()` / `MarchlineWorld.LoadBytes`
  serialize the COMPLETE world — in-flight planning, link queues,
  agents mid-crossing — and the loaded world continues bit-identically
  with the original. Save-game grade, deterministic bytes.

## 9. Advanced: direct world access

`NavWorld.World` exposes the managed wrapper (`MarchlineWorld`): raw
fixed-point conversion (`ToFixed`/`FromFixed`), bulk `ReadPositions`,
`Stats()`. In `backgroundPlanning` mode, do NOT call it from the main
thread except through NavWorld's snapshot-safe methods. For engine-less or
server-side use, the C ABI (`include/marchline.h` in the repository) is
the supported surface — see the determinism guide.

## Upgrading from 1.x

2.0 moves agent control and reachability into the native core (ABI 206)
and changes the public units to world units and seconds. Replace the
whole package, plugins included: the 2.0 scripts refuse to run on 1.x
plugins. Replays recorded by 1.x do not open in 2.0 (the replay format
changed); world saves from 1.x load and continue under the 2.0 rules.

Scenes and prefabs convert themselves: a saved `OverrideSpeed`, link
`durationTicks` / `cost` and crowd `speed` are carried over to the new
fields the first time the object meets a `NavWorld` (at `Start`, or in
the inspector), using that world's cell size and tick rate. Code keeps
compiling, with a warning on each old name:

| 1.x | 2.0 | Old name still works? |
|---|---|---|
| `agent.OverrideSpeed` (cells per tick) | `agent.speed` (units per second) | Yes, obsolete, old meaning |
| `agent.SetSpeed(cellsPerTick)` | `agent.speed = unitsPerSecond` | Yes, obsolete, old meaning |
| `link.durationTicks` | `link.duration` (seconds) | Yes, obsolete, old meaning |
| `link.cost` (10 = one cell) | `link.costDistance` (world units) | Yes, obsolete, old meaning |
| `crowd.speed` (cells per tick) | `crowd.agentSpeed` (units per second) | No: rename it |
| `nav.DefaultAgentSpeed` (cells per tick) | `nav.DefaultSpeed` (units per second) | Yes, obsolete, old meaning |
| `nav.AddBulkAgent(pos, cellsPerTick)` | `nav.AddBulkAgent(pos)` then `nav.SetSpeed(handle, unitsPerSecond)` | Yes, obsolete, old meaning |

To convert by hand: units per second = cells per tick × `cellSize` ×
`ticksPerSecond`; seconds = ticks ÷ `ticksPerSecond`.

Behaviour that changed:

- **Preset default speeds are fixed in world units** (10.5 / 7.5 / 13.5
  per second). They used to be cells per tick, so they scaled with
  `cellSize` and `ticksPerSecond`. With 1 m cells at 30 ticks per second
  nothing changes; on any other grid, agents without their own speed now
  move at the same world speed as on that one.
- `NativeId` no longer changes on a speed change or a warp. Code that
  re-read it after every `SetSpeed` still works; code can now cache it.
- A speed change applies at once, also to an agent at rest or in a
  formation (it keeps its slot). Inside a link the crossing keeps its
  duration and the new speed applies from the exit.
- An order to a place that cannot be reached reports `Unreachable` on the
  next tick, and a ground goal reachable only over links is now reached.
- `NavMeshAgentCompat.isStopped` and `ResetPath` stop the agent where it
  stands (they used to order it to its own cell).
- The plugin check is stricter: the native plugin must be at least the
  ABI the scripts were written for.
