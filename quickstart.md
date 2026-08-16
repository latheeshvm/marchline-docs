---
layout: default
---

# Quickstart & recipes

The fastest route from an installed package to moving agents, then a
recipe per genre. Every configuration option referenced here is covered
in depth in the [manual](index.md).

## Your first agents: a player and two enemies

The smallest real integration — an action game where enemies hunt the
player — takes one scene component, one layer setting, one component per
enemy, and about a dozen lines of code.

**1. Add a Nav World (once per scene).** Create an empty GameObject and
add **Marchline → Nav World**. Size `gridSize` to cover your play area
(1 cell = 1 world unit by default) and pick a preset. On Play it
physics-scans your level's colliders into walkable cells — there is no
bake button.

**2. Exclude characters from `obstacleMask`.** Put the player and the
enemies on a character layer and untick that layer in the Nav World's
obstacle mask. This is the single most common setup mistake: a player's
`CharacterController` **is** a collider, and with the default
everything-mask the scan walls off the player's own cell — every chase
goal becomes unreachable and "enemies never arrive." Characters are never
walls; only level geometry belongs in the mask.

**3. Add a Marchline Agent to each enemy.** That is the entire per-enemy
setup. Marchline drives the enemy's transform; your animator reads the
motion. Set `OverrideSpeed` if one enemy should run faster than the
preset default.

**4. The player is not an agent.** The player is input-driven — Marchline
only needs their position as a goal.

**5. Give the enemies intent.** Either copy the `ChaseEnemy` reference
from the Chase demo (patrol → aggro → hunt through doorways → attack in
range → give up and walk home), or write the minimal hunter:

```csharp
using Marchline;
using UnityEngine;

[RequireComponent(typeof(MarchlineAgent))]
public class Hunter : MonoBehaviour
{
    public Transform player;
    public float attackRange = 1.6f;
    MarchlineAgent agent;
    Vector3 lastGoal;

    void Start() { agent = GetComponent<MarchlineAgent>(); }

    void Update()
    {
        // Re-order only when the target has actually moved a cell.
        // Orders are cheap and stale plans are discarded, but this
        // keeps the planner idle while nothing changes.
        if ((player.position - lastGoal).sqrMagnitude > 1f)
        {
            lastGoal = player.position;
            agent.MoveTo(lastGoal);
        }
        // Ground-plane distance — pivot heights inflate 3D distances.
        Vector3 d = player.position - transform.position; d.y = 0f;
        if (d.sqrMagnitude < attackRange * attackRange) Attack();
    }

    void Attack() { /* damage, animation, cooldown */ }
}
```

If the player walls themselves off, `agent.State` reports
`Unreachable` — honestly, in the inspector too — and the enemies re-plan
on their own the moment the world opens up again.

**6. When something looks wrong.** Switch the Nav World to synchronous
update while developing, enter Play, and select an enemy: the Explain
panel shows its state, goal, and cost-to-go, and the flow-field overlay
draws the route it believes in. If agents stand still, check
`obstacleMask` first — it is that, ninety percent of the time.

## Grid sizing and sectors

Marchline's hierarchical pathfinder cuts the grid into **32x32-cell
sectors**. Paths that start and end inside one sector are direct and
optimal; paths crossing sector borders route through border portals and
are bounded at 1.5x optimal — imperceptible on RTS-scale maps, but on a
small action map a cross-sector route can visibly bow away from the
straight line.

The rule of thumb: **for action-scale maps, pick a `cellSize` that fits
the playfield in one sector.** A 60x60-unit level at `cellSize 2` is a
30x30 grid — single sector, every path direct. The NavWorld's scene
gizmo draws sector boundaries in orange, and its inspector suggests the
single-sector cell size when your map qualifies.

Two consequences of changing `cellSize`: agent speeds are **cells per
tick**, so halving the cell count means halving speed values to keep
the same meters/second; and walls thinner than a cell still block the
whole cell they touch.

## Chasing targets near walls

A chase target hugging a wall can put its center inside a blocked cell —
and an order to a blocked cell honestly reports `Unreachable`. For chase
AI, order with goal snapping instead:

```csharp
agent.MoveTo(player.position, snapToWalkable: true);
```

The goal lands on the nearest walkable cell (same physics predicate the
scanner bakes with), so pursuers close to biting range instead of
parking. `NavWorld.SnapToWalkable(worldPos)` exposes the snap directly.

## Animating characters

Marchline drives agent transforms; your Animator rides on top. The
recipe that works (verified with Mixamo characters):

- **In-place playback**: for clips that physically travel, bake root
  rotation and Y into the pose but leave root XZ *unbaked*, and disable
  `applyRootMotion` — the sim stays the only mover. (Baking XZ into the
  pose makes the body glide off its transform and snap back per loop.)
