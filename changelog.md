---
layout: default
---

# Changelog

All notable changes to this package are recorded here. The format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [2.0.1] - 2026-10-05

Wrapper-only; plugins unchanged (ABI 206).

### Fixed

- `IsReachable`, `MarchlineAgent.CanReach` and `TrySnapToReachable`
  from a start inside something now give the planner's answer. The core
  steps an agent out of a blocked cell to the first free cell up to six
  cells away and plans from there; the wrapper only looked a cell and a
  half, so an agent a prop appeared on (or a point deep in a counter)
  was "unreachable" while `TryFindPath`, `PathLength` and `MoveTo` found
  the route. The wrapper now runs the core's own search.
- `IsReachable` and `CanReach` with a point outside the grid said no,
  while `MoveTo` and `TryFindPath` take such a point as the nearest cell
  on the grid's edge and go there. They now do the same, for the start
  and for the target.

## [2.0.0] - 2026-10-05

Ask before you walk, in real units. Agent control and reachability move
into the native core (ABI 206), and the public API changes from cells and
ticks to world units and seconds. Replace the whole package, plugins
included. Replays recorded by 1.x do not open (replay format 6); saves
from 1.x load and continue under the 2.0 rules. See "Upgrading from 1.x"
in the manual.

### Added

- `MarchlineAgent.speed`, in world units per second (0 = the preset
  default). Assign it any time, in the inspector or from code: the agent
  keeps its order, its place in a formation and its native id, and
  carries on at the new pace.
- `MarchlineAgent.Stop()`: drop the order and stand, `Idle`.
- `NavWorld.TryFindPath(from, to, points, out length)` and
  `NavWorld.PathLength(from, to)`: the route the planner would take and
  its length in world units, without ordering anybody.
  `MarchlineAgent.PathLengthTo(position)` and `RemainingDistance()` ask
  for one agent. In both update modes.
- Agent control by handle for bulk agents: `NavWorld.SetSpeed(handle,
  speed)`, `Warp(handle, position)`, `Stop(handle)`.
- `NavMeshAgentCompat.speed`.
- Core and C ABI (v206): `ml_agent_set_speed`, `ml_agent_stop`,
  `ml_agent_warp`, `ml_agent_link`, `ml_world_agent_links`,
  `ml_world_reachable`, `ml_world_nearest_reachable`,
  `ml_world_find_path`; the matching `MarchlineWorld` methods. The core
  keeps connectivity labels per floor, updated a sector at a time as cells
  change, and a graph of the links between them.

### Changed

- **Units.** Speeds are world units per second, link durations seconds,
  link costs world units. `OverrideSpeed`, `SetSpeed(cellsPerTick)`,
  `durationTicks`, `cost`, `DefaultAgentSpeed` and
  `AddBulkAgent(position, cellsPerTick)` still compile, marked obsolete,
  with their old meaning; `MarchlineCrowd.speed` is now `agentSpeed`.
  Values saved in scenes and prefabs convert themselves.
- **Preset default speeds are fixed in world units** (RTS and TowerDefense
  10.5 per second, ColonySim 7.5, Horde 13.5). They were cells per tick
  and so scaled with `cellSize` and `ticksPerSecond`; with 1 m cells at 30
  ticks per second nothing changes.
- An order to a goal that cannot be reached reports `Unreachable` on the
  next tick, from the connectivity labels, with no planning work (the
  planner used to search the whole reachable area first). It still retries
  by itself when the world changes.
- A ground-level goal that can only be reached over links (up, along a
  deck, down) is routed over them. It used to report `Unreachable` while
  `IsReachable` said yes.
- Changing speed and warping no longer re-create the native agent:
  `NativeId` is stable for the agent's life, a speed change applies at
  once (also at rest), and an agent in a formation keeps its slot.
- Link crossings are drawn from what the simulation reports (which link,
  ticks left) instead of a guess by nearest entrance: `CurrentLink` and
  `LinkProgress` are right for links that share a foot or stand in
  neighbouring cells, and the drawn crossing ends exactly when the agent
  lands.
- `NavMeshAgentCompat.isStopped` and `ResetPath` stop the agent where it
  stands instead of ordering it to its own cell.
- The wrapper requires a plugin of at least its own ABI (it accepted any
  plugin of the same major version).
- Replay format 6 (three new commands: set speed, warp, stop).

### Removed

- `MarchlineAgent.SetSpeed`'s re-create behaviour and the notes that came
  with it (the id changing, a resting agent taking the speed on its next
  order).

## [1.1.0] - 2026-10-04

An audit of 1.0.2 to 1.0.5 and of the obstacle component. Wrapper-only:
native plugins and ABI (205) unchanged, so simulation results, saves and
replays are identical.

### Added

