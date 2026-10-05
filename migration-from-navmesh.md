---
layout: default
---

# Migrating from Unity NavMesh

Two routes: the drop-in shim (minutes) or the native API (better results).

## Route 1: the shim

`NavMeshAgentCompat` mirrors the everyday `NavMeshAgent` surface. In your
movement scripts, swap the type:

```csharp
// before
var agent = GetComponent<UnityEngine.AI.NavMeshAgent>();
agent.SetDestination(target);

// after
var agent = GetComponent<Marchline.NavMeshAgentCompat>();
agent.SetDestination(target);
```

Supported: `destination`, `SetDestination`, `speed` (world units per
second, as in NavMesh), `isStopped`, `pathPending`, `hasPath`,
`remainingDistance` (straight-line approximation; for the walking distance
use `MarchlineAgent.RemainingDistance()`), `ResetPath`, `Warp`. Scene setup: delete your baked NavMesh and NavMeshAgent
components, add one `NavWorld` (it scans obstacles itself — no baking),
and put `MarchlineAgent` + `NavMeshAgentCompat` on each unit.

Not mirrored (use the native API instead): off-mesh links, area costs,
`NavMeshObstacle` carving (use `MarchlineObstacle` — it is both simpler
and live), agent radius/height per-agent shape (Marchline agents share a
hard collision radius derived from cell size).

## Route 2: native API

| NavMesh concept | Marchline equivalent |
|---|---|
| Baking a NavMesh | None — `NavWorld` scans physics at Play, live-updates after |
| `NavMeshAgent.SetDestination` | `MarchlineAgent.MoveTo(worldPos)` |
| Many agents, one target | `NavWorld.MoveGroup(...)` — ONE shared flow field, any group size |
| `NavMeshObstacle` (carve) | `MarchlineObstacle` — instant, moving-obstacle-aware |
| `NavMesh.SamplePosition` | `NavWorld.WorldToCell` + your own walkability check via gizmo/`RefreshRegion` |
| Agent avoidance quality slider | Built-in: separation + anticipatory steering + hard no-overlap constraint, always on |
| `pathStatus` | `MarchlineAgent.State` (incl. an honest `Unreachable`) |
| `NavMeshAgent.speed` | `MarchlineAgent.speed` (world units per second) |
| `NavMeshAgent.isStopped`, `ResetPath` | `MarchlineAgent.Stop()` |
| `NavMesh.CalculatePath`, `NavMeshPath.corners` | `NavWorld.TryFindPath(from, to, points, out length)` |

What you gain: 10,000-agent scale, deterministic simulation (lockstep-
capable), live world edits without re-baking, group arrival that doesn't
stack, formations. What you lose today: 3D navmesh surfaces (Marchline v1
is grid-based — ideal for RTS/TD/colony maps; sloped terrain works via
the height-band scan, but true multi-storey interiors do not yet).
