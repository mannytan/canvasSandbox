# Polygon Generator — Recreation Spec

A full-screen HTML5 Canvas app that generates a random polygon guaranteed to
have no self-intersecting edges, with rounded corners, draggable vertices,
and a few supporting controls. This document specifies the behavior and
algorithms precisely enough to rebuild the app in any language/framework —
it isn't tied to Canvas or JavaScript, though the reference implementation
(`index.html` in this repo) uses both.

## 1. Core concept: a non-crossing polygon

Generate `n` random points, then **sort them by angle around their
centroid** before connecting them in order.

```
centroid = average(all points)
sort points by atan2(point.y - centroid.y, point.x - centroid.x)
```

This is the whole trick: connecting points in increasing angular order
around a shared center always produces a simple (non-self-intersecting)
polygon, for *any* set of points. No collision detection or backtracking
needed. The shape is star-shaped from the centroid by construction.

Caveats to carry over:
- This guarantee holds at **generation time only**. If vertices are later
  moved (see §4) without re-sorting, edges can cross again — that's
  expected and fine; it's a direct-manipulation editor, not a solver.
- Re-sorting *during* a drag causes a vertex's neighbors to swap abruptly
  when it crosses another vertex's angle — this reads as a confusing
  "jump" bug, not a feature. Don't re-sort after generation (see §4).

## 2. Rounded corners

