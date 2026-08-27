# csar_viz — interactive views of the ellipsoidal cone

Two pages, no build step:

- `index.html` — 3D. You set the *points*, and it solves the TEEC problem for
  them, running the real solver as WebAssembly (`vendor/`, see below). ES
  modules and wasm need http, so serve the directory rather than opening the
  file: `python3 -m http.server`.
- `playground.html` — 2D playground, self-contained and dependency-free. You
  set `A` and `b` by hand and watch `C(A, b)`.

---

# playground.html — 2D playground for C(A, b)

Interactive 2D version of the object from `csar_paper`:

    C(A, b) = { x : ||A x||_2 <= b^T x },   A > 0,  b != 0.

## What's drawn

- square (equal-aspect) axes, unit circle for reference
- the ellipse `E = { x : ||A x||_2 <= 1 }` — orange, with four semi-axis
  handles (drag to rotate + rescale each axis) and a separate rotate handle
- the axis vector `b` — blue, draggable, snapped to the unit circle by default
- the cross-section line `b^T x = 1`, and the chord `E ∩ {b^T x = 1}`
- the cone `C(A, b)` — teal, the conic hull of that chord
- `E* = A·B` (dotted blue), the admissible region for `b`: the cone has nonempty
  interior iff `b` lies *outside* `E*`. Solid blue arcs mark the unit `b`
  directions that work.
- test points, colored by whether `||A x_i|| <= b^T x_i` holds

## Why some b give the empty set

Substituting `y = A x` turns the defining inequality into

    ||A x|| <= b^T x    <=>    ||y|| <= (A^{-1} b)^T y,

so every ellipsoidal cone is the `A^{-1}`-image of a *circular* cone about
`c = A^{-1} b` with half-angle `arccos(1 / ||c||)`. That cone is more than the
origin iff `||A^{-1} b|| > 1`, i.e. iff `b` lies outside `E* = A·B` — the
ellipse with semi-axes `sigma_i = 1/r_i`, reciprocal to the one you drag.

With `||b|| = 1` locked and `A` dragged independently, directions where `E` is
thinner than the unit circle therefore fail. This is an artifact of holding
`||b|| = 1` while moving `A` separately: since `C(cA, cb) = C(A, b)`, unlocking
`||b||` and growing it rescues any direction. And the solver never sees it — one
containment constraint `||A x_i|| <= b^T x_i` with `x_i != 0` already forces
`||A^{-1} b|| >= 1`.

## Two modes

- **default**: the ellipse is origin-centered and *is* `A`. The apex of the
  cone is pinned at the origin, so translating `E` is not a degree of freedom
  of `(A, b)` — rotate and reshape it, and drag `b`.
  The cone has nonempty interior iff `||A^{-1} b|| > 1`, i.e. iff the line
  `b^T x = 1` actually cuts the ellipse.
- **free ellipse**: the ellipse can be translated, and the cone is its conic
  hull (the tangent wedge from the origin). The panel then shows the derived
  `(A, b)` for that wedge.

## Caveat

In 2D the cross-section of the cone is a *segment*, so every ellipsoidal cone
is just a wedge — and every wedge of half-angle α about a unit axis m is
`C(cos(α)·I, m)`, i.e. circular. The shape degree of freedom in `A` only
becomes visible in 3D, where the cross-section is a genuine ellipse. This page
is for intuition about how `A` and `b` jointly generate the cone, not about
cross-section aspect ratio (CSAR).

## Controls

drag handles · scroll to zoom · `+ point` / drag points / double-click to
delete · `r` or **reset all** to restore defaults.


---

# index.html — the TEEC problem in 3D

The controls are the points. Drag them on the unit sphere and the page re-solves

    minimize   -log det A
    subject to ||A x_i||_2 <= b^T x_i,  i = 1..n
               ||b||_2 <= 1

from scratch (~3 ms), drawing the resulting cone, its rim on the sphere, and the
support set (the points that end up on the cone boundary).

## How it solves it in the browser

It runs **csar itself**, compiled to `wasm32-freestanding`. The page loads
`vendor/csar.wasm` through `vendor/csar.js` — the ABI's own JavaScript
declaration — both vendored as a pair from a
[csar_abi](https://github.com/ajfriend/csar_abi) release. So there is no second
implementation of the algorithm to keep in step, and the status codes and
result-struct offsets are declared by `csar.js` rather than transcribed here —
csar_abi gates those declarations against the solver's C ABI. (One constant is
still spelled out in the page: the `1e30` no-certificate gap sentinel, which
the ABI passes through but does not yet name.) The panel reports the *certified
duality gap* instead of asserting a number, and rank-deficient input is
reported as such rather than silently answered.

