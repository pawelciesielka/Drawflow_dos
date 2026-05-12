# ADR 0002 — Orthogonal routing: step path, minimal obstacle awareness, transit-point waypoints

Date: 2026-05-12
Status: Accepted

## Context

For the orthogonal connection style introduced in ADR 0001 we need to fix three closely-related choices that together define the *shape* of an orthogonal connection:

1. **Routing algorithm.** How is the path between two ports computed?
2. **Obstacle awareness.** Which nodes does the router treat as obstacles to avoid?
3. **Waypoint semantics.** Drawflow already supports user-placed reroute points (`connections[i].points`) for Bézier connections. What do these points *mean* in orthogonal mode?

These three are coupled: the answer to "what does a waypoint mean" determines what the router has to compute between waypoints; the answer to "how smart is the router" determines whether users will even need waypoints; the routing algorithm dictates the visual stability of the line under node drag.

Once shipped, all three are hard to change:

- Switching the routing algorithm changes the rendered shape of every existing orthogonal connection.
- Changing waypoint semantics is a breaking change in `connections[i].points` interpretation (a v1 JSON would render differently in v2).
- Adding obstacle awareness later is non-breaking; *removing* it is breaking (users will have grown to rely on auto-routing around third-party nodes).

## Decision

**Routing.** Step path with three segments in the forward case (target right of source): horizontal stub from output → vertical → horizontal stub to input. The vertical segment is placed at `source.x + max(stub, dx/2)` ("adaptive mid-X"): mid-X for normal distances, fixed-stub fallback when source and target are close enough that mid-X would land inside a node. Default `stub ≈ 30–40px`.

For back edges (target.x ≤ source.x) the path expands to five segments: stub right from output, vertical, horizontal bypass running above or below both nodes, vertical, stub left into input. The bypass side defaults to `sign(source.y − target.y)` — if the target is above the source, go over the top; if below, go under. The user can override the side per-connection via the optional `bypass_side` field on the output-side data entry (`'top'` / `'bottom'`; absent → automatic). This addresses the failure mode where the auto-chosen side happens to be densely populated with other nodes and the line gets visually lost — the user flips the side rather than reaching for a heavyweight pathfinder.

Self-loops (same node) fall back to Bézier in v1. No special-case orthogonal trace; reconsider when a real use case appears.

**Obstacle awareness.** The router treats only the source-node bbox and target-node bbox as obstacles. Unrelated nodes on the canvas are not avoided automatically. When a routed line passes through a third-party node, the user resolves it by adding a waypoint.

**Waypoint semantics.** A waypoint is a *transit point* — a position the connection must pass through — exactly as it has always meant for Bézier connections. It is **not** a corner. Between two adjacent waypoints (or port↔waypoint) the renderer draws an independent full step path using the same adaptive mid-X algorithm. Waypoint drag remains free in 2D, unchanged from the Bézier code path.

## Alternatives considered

**Routing — alternatives**

- *Naive elbow without stubs* — rejected. Without stubs, lines exit ports at angles that visually contradict the port orientation (output is on the right edge; a path going up-and-left from it looks broken). Stubs are cheap and fix this.
- *Full pathfinding (A* on grid or visibility graph)* — rejected for v1. Three reasons: (a) line shape would change every frame as a third-party node is dragged near the route, destroying visual stability; (b) recomputing A* for every connection on every drag-move of any node is O(N·M·grid_cells), unacceptable for a library aiming at small bundle size and snappy UX; (c) bundle cost of a competent pathfinder (~500–1000 LOC) is disproportionate to a ~50 KB library. A* remains a viable opt-in extension behind a per-connection flag if real demand appears, but is explicitly out of scope for v1.

**Obstacle awareness — alternatives**

- *Avoid all nodes in the corridor between source and target* — rejected. Costs O(N) per connection per drag-frame and, more importantly, makes the shape of one connection a function of *other, unrelated* nodes' positions. That coupling is the thing users find confusing in heavy-handed routers (lines "jumping" when an unrelated node moves). Drawflow already gives users a deterministic escape hatch — the reroute point — and we prefer to lean on that.
- *Per-connection `obstacle_avoidance: true` flag* — rejected for v1. No use case yet; we can add the flag later without changing existing semantics.

**Waypoint semantics — alternatives**

- *Waypoint = corner (Visio / Lucidchart / Diagrams.net model)* — rejected. Requires (a) reinterpreting the existing `points` field (breaking change for Bézier consumers), (b) constraining drag to one axis with self-correction logic when the user releases in a non-orthogonal position, and (c) a separate code path in the position handler. The win is aesthetic ("looks like Visio") but the cost is a divergent meaning for `points` depending on style — a footgun for anyone reading export JSON. If this becomes a real pain point, a v2 opt-in mode (`orthogonal_waypoint_mode: 'elbow'`) is non-breaking.

## Consequences

**Positive**
- Connection shape is a pure function of `(source_port_position, target_port_position, waypoints)`. Predictable, testable, stable under arbitrary node motion elsewhere in the diagram.
- One waypoint code path across both styles. No new interaction code; the existing drag-point handler stays as is. The only change in `updateConnectionNodes` is a style-dispatched switch on the *segment generator*: `createCurvature(...)` for Bézier, `createOrthogonalPath(...)` for orthogonal.
- O(1) routing per connection. Performance under drag is identical to the current Bézier path.
- Bundle cost is the size of a step-path generator plus the rounded-corner helper — order of 100 LOC, no new dependencies.

**Negative**
- Orthogonal connections can visually pass through third-party nodes. Acceptable because (a) the user has an explicit, learnable workaround (add a waypoint), (b) the alternative is the visual-instability problem of pathfinders, which we judge worse.
- A user expecting Visio-style "drag the elbow" UX will be surprised when adding a waypoint produces two H–V–H segments meeting at the waypoint rather than a single corner. Mitigated by `CONTEXT.md` Waypoint entry; reconsider if it generates real friction.
- Adaptive mid-X can still place the vertical segment over a third-party node in dense layouts. Same escape hatch (waypoint) applies.