Corners are rounded per-vertex using an arc tangent to both adjacent
edges (the same primitive as CSS `border-radius`, applied to an arbitrary
polygon instead of a rectangle). If your 2D API has a "line-to-line arc"
primitive (Canvas's `arcTo(x1, y1, x2, y2, radius)`, or equivalent), use
it — it does the tangent-point math for you. If not, implement the
formula in §2.2 directly.

### 2.1 Path construction

For a polygon with vertices `P[0..n-1]`:

1. Start the path at the **midpoint of the last edge** (`P[n-1]` to
   `P[0]`) — any point already lying on an edge works; the midpoint is a
   convenient one.
2. For each vertex `i` from `0` to `n-1`, draw an arc through
   `P[i]` tangent to the edges `(P[i-1] → P[i])` and `(P[i] → P[i+1])`,
   with that vertex's radius (see §2.3 for per-vertex radii).
3. After the loop, draw a straight line back to the starting midpoint
   and close the path.

**Why start at a midpoint, not at `P[0]` or an offset point:** a
tangent-arc primitive like `arcTo` needs the path's current point to
already lie on the incoming edge's line — it then draws the straight
run-up to the true tangent point *for you*. If you compute that offset
by hand rather than letting the arc primitive derive it, you'll hit the
bug in §2.2.

### 2.2 The bug to avoid: radius ≠ tangent distance

The distance from a corner to where its arc becomes tangent to an edge is
**not** the radius. It's:

```
tangent_distance = radius / tan(interior_angle / 2)
```

where `interior_angle` is the angle at that vertex between the two
adjacent edges (via the vectors from the vertex to its previous and next
neighbors). The two are only equal at exactly 90°.

If you (incorrectly) place the arc's start point at `radius` distance
from the corner instead of `tangent_distance`, the result is visually
wrong at any corner that isn't ~90°: sharp corners (interior angle < 90°)
show a stray straight line jutting out past where the arc should start;
wide/reflex corners (> 90°, especially near 180°) show the opposite
error. This is subtle because it looks *almost* right and only breaks
down away from right angles — verify visually with an irregular polygon
(not a rectangle) before considering rounding "done."

The fix (used in §2.1): don't compute the tangent offset by hand at all.
Let the arc primitive derive it from the actual radius and angle.

### 2.3 Clamping radius to fit the corner

A requested radius can be larger than a short or sharp corner can
physically hold, which makes adjacent arcs overlap. Clamp per corner,
accounting for the angle (not just edge length — see §2.2, the two are
related but not the same):

```
function clampCornerRadius(prev, curr, next, wantedRadius):
    v1 = prev - curr   # vector from corner to previous vertex
    v2 = next - curr   # vector from corner to next vertex
    d1 = length(v1)
    d2 = length(v2)
    if d1 ≈ 0 or d2 ≈ 0: return 0        # degenerate (coincident points)

    cosA = dot(v1, v2) / (d1 * d2)
    if cosA ≈ -1: return wantedRadius     # ~180°: no bend, tangent dist ~0, no cap needed

    sinA = |cross(v1, v2)| / (d1 * d2)
    tanHalfAngle = sinA / (1 + cosA)      # tan(interior_angle / 2)
    maxTangentDist = min(d1, d2) / 2      # never pass either edge's midpoint
    return min(wantedRadius, tanHalfAngle * maxTangentDist)
```

A simpler `min(wantedRadius, d1/2, d2/2)` clamp (bounding the *radius* by
edge length) is tempting but wrong for the same reason as §2.2 — it
still lets sharp corners overshoot, because it ignores that tangent
distance scales with `1/tan(angle/2)`, not with the radius directly.

### 2.4 Per-vertex radius variation

Each vertex is assigned a random weight at generation time
(`0.4 + random() * 0.6`, i.e. 40%–100%), stored on the point. The
corner-radius slider value is a **maximum**, multiplied by each vertex's
own weight to get that vertex's requested radius before clamping (§2.3).
This makes corners round by visibly different amounts rather than
uniformly, without needing per-vertex UI controls.

## 3. Data model

Each point is an object/struct with:
- `x`, `y` — position in canvas/pixel coordinates
- `radiusFactor` — a per-vertex weight in `[0.4, 1.0]`, set once at
  generation and preserved through drags/snaps (copy it along when
  transforming a point, don't regenerate it)

The polygon is just an ordered array of these points. Order encodes
connectivity — see §4 for why that matters during interaction.

## 4. Dragging vertices

- Hit-test the pointer against each point (a click/touch within some
  pixel radius, e.g. 14–16px, counts as grabbing that point).
- While dragging, update only that point's `x`/`y` (clamped to the
  canvas bounds) and re-render every frame.
- **Do not re-sort the array by angle during a drag.** Array order (i.e.
  which two neighbors each vertex connects to) is fixed once dragging
  starts. Dragging a vertex far enough will visibly self-intersect the
  polygon rather than silently reconnecting to different neighbors. This
  is the expected, intuitive behavior (same as dragging a point in
  Illustrator/Figma) — re-sorting live was tried and rejected because it
  makes a vertex's neighbors swap abruptly and unpredictably as it
  crosses another vertex's angle, which reads as a bug, not a feature.
- Optional polish: track a "hovered" point separately from the
  "dragging" point to show a grab-cursor / highlight ring before the
  user commits to a drag.

## 5. Snap to grid

A toggle that, when on:
- Rounds every generated point's `x`/`y` to the nearest multiple of a
  fixed grid step (e.g. 40px) at generation time.
- Rounds a point's position to the grid on every drag move, live.
- Immediately re-snaps all *existing* points when the toggle is switched
  on (don't wait for the next regenerate/drag).
- Optionally draws a faint grid overlay while active, so the target
  cells are visible.

```
function snapToGrid(point, step):
    return { x: round(point.x / step) * step, y: round(point.y / step) * step }
```

## 6. UI controls

| Control | Range / Type | Effect |
|---|---|---|
| Regenerate button | action | Generates a fresh random point set (new count, new positions, new radius weights) and re-renders |
| Point count slider | 3–20 | Number of vertices; changing it regenerates the polygon |
| Corner radius slider | 0–1000 (px) | Max radius passed to §2.4's per-vertex scaling |
| Show points checkbox | on/off | Toggles drawing the vertex markers (dragging still works when hidden — hit-testing is independent of visibility) |
| Snap to grid checkbox | on/off | See §5 |

## 7. Rendering

- Full-viewport canvas, resizes with the window (regenerate or re-render
  on resize — don't try to rescale existing point positions).
- Polygon: filled with a translucent color (reference uses
  `rgba(80, 160, 255, 0.25)`), **no stroked outline** — the fill alone
  defines the shape.
- Vertex markers (when shown): small filled circles, ~4px radius,
  reference uses green (`#4ade80`).
- Grid overlay (when snap is on): very faint lines (reference uses
  `rgba(255, 255, 255, 0.06)`) at the same step as the snap grid.

## 8. Suggested build order

1. Canvas setup + resize handling.
2. Random point generation + angular sort (§1). Verify visually with a
   irregular/asymmetric point count that edges never cross.
3. Rounded-corner path tracing (§2) using your platform's tangent-arc
   primitive if it has one. Verify on a polygon with a mix of sharp and
   wide corners, not just a regular shape — that's where the §2.2 bug
   shows up if present.
4. Point-count and corner-radius sliders, wired to regenerate/re-render.
5. Vertex dragging (§4) with fixed connectivity.
6. Show/hide points checkbox.
7. Snap-to-grid (§5).

## Reference implementation

`index.html` in this repo is a complete, working reference in vanilla JS
+ Canvas 2D. Function names there match the pseudocode above
(`generatePolygon`, `clampCornerRadius`, `tracePolygonPath`, `snapToGrid`)
if you want to cross-check an exact translation.
