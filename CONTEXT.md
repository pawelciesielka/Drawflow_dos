# Drawflow — domain glossary

Single-page glossary of domain terms used in the Drawflow codebase. Terms here are meaningful to domain experts (users building flow editors), not implementation details.

## Node

A draggable block placed on the canvas. Has zero or more **input ports** on its left edge and zero or more **output ports** on its right edge. Identified by numeric `id` (or UUID when `useuuid=true`). Lives in `drawflow.drawflow[module].data[id]`.

## Port

A connection attachment point on a node. Two kinds:

- **Output port** — on the right edge. Belongs to one **output class** (`output_1`, `output_2`, ...).
- **Input port** — on the left edge. Belongs to one **input class** (`input_1`, ...).

Ports are always horizontal: outputs emit rightward, inputs receive from the left. This is structural — there is no vertical-port mode.

## Connection

A directed link from one output port to one input port. Renders as an SVG `<path class="main-path">` inside a `<svg class="connection">` wrapper appended to `precanvas`.

Stored **asymmetrically**: rich connection data (waypoints, style) lives only on the output side at `drawflow[module].data[outId].outputs[outClass].connections[i]`. The input-side entry `inputs[inClass].connections[i]` is a back-pointer with only `{ node, input }`.

## Connection style

The visual mode used to render a connection. One of:

- **Bezier** — cubic Bézier curve with two horizontal control points. The original (and only pre-existing) style. Shape parametrized by `editor.curvature`.
- **Orthogonal** — step path made of axis-aligned horizontal and vertical segments connected by rounded corners. Parametrized by `editor.connection_corner_radius`.

Default style is set globally via `editor.connection_style`. Individual connections may override it via the `style` field in their data entry (see *Connection*). Missing or invalid value → fall back to the global default.

## Waypoint (a.k.a. reroute point)

A user-placed point that the connection must pass through. Stored in `connections[i].points = [{pos_x, pos_y}, ...]`. Added by clicking on a connection (when `editor.reroute=true`), draggable in 2D.

Semantics are **identical across styles**: a waypoint is a *transit point*, not a corner. Between two adjacent waypoints (or port↔waypoint), the renderer draws an independent path segment using the connection's style — a full Bézier curve for Bézier style, or a full step path (H–V–H) for Orthogonal style. A waypoint placed mid-connection in Orthogonal style therefore produces **two adjacent step paths**, not a single elbow.

## Step path

The shape of an orthogonal connection segment. Three sub-cases:

- **Forward edge** (target x > source x): H → V → H. Three segments. Vertical segment placed at `source.x + max(stub, dx/2)` ("adaptive mid-X").
- **Back edge** (target x ≤ source x): H → V → H → V → H. Five segments. The horizontal "bypass" segment runs above or below both nodes, side chosen by `sign(source.y − target.y)` (target above → bypass on top).
- **Self-loop**: falls back to Bézier in v1.

## Stub

The short fixed-length horizontal segment leaving an output port (or entering an input port) before the first corner. Ensures every connection visually exits "to the right" of an output and enters "from the left" of an input regardless of target position. Default length ~30–40px.

## Elbow (corner)

The 90° transition between a horizontal and a vertical segment in a step path. Rendered as a rounded curve of radius `editor.connection_corner_radius` (default 8px; 0 = sharp). When a segment is shorter than `2 * radius` the effective radius is clamped to half the segment length.

## Obstacle awareness (v1)

Orthogonal routing avoids only the **source node bbox and the target node bbox**. Other nodes on the canvas are *not* avoided automatically — if a routed line passes through an unrelated node, the user resolves it by adding a waypoint. This is a deliberate trade-off: visual stability (line shape depends only on its endpoints) over routing intelligence.

## Module

A named workspace (`drawflow.drawflow[name]`). The editor renders one module at a time (`editor.module`). Connections live within a module — no cross-module edges.