- **Smooth the model, not the sim**: simulation positions advance in
  discrete ticks. Put the model on a child and glide it toward the root
  each frame; measure animator Speed from the *smoothed* child so blend
  trees don't shiver between gaits.
- **Speed units**: `OverrideSpeed` is cells per tick. At 30 ticks/s and
  `cellSize 1`, a 2.8 m/s walker is `0.095`; the RTS preset default
  (0.35 = 10.5 m/s) reads as teleporting at character scale.
- **Blend trees over state switches**: drive Idle-Walk-Run from measured
  speed with thresholds at each clip's natural stride speed.

## Recipes

### RTS squads

Select agents however your game does it, then issue one group order —
Marchline coalesces shared destinations into a single flow field
automatically, so a thousand-unit order costs barely more than a
fifty-unit one:

```csharp
// selected: List<MarchlineAgent> from your drag-select box
nav.MoveGroup(selected, clickPoint);                       // move
nav.MoveGroup(selected, clickPoint, keepFormation: true);  // keep shape
```

Use the **RTS** preset. The Mini RTS demo scene is this recipe complete:
drag-select, right-click orders, formations, building placement.

### Tower defense

Creeps are agents ordered to the base; towers are colliders with a
`MarchlineObstacle` component. Placing a tower mid-wave refreshes just
its footprint and every creep reroutes instantly — mazing works with no
extra code. If a placement fully seals the maze, creeps report
`Unreachable` (check `agent.State` to reject the placement) and resume
by themselves when a tower sells. Use the **TowerDefense** preset.

### Hordes and survivors-likes

For thousands of chasers, skip per-agent GameObjects entirely:
`MarchlineCrowd` spawns `count` native agents and draws them with GPU
instancing. Re-aim the whole horde at the player on a timer:

```csharp
crowd.MoveAll(player.position);           // one call, entire horde
int down = crowd.CountInState(AgentState.Arrived);
```

Use the **Horde** preset — it enables deterministic crowd LOD tuned for
50,000-agent scenes. The Citadel demo runs this recipe as a siege.

### Colony sims

Dozens of workers with dozens of *different* destinations is the
individual-path workload: just call `MoveTo` per worker. The planner
amortizes cold planning under a work-unit budget, so a burst of new
orders never spikes a frame. Furniture and doors carry
`MarchlineObstacle`; blocked workers report `Unreachable` and retry on
their own when a door opens. Use the **ColonySim** preset.

### Patrol and stealth AI

The `ChaseEnemy` reference in the Chase demo is the pattern: a patrol
loop of `MoveTo` waypoints; aggro inside a detect range; while hunting,
re-issue `MoveTo(player.position)` as the player moves; give up beyond a
leave range and walk home. Agent states (`Idle`, `Planning`, `Moving…`,
`Arrived`, `Unreachable`) are all you need for the state machine — no
coroutine juggling.

### Multi-storey buildings

Bake upper floors once with a **Layered Grid** asset, then connect
floors with `MarchlineLink` components — stairs, jumps, teleporters, and
doors with `capacity` (units queue deterministically) and `durationTicks`
(crossing time). A plain `MoveTo` at an upper-floor height routes across
floors automatically. Toggle a door at runtime with `link.LinkEnabled` —
units inside finish crossing; units queued reroute or report
`Unreachable` until it reopens. See the manual's Vertical Worlds section.

### Doors, gates and moving props

Anything that moves and blocks: add `MarchlineObstacle`. It refreshes
its old and new footprint automatically when the transform changes —
no scan calls, no per-frame cost when it holds still. For flow-through
gates that open and close, a disabled `MarchlineLink` is often cleaner
than a physical blocker: it reroutes crowds through topology, not
geometry.

### Lockstep multiplayer and replays

Run the Nav World in synchronous mode and drive it from your fixed
tick — simulation cost is budgeted in deterministic work units, never
wall-clock, so peers with different CPUs stay bit-identical. Record any
session (`recordSession`), save it with `TakeSessionRecording()`, and
scrub it tick-by-tick in **Window → Marchline → Replay Scrubber**.
`SaveWorldBytes()` snapshots the entire simulation mid-battle,
mid-staircase, mid-queue. The full contract — what is guaranteed, what
breaks it — is in the [determinism guide](determinism.md).

## Where next

- [Manual](index.md) — every component and field, performance guidance
- [Migrating from Unity NavMesh](migration-from-navmesh.md)
- [Migrating from A* Pathfinding Project](migration-from-app.md)
- Support: **support@marchline.dev**
