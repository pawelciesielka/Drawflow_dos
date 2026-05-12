# ADR 0001 — Connection style is per-connection, stored on the output side

Date: 2026-05-12
Status: Accepted

## Context

Drawflow originally renders every connection as a cubic Bézier curve. We are adding a second rendering mode — orthogonal step paths — and need to decide:

1. Whether the style is a single global setting, applies per-module, or can vary per connection.
2. Where in the data model the per-connection style lives, given that connections are represented asymmetrically: rich data (waypoints) is stored on the output side at `drawflow[module].data[outId].outputs[outClass].connections[i]`, while the input-side entry `inputs[inClass].connections[i]` carries only `{ node, input }` as a back-pointer.

The decision has to outlive the v1 implementation because:

- The JSON produced by `editor.export()` is consumed by downstream tooling (storage, server-side rendering, manual edits).
- Changing where the field lives later is a breaking change for every consumer of that JSON.

## Decision

- A global default lives on the editor instance: `editor.connection_style = 'bezier' | 'orthogonal'` (default `'bezier'`).
- Individual connections may override it via a new field on the **output-side** entry only: `outputs[outClass].connections[i].style`. The input-side mirror does *not* carry this field.
- Missing or unrecognised `style` value → silent fallback to `editor.connection_style`. No exception; we want stale or hand-edited JSON to keep working.
- Public mutators:
  - Property assignment for the global: `editor.connection_style = 'orthogonal'` (no auto-refresh, consistent with `editor.curvature`).
  - Method for per-connection: `editor.updateConnectionStyle(id_output, id_input, output_class, input_class, style)` — updates data **and** re-renders that connection atomically.
  - New helper `editor.refreshConnections()` re-runs `updateConnectionNodes` for every connection in the current module, for users who flip the global and want immediate re-render.

## Alternatives considered

- **Global-only style** — rejected. Realistic diagrams want a Bézier "shortcut" line next to an otherwise orthogonal layout; forcing one style per editor would push that decision onto consumers, who would work around it with a second editor instance.
- **`style` mirrored on both output and input sides** — rejected. The codebase already stores `points` (the other piece of per-connection rich data) only on the output side. Mirroring `style` would introduce a new asymmetry in the asymmetry: two write paths, two read paths, and a desync hazard for anyone hand-editing JSON. Single-write is strictly simpler.
- **Separate top-level map** (`drawflow[module].connection_styles[key] = ...` keyed by `outId:outClass→inId:inClass`) — rejected. Introduces a new top-level structure with a composite key, inconsistent with how `points` is stored. Loses locality (style of a connection no longer lives next to the connection).
- **DOM-only state** (CSS class on the SVG element, no data field) — rejected. `editor.export()` would lose the information; importing a previously orthogonal diagram would silently revert it to the global default.

## Consequences

**Positive**
- One write site for per-connection style; no possibility of left/right desync.
- Backward-compatible: existing JSON without `style` continues to deserialise correctly and renders in the global default style.
- Setting `editor.connection_style` after construction matches the existing configuration idiom (`curvature`, `reroute`, `zoom_max`, etc. are all post-construction properties with no auto-refresh).

**Negative**
- Code that walks input-side entries (e.g. when removing a node and reindexing connections) must remember to look up the corresponding output-side entry if it needs the style. This is the same pattern that already applies to `points`, so the team is used to it.
- The asymmetry is invisible to a first-time reader of the data structure. Mitigated by the `CONTEXT.md` Connection entry.