- Reachability. `NavWorld.IsReachable(from, to)` and
  `MarchlineAgent.CanReach(position)` answer "could an agent walk there?"
  at once, instead of ordering the move and waiting for `Unreachable`.
  `NavWorld.SnapToReachable(from, target, maxRadius)` (and
  `TrySnapToReachable`) return the nearest point to a target that an agent
  at `from` can actually get to. Judged on the planner's own grid and the
  enabled `MarchlineLink`s (one-way links in their direction), in both
  update modes. After the ground changes, the next query relabels it (one
  pass over the grid, about 1 ms per 100,000 cells); queries between
  changes are lookups.
- `NavWorld.RefreshRegionDeferred(bounds, margin)`: a refresh at the next
  simulation step, for geometry removed from inside `OnDisable` /
  `OnDestroy`.
- `MarchlineObstacle.Rescan()`: pick up colliders added after the
  component was enabled.
- A warning when a `MarchlineLink` end lands in a blocked cell (the link
  then carries nobody, and used to say nothing).

### Changed

- **`RefreshRegion`'s `margin` is in world units**, as its documentation
  always said; it was applied as cells. `MarchlineObstacle.margin` is
  world units too (its tooltip said cells). With `cellSize` 1 nothing
  changes; on a finer grid the same number now pads further. Only the
  number of cells re-sampled changes, never the result.
- `MarchlineAgent.SetSpeed` on an agent at rest (`Arrived` or `Idle`) no
  longer re-creates it at once: it stays as it is and takes the new speed
  with its next order. `Arrived` used to flip to `Idle`.
- `MarchlineAgent.State` in background mode no longer reports the previous
  order's resting state after a new order: until the snapshot shows the
  order, `Arrived` / `Idle` / `Unreachable` read as `Planning`, as the
  simulation itself reports.
- `initialDestination` is used only when the agent was given no order
  before `Start`.
