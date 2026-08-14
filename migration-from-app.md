---
layout: default
---

# Migrating from A* Pathfinding Project

Marchline and A* Pathfinding Project (APP) overlap on "agents find paths",
but they are built around different centers of gravity: APP around
per-agent path queries over rich graph types; Marchline around
deterministic, massively shared planning for crowds. Migrate when your
game is armies/crowds/RTS-scale; stay on APP if you depend on 3D recast
navmeshes or point graphs.

## Concept map

| APP | Marchline |
|---|---|
| `AstarPath` (singleton, graph owner) | `NavWorld` |
| Grid Graph | `NavWorld` grid (scanned from physics; no inspector graph setup) |
| Recast/Navmesh/Point/Hex graphs | Not in v1 — grid only |
| `Seeker` + `AIPath`/`RichAI`/`AILerp` | `MarchlineAgent` (one component; no seeker/movement split) |
| `Seeker.StartPath` callback | `MoveTo` + poll `State` (plans are delivered by the budgeted planner; agents keep their old plan meanwhile) |
| Path pooling ("don't hold `p.vectorPath`") | Not your problem — no path objects cross the API |
| `GraphUpdateObject` | `NavWorld.RefreshRegion(bounds)` or just `MarchlineObstacle` |
| `NavmeshCut` | `MarchlineObstacle` (live, physics-driven) |
| Graph scanning (`AstarPath.Scan`) | Automatic at Play + live updates; `RefreshAll()` exists but is rarely needed |
| RVO (`RVOSimulator` + `RVOController`) | Built-in: separation + anticipatory steering + hard collision, no extra components |
| Multithreading setting | Always on internally (deterministic reduction); `backgroundPlanning` moves the whole sim off the main thread |
| Penalties/tags per node | Not in v1 (uniform costs) |

## Typical port

APP:

```csharp
seeker.StartPath(transform.position, target, OnPathComplete);
void OnPathComplete(Path p) { if (!p.error) ai.destination = target; }
```

Marchline:

```csharp
GetComponent<MarchlineAgent>().MoveTo(target);
// later, if you care:
if (agent.State == AgentState.Unreachable) { /* honest failure */ }
```

Group movement — the place APP projects accumulate custom code — becomes
one call: `NavWorld.MoveGroup(units, point)` (add
`keepFormation: true` for shape-preserving moves). Delete your custom
flow-field/formation/time-slicing scaffolding; that scaffolding is the
product.

## What Marchline does NOT replace (v1)

- Recast-style 3D navmeshes and multi-storey interiors
- Point graphs / off-mesh links (jumps, teleporters)
- Per-node penalties and tag systems
- APP's editor graph inspectors

If your project depends on those, a full migration is premature — but the
deterministic C ABI still makes Marchline usable server-side alongside an
APP client.