`vendor/` holds a matched pair from one release — `csar.js` reads the result
struct at offsets `csar.wasm` defines, so they are only valid together. The
page checks that the module agrees with the version it was written against, and
shows the solver and ABI versions in the panel. `vendor/PROVENANCE` records the
release, the checksums, and how to update.

The page previously carried its own JavaScript solver. What that solver did,
and why it is the same algorithm, is kept below because it explains the picture
the page draws — the gnomonic plane, the cross-section ellipse, the support set.

No SDP solver is needed. The problem collapses for a fixed axis `b` — see the derivation in
the "Why some b give the empty set" section above for the change of variables, and:

    G = A^2 in the frame (f1, f2, b), complete the square
      -> (u_i - v)' H (u_i - v) <= 1   for the gnomonic projection u_i
      -> det G = tau^2 (1 - tau) det H,   which separates

so `tau = 2/3` and `H` is the **ordinary 2D minimum-area enclosing ellipse** of the
gnomonic projection onto the tangent plane at `b`. Therefore

    minimise -log det A  ==  minimise the AREA of the MVEE of the projection.

* inner: Wolfe-Atwood (Frank-Wolfe with away steps) for the 2D MVEE, plus a
  rescale safeguard so the returned ellipse always encloses the points even if
  the iteration stops short. The cone is therefore always primal feasible.
* outer: the optimum has the MVEE centred on the tangency point (`v = 0`), so the
  step is "re-aim b at the centre of the cross-section ellipse", with backtracking
  on `det H` and a bounded pattern search as fallback.
* feasibility: the points lie in an open half-space iff the origin is outside
  `conv{x_i}`. Gilbert's algorithm finds the min-norm point `z*` of that hull;
  `z*/||z*||` is the max-margin axis and the starting `b`.

## Checks it passes

* `b` comes out an eigenvector of `A` with eigenvalue exactly `1/sqrt(3)` — the
  paper's appendix result, never imposed by the code. Shown in the panel.
* max `||A x_i|| - b^T x_i` is `<= ~1e-14` on every configuration tested. Also
  shown in the panel, so it is checkable live rather than asserted here.

The removed JS solver additionally agreed with a 90,000-axis brute-force scan
of `-log det A` to ~5e-13, and matched an independent max-margin oracle on 300
random point sets in both directions. The wasm solver carries its own test
suite upstream, and reports a certified duality gap per solve — a stronger
statement than either.

## Display

The "convex hull of the points" toggle fills the geodesic convex hull of the
input as a spherical polygon — the cell the cone is enclosing. Because the
gnomonic projection sends great circles to straight lines, the planar hull of
the projected points is exactly the geodesic hull on the sphere, so it is
computed with an ordinary 2D monotone chain and mapped back.

The cap is tessellated in both directions — along each edge AND radially from
the centre — because subdividing only the edges leaves long thin triangles
running centre-to-rim that chord straight through the sphere, so a large cap
visibly sags.

Triangles are grouped into depth bands and each band is filled as ONE compound
path. Filling adjacent translucent triangles separately leaves antialiasing
seams, so the mesh shows through as a web of hairlines; a single fill has no
interior edges. That also means interior mesh density is invisible, so the
along-edge subdivision is sized in SCREEN space (a target edge length in pixels,
which accounts for zoom) while the radial direction stays coarse. Cost is ~0.4 ms
for a normal cap and ~1 ms for a zoomed-in near-hemisphere one.

The "coastlines" toggle draws Natural Earth's 110m coastline on the globe,
which makes the sphere's orientation legible and puts the DGGS use case in
view. The data is embedded, not fetched: quantised to 0.1 degrees and delta
encoded, 133 rings / 5119 points / ~27 KB — small enough to inline, and it has
no ABI to keep in step the way the solver does. Natural Earth is public domain;
the same source `csar_zig/scripts/gen_countries.py` uses.

The "gnomonic plane" toggle draws the tangent plane at `b`, the projected points,
and the cross-section ellipse — whose axis ratio *is* the CSAR. Note the cone rim
and that ellipse are the same curve, and its centre sits exactly on the tangency
point: that is the optimality condition, visible.

## Trap: sigma_0 is not the smallest eigenvalue

`sigma_0 = 1/sqrt(3)` is the eigenvalue of `A` whose eigenvector is `b`. It is
NOT necessarily the smallest. Once the cone exceeds ~54.7 deg in some
direction, that tangent semi-axis drops below `1/sqrt(3)`, so identifying
`sigma_0` by sorting mislabels it and corrupts the reported CSAR — at an 85 deg
cap it read 4.44 instead of 1.53. The removed JS solver identified it by
eigenvector alignment with `b`; the wasm solver avoids the trap structurally —
its `sigma[0]` is the axial eigenvalue by construction, which is what the page
relies on now.