- `MarchlineAgent` no longer has a `LateUpdate` (a pending speed change is
  applied from NavWorld's own agent loop).
- `NavMeshAgentCompat.Warp` returns `bool`, like `NavMeshAgent.Warp`.

### Fixed

- `MarchlineObstacle`: a destroyed obstacle left its cells blocked until
  the reconcile sweep came round (never, with the sweep off), and a
  deactivated one cleared only when its collider sat above the component.
  Inside `OnDisable` the departing collider still answers physics queries;
  the old footprint is now refreshed at the next step.
- `MarchlineObstacle` follows the colliders on its children (the
  footprint was a guess from the transform), and a collider being switched
  off or on.
- `SetSpeed` stopped an agent whose order came through
  `NavWorld.MoveGroup`, `NavWorld.MoveTo(id, …)` or `MoveHandles`: only
  orders given through `MarchlineAgent.MoveTo` were re-issued. Every order
  is remembered now. (An agent in a formation move carries on to the
  group's destination, out of its slot.)
- `Warp` (and `SetSpeed`) in background mode: the agent was drawn back at
  its old place, with its old state, for one physics step.
- `agentRadius` under 45% of a cell shrank the scan probe, so a thin wall
  on a cell boundary fell between two probes and vanished (1 m cells,
  radius 0.2: a 0.3 m wall was invisible). The probe is never narrower
  than the classic 45%.
- `RefreshRegion` with a wide `agentRadius` in Physics2D mode widened X
  and Z instead of the grid's plane.
- `MarchlineAgent.MoveTo` called before the agent's `Start` (the frame it
  was created) was silently dropped; it is issued on registration.
- `NavMeshAgentCompat.Warp` only set the transform, which the next
  simulation sync overwrote.
- `LinkProgress` stuck at 1 after the first crossing of a link that has no
  `MarchlineLink` component.

## [1.0.5] - 2026-10-04

Wrapper-only: native plugins and ABI (205) unchanged.

### Added

- `NavWorld.agentRadius` and `NavWorld.agentHeight`: give the obstacle
  scan a body. A cell is walkable only when nothing stands within
  `agentRadius` of its centre, up to `agentHeight` above the floor, so a
  guard 0.6 m wide is no longer routed through 0.5 m gaps between props
  or under table tops. 0 (the default) keeps the classic rule (45% of a
  cell either side, up to 90% of a cell), so existing scenes are
  unchanged. The layered baker uses the same box; rebake after changing
  either.
- `NavWorld.ProbeBox(...)`: the box the scan and the baker test, for
  tools that want to match them.

### Changed

- `RefreshRegion` also re-checks the cells whose probe reaches into the
  region when `agentRadius` is wider than half a cell, so a moved
  `MarchlineObstacle` updates every cell it affects.

### Fixed

- `IsWalkable` on the ground no longer throws in the editor after the
  grid was resized: a scan mirror left from the old size (domain reloads
  keep it) is ignored and physics is sampled instead.

## [1.0.4] - 2026-10-02

Wrapper-only: native plugins and ABI (205) unchanged.

### Added

- `MarchlineAgent.SetSpeed(cellsPerTick)`: change an agent's speed while
  playing (a guard breaking from a patrol walk into a run). The core fixes
  speed when an agent is created, so the agent is re-created where it
  stands and its current order re-issued; a crossing in progress finishes
  first. Before this, `OverrideSpeed` was read once at registration and
  later changes were silently ignored.
- `MarchlineAgent.Warp(position)`: move an agent instantly (a respawn, a
  cutscene cut). It arrives idle, with its order and any crossing dropped.

## [1.0.3] - 2026-10-02

Wrapper-only, like 1.0.2: native plugins and ABI (205) unchanged.

### Added

- `MarchlineLink.waypoints`: points a crossing passes through between the
  entrance and the exit, in order. An escalator's flat landings, a stair's
  half-landing or a ramp's bend can be followed exactly: smoothed crossings
  walk the path at constant speed, and goal snapping finds a target standing
  anywhere on it. A straight entrance-to-exit line floated agents up to a
  metre above an escalator's lower landing and cut through its upper one.
  Planning still uses the two ends only. The link gizmo draws the path.
- `MarchlineLink.GetPath(list, reverse)`: the crossing's path in world space.

## [1.0.2] - 2026-10-02

Found integrating Marchline into a multi-storey game (a three-floor mall with
escalators). Wrapper-only: the native plugins and ABI (205) are unchanged
from 1.0.1, so simulation results, saves and replays are identical.

### Fixed

- Trigger colliders no longer block navigation. The live scan and the
  layered-grid baker counted triggers on `obstacleMask` as walls, so a
  level-wide volume (post-processing, audio zones, checkpoints) walled off
  every cell it covered — in the mall, a post-processing volume left 15% of
  the concourse walkable and every cross-floor order Unreachable. Both now
  ignore triggers regardless of the project's "Queries Hit Triggers"
  setting. A trigger that should block must become a solid collider.
- Agents crossing a link (stairs, escalator, door) were drawn standing at the
  entrance for the whole crossing and then appeared at the exit. They are now
  drawn moving from the entrance to the exit over the link's `durationTicks`.
  Display only: the simulation still holds the agent at the entrance, so
  determinism, saves and replays are unaffected. Teleporters still jump.

### Added

- `NavWorld.smoothLinkCrossings` (default on): turn it off to keep the 1.0.1
  presentation.
- `MarchlineAgent.CurrentLink` and `MarchlineAgent.LinkProgress` (0 to 1):
  the link an agent is crossing and how far along it is, to drive climb or
  door animations.

## [1.0.1] - 2026-09-26

### Fixed

- Goal snapping (`NavWorld.SnapToWalkable`, `MoveTo(pos, snapToWalkable: true)`)
  now respects floors. It searched the ground grid only, so a target on a
  baked upper floor above blocked ground (a tower deck over the keep) was
  silently re-aimed at the foot of the tower, and a target on an unwalkable
  cell of its floor (a parapet) reported Unreachable. The snap now searches
  the floor the target's height selects, keeps that floor's height, and falls
  back to the nearest other floor only when its own has nothing within reach.
- A chase target standing on a stairs link (mid-ramp, on no floor) snapped to
  the ground cell beside the ramp, where pursuers parked under it, Arrived,
  with no route to it. The snap now recognises a target on an enabled link's
  crossing (within its `width` plus half a cell of margin for a body on the
  edge, at the crossing's height there) and sends the order to the nearer end
  of the link, so pursuers wait where the target must step off.
- Agents standing inside a blocked cell (an obstacle rising under them, or a
  spawn inside a wall) step out to the nearest walkable cell instead of
  freezing.
- Chase AI that re-issues `MoveTo` every few frames no longer resets the
  agent's stuck detection with every order: a positional stall clock keeps
  escalating to a reroute, so re-aimed pursuers still leave a plugged doorway.
- Core: a flat `MoveTo` (a ground-floor goal) issued to an agent while it is
  crossing a link pulled it out of the link back onto the floor and never
  released the link's capacity slot — a chaser whose target ran back down the
  stairs while it climbed. After `capacity` such re-aims the link admitted
  nobody, for good (the Upstairs Chase hunters stopped climbing after a few
  quick reversals). The held order now waits for the crossing to complete;
  landed upstairs it routes back down through the layered planner. Ships
  with native plugins rebuilt from this core on every platform.

### Added

- `NavWorld.SnapToWalkableOn(layer, pos, maxRadius)` and
  `NavWorld.IsWalkable(layer, cell)`: per-floor snapping and the walkability
  predicate behind it (the live scan mirror for the ground, baked bytes for
  upper floors).
- `MarchlineLink.width`: plan width of a link's crossing surface (0 = one
  cell), used by goal snapping to recognise targets standing on the link.

### Changed

- Ground-floor snapping is judged by the scan mirror the planner plans on
  rather than a fresh physics query per cell: the same answer once a cell has
  been scanned, and no physics cost per order.
- Snapping returns the closest free cell, not the first one scanned in a ring
  (a target under the middle of a building could land a cell further away
  than necessary).

## [1.0.0] - 2026-08-16

Initial release.
