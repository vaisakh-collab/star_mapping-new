# Stardome Star-Mapping Software
## Error and Logical-Issue Review

**Scope:** Full repository, not just `src/`.

**Purpose:** This is a working defect register. It deliberately separates:
1. **Obvious/confirmed errors** — things that are clearly wrong from the code itself.
2. **Logical/geometry errors** — code that may work in common cases but is mathematically or structurally unsafe.
3. **Legacy/stale problems** — broken or obsolete code that is not on the main runtime path, but should not be mistaken for healthy code.

This review is intended to complement the shorter team documentation. It is **not** a repair plan yet.

> **Review limitation:** The repository was inspected statically. A full `npm ci`/runtime execution could not be completed in this environment, so runtime-only claims are marked as likely where appropriate.

---

# A. Obvious / confirmed errors

## A1. `circumscribedRadius` UI option is broken

**File:** `src/index.html`  
**File:** `src/template/scaling.js`

The UI offers:

```text
circumscribedRadius
```

but the exported function is misspelled:

```js
export const cicrcumscribedRadius = ...
```

`template/index.js` dynamically does:

```js
scaling[netOptions.scaleDimension](...)
```

Therefore selecting **Circumscribed Radius** looks up:

```js
scaling.circumscribedRadius
```

which does not exist.

### Result

Selecting that option should produce a runtime error such as:

```text
TypeError: scaling[netOptions.scaleDimension] is not a function
```

### Severity

**High — direct user-facing failure.**

---

## A2. The Padding control does not actually control padding

**File:** `src/index.html`  
**File:** `src/app.js`  
**File:** `src/template/index.js`

The UI exposes:

```text
Padding (centimeters)
```

and `netOptions.padding` defaults to `2`.

However `template/index.js` explicitly ignores it:

```js
// TODO: generate this from netOptions.padding?
const padding = tabHeight * 1.25
```

So changing the Padding field has no effect on the SVG.

### Result

The UI tells the user they can control padding, but the program uses a hard-coded value instead.

### Severity

**Medium — definite functional/UI mismatch.**

---

## A3. `database.js` drops a magnitude query of exactly zero

**File:** `src/database.js`

`findQueryValues()` ends with:

```js
.filter(value => value)
```

A magnitude value of `0` is falsy in JavaScript.

Therefore a query containing:

```js
{ magnitude: { $lte: 0 } }
```

loses the `0` during query-value extraction.

The subsequent:

```js
Math.max(...)
```

can therefore receive no numeric magnitude.

### Result

The catalog-selection logic behaves incorrectly for a magnitude threshold of exactly `0`.

This matters because the UI allows the user to enter `0`.

### Severity

**Medium — definite edge-case bug.**

### Correct principle

Filter out only `undefined`/`null`, not all falsy values.

---

## A4. The star-catalog bucket condition is logically wrong

**File:** `src/database.js`

The code says:

```js
.filter(({min, max}) => min < magnitude || magnitude > max)
```

For a bucket `[min, max]`, this condition is not a correct "does this bucket cover the requested magnitude?" test.

The two comparisons are joined with `||`.

For the current single bucket this happens to include the file for most values, which hides the problem.

It fails at the lower boundary and would become much worse if multiple magnitude buckets were added.

### Example

For:

```text
min = -2
max = 6.5
magnitude = -2
```

the expression becomes:

```text
-2 < -2 || -2 > 6.5
false || false
```

so the bucket is not loaded.

### Severity

**High for future multi-bucket catalogs; medium in the current catalog.**

---

## A5. `loadStarCatalog()` with no magnitude query does not load the normal star catalog

**File:** `src/database.js`

With the default:

```js
loadStarCatalog({})
```

there is no magnitude value.

The code computes:

```js
Math.max(...[])
```

which gives:

```text
-Infinity
```

The bucket selection then fails to select the normal magnitude file.

The function can therefore return only the `hd_asterisms.json` data rather than the complete available star catalog.

### Severity

**Medium — broken general-purpose API behavior.**

The current UI normally supplies a magnitude query, so this can remain hidden.

---

## A6. The standalone `filter.py` is broken relative to the current repository

**File:** `filter.py`

It references:

```python
src/catalogs/hd.json
src/catalogs/asterisms.json
```

but the current repository stores these under:

```text
src/assets/
```

It also writes to:

```text
src/catalogs/hd_filtered.json
```

which likewise does not exist.

Additionally:

```python
print len(filtered)
```

is Python 2 syntax.

The current project otherwise gives no indication that Python 2 is part of the active development environment.

### Other issue

```python
if args.magnitude:
```

fails to recognize a magnitude of exactly `0`.

### Severity

**High if anyone tries to use this script; otherwise legacy/stale.**

---

## A7. The project's test command is not a test command

**File:** `package.json`

The project defines:

```json
"test": "echo \"Error: no test specified\" && exit 1"
```

Therefore:

```bash
npm test
```

always fails.

There is currently no automated test suite protecting the projection algorithms.

### Severity

**High as an engineering/process problem**, especially because the project contains complex geometry.

---

## A8. CI and dependency metadata are inconsistent

**File:** `.gitlab-ci.yml`

CI specifies:

```text
node:4.2.2
```

while the checked-in `package-lock.json` is:

```text
lockfileVersion: 3
```

Lockfile version 3 belongs to modern npm generations and is not compatible with the old npm environment normally paired with Node 4.

The project also uses extremely old versions of Webpack, Vue, Three.js, and related tooling.

### Result

The repository's declared CI environment and its current dependency metadata do not form a reliable reproducible build environment.

### Severity

**High for build/reproducibility.**

---

## A9. Two different Three.js versions are declared/loaded

**File:** `package.json`

The application dependency is:

```text
three 0.88.0
```

but `src/index.html` also loads:

```html
https://cdnjs.cloudflare.com/ajax/libs/three.js/85/three.js
```

That is Three.js **r85**.

The bundled application therefore has one Three.js version while the page also loads another global Three.js version.

Even if the global copy is not currently used directly by the application, this is a dangerous version mismatch and unnecessary duplication.

### Severity

**Medium — architectural/build error.**

---

## A10. The old SVG implementation contains a definite missing-fallback error

**File:** `src/svg.js`

The active application imports:

```js
import { drawSVG } from './template'
```

so `src/svg.js` appears to be legacy/dead code.

However, if somebody reactivates it, this code can fail:

```js
starPathsByPolygon[polygon.index].map(...)
```

with no `|| []` fallback.

A polygon with no star paths therefore causes:

```text
undefined.map(...)
```

### Severity

**Low for current runtime; important legacy warning.**

---

## A11. `template/index.js` contains an obvious typo in the SVG filename-selection condition

**File:** `src/template/index.js`

The code uses:

```js
selectedPolygons.Length
```

JavaScript arrays use:

```js
selectedPolygons.length
```

### Result

The condition:

```js
polygons.length > selectedPolygons.Length
```

evaluates against `undefined`.

For normal numbers this becomes effectively false, so the generated filename can incorrectly use:

```text
full
```

even when only a subset was rendered.

### Severity

**Low — output naming error, not geometry.**

---

# B. Geometry and logical errors

These are more important for the Stardome because they can produce **plausible-looking but physically incorrect output**.

---

## B1. The scaling formulas for the 32-face truncated icosahedron are wrong

**File:** `src/template/scaling.js`

The code uses:

```js
inscribedRadiusByFaceCount[32] =
  (sqrt(58 + 18 sqrt(5)) + sqrt(78 + 18 sqrt(5))) / 8
```

This evaluates to approximately:

```text
2.59829
```

For the standard unit-edge truncated icosahedron, the inradius is approximately:

```text
2.37713
```

while the circumradius is approximately:

```text
2.47802
```

The circumradius formula in the file is consistent with the standard value, but the inradius formula is not.

### Why this matters

If the user requests:

```text
Inscribed Radius = 10 cm
```

the scaling factor is calculated from the wrong edge-to-radius ratio.

The resulting physical template therefore does **not** have the requested inscribed radius.

### Severity

**High — physical dimension error.**

---

## B2. The same face-count scaling table is incorrectly applied to the custom Isodistant TI

This is even more important.

Both:

```text
Truncated Icosahedron
Isodistant TI
```

have:

```text
32 faces
```

The scaling code chooses formulas solely from:

```js
polygons.length
```

Therefore both geometries use the same standard truncated-icosahedron radius constants.

But the Isodistant TI was deliberately constructed with a different geometry and normalized so that its face-plane distance is:

```text
1
```

Its edge-length/radius relationship is therefore different.

Using the standard TI formula for it is mathematically invalid.

### Consequence

`Inscribed Radius` and `Circumscribed Radius` scaling are wrong for the Isodistant TI.

### Severity

**Critical for physical Stardome work.**

### Required conceptual fix

Scaling should be based on **geometry metrics**, not merely face count.

---

## B3. Tab dimensions are calculated from only the first polygon

**File:** `src/template/index.js`  
**File:** `src/template/tabs.js`

The code does:

```js
const tabMaker = getTabMaker(
  transformations,
  tabScale,
  polygons[0].polygon
)
```

The tab geometry is therefore calculated from polygon 0.

A truncated icosahedron contains:

```text
12 pentagons
20 hexagons
```

Even though their edge lengths are equal, their interior angles are different.

Therefore tab geometry cannot safely be derived from one polygon and reused for every face.

### Severity

**High for truncated-icosahedron templates.**

---

## B4. `getTabAngle()` produces an obviously suspicious value for hexagons

**File:** `src/template/tabs.js`

For a regular hexagon:

```text
interior angle = 120°
```

The code calculates:

```js
remainingAngle = (2*Math.PI) % innerAngle
```

For 120°:

```text
360° mod 120° = 0°
```

and therefore chooses:

```text
tabAngle = 120°
```

Then:

```js
Math.tan(tabAngle)
```

is negative.

With the current `tabScale = 0.1`, this produces a tab base factor greater than 1 for a regular hexagon.

That means the tab can extend beyond the endpoints of the edge.

This is a strong indication that the tab-angle formula is not doing what was intended.

### Severity

**High — likely physical-template defect.**

---

## B5. `Line3.prototype.intersectLine()` does not actually test intersection with the first line segment

**File:** `src/extensions/line.js`

The implementation constructs a plane perpendicular to the first line and finds where the second line intersects that plane.

It checks whether the point lies within the **second** line:

```js
const t = line.closestPointToPointParameter(point)
```

but never checks whether the point lies within the **first** line segment.

Therefore it can return a point on the infinite plane of the first segment even when that point is outside the first segment.

### Example

For:

```text
first line:  (0,0) → (1,0)
second line: (2,1) → (2,-1)
```

the computed intersection with the first line's perpendicular plane is:

```text
(2,0)
```

which is not on the first segment.

The function can still return it.

### Why this matters

This method is used during SVG/tab clipping.

False intersections can therefore create incorrect clipping and tab geometry.

### Severity

**High — core geometry utility.**

---

## B6. Multi-intersection Bézier clipping uses intersection objects as numbers

**File:** `src/template/index.js`

In the `intersections.length > 1` branch:

```js
const pieces = intersections.map((t, i, intersections) => {
  const t0 = i === 0 ? 0 : intersections[i - 1]
  ...
  const [,remaining] = curve.splitAt(t0)
  const [segment] = remaining.splitAt(t)
```

But `curve.intersectLine()` returns objects:

```js
{ t }
```

not raw numbers.

So the code is effectively passing:

```js
{t: ...}
```

where `splitAt()` expects a numeric parameter.

### Result

The multi-intersection path-clipping branch can generate invalid calculations.

### Severity

**High for affected curves; low frequency.**

---

## B7. Bézier clipping assumes control-point containment too aggressively

**File:** `src/template/index.js`

The clipping logic often decides whether a curve is inside a tab by checking its Bézier control points.

But Bézier control points do **not** necessarily lie on the curve.

A curve can:

```text
have all control points outside
```

while:

```text
the actual curve enters the region
```

or vice versa.

The code itself acknowledges this with TODO comments.

### Consequence

Some stars can be clipped incorrectly or fail to appear on a neighboring tab.

### Severity

**Medium/High — important for precise SVG output.**

---

## B8. Asterism/tab overlap detection can miss a line crossing a tab

**File:** `src/template/index.js`

The code checks whether any endpoint of the asterism quad lies inside the tab.

It explicitly contains:

```text
TODO: Account for lines that span an overlapping tab without either end
point being contained by the tab
```

So a line can pass through a tab while both ends remain outside.

The current algorithm can then miss the overlap entirely.

### Severity

**Medium — constellation-line edge case.**

---

## B9. Asterism clipping can use an undefined intersection

**File:** `src/template/index.js`

When one endpoint is inside a tab and the other is outside, the code eventually uses:

```js
intersectionPoints[0]
```

without always proving that an intersection exists.

The surrounding logic assumes the geometry guarantees one.

Given the weaknesses in `Line3.intersectLine()`, this assumption is not robust.

### Severity

**Medium — defensive/geometry correctness issue.**

---

## B10. `drawSVG()` mutates the polygon fold/cut state

**File:** `src/template/index.js`

When `disconnectPolygons` is true:

```js
polygon.cuts.push(polygon.fold)
polygon.fold = null
```

This permanently modifies the polygon data held by the current projection.

If the user later changes back to connected polygons, the original fold information is gone.

### Consequence

The SVG output can depend on what options the user selected **previously**, not just the current options.

This is particularly dangerous in a reactive application.

### Severity

**High — state-management bug.**

### Correct principle

SVG generation should operate on copies or derived local data, not permanently mutate the topology's fold/cut definition.

---

# C. Projection robustness issues

These are not necessarily broken for the current common inputs, but they are unsafe.

---

## C1. Projecting a vector can return `undefined`

**File:** `src/projections/vector.js`

The function returns:

```js
polygon && { polygonId, point }
```

So there is no result if the ray fails to hit a polygon.

But `catalogs.js` immediately does:

```js
.then(({ point }) => {
```

without checking for `undefined`.

### Result

Any failed projection becomes an exception.

### Severity

**Medium — insufficient error handling.**

---

## C2. Asterism projection assumes every referenced star was successfully projected

**File:** `src/catalogs.js`

This line:

```js
projectedStars.find(s => s.star.id === id).point
```

assumes the star exists and was projected.

If it does not:

```text
undefined.point
```

causes a crash.

### Severity

**Medium — data/projection robustness.**

---

## C3. Projection algorithms have singular-axis cases

**Files:**
- `src/projections/curve.js`
- `src/topology/Topology.js`
- `src/project.js`
- `src/components/preview.js`

Several places do:

```js
cross(...).normalize()
```

without handling the case where the vectors are parallel.

For example, rotating a vector that is already aligned with the target axis produces a zero cross product.

The code generally assumes the zero-axis case will not occur.

### Severity

**Low/Medium — robustness issue.**

It may never appear with the current geometry orientation, but it should be handled explicitly.

---

# D. Application-state and architecture problems

---

## D1. `app.js` contains duplicate/shadowed state

**File:** `src/app.js`

The `data` object contains:

```js
starQuery
asterismQuery
```

but the same names are defined again as computed properties.

The computed properties are what the application actually uses.

Similarly:

```text
filters.selectedAsterisms
selectedStars
availableStars
```

are not meaningfully used by the current projection path.

### Result

There is dead/duplicated state that makes the application harder to reason about.

### Severity

**Low functionally; high maintainability risk.**

---

## D2. `updateStars()` is largely redundant with the computed projection query

`updateStars()` loads:

```text
availableStars
```

but the actual projection calls use the computed:

```text
starQuery
```

which independently loads the catalog.

This means the application can perform redundant catalog work.

### Severity

**Low — inefficiency / confusing architecture.**

---

## D3. Preview has an asynchronous object race

**File:** `src/components/preview.js`

The component can mount before the async-computed `object` exists.

The initial `mounted()` code only sets:

```js
this.lastDirection
```

if:

```js
if (this.object)
```

Later, the `object` watcher adds the object but does not update `lastDirection`.

Yet the animation function assumes:

```js
let source = this.lastDirection
let angle = source.angleTo(target)
```

If the object arrives asynchronously after mount, `lastDirection` can remain undefined.

### Severity

**Medium/High — plausible runtime race.**

---

## D4. Event listeners are never removed

**File:** `src/components/preview.js`

`mounted()` registers:

```js
window.addEventListener('resize', ...)
```

and mouse/click listeners.

There is no `beforeDestroy` cleanup.

If the component is destroyed/recreated, listeners can accumulate.

### Severity

**Low currently; important if the UI becomes more dynamic.**

---

# E. Data-quality issues

---

## E1. The active catalog is not the 98k+ star catalog described in the README

**README:** says the full catalogue has over 98,000 stars.

Current assets contain:

```text
hd_mag__-2.00-6.50.json   8,887 stars
hd_asterisms.json           665 stars
hd.json                     774 stars
```

The active loader uses:

```text
hd_asterisms.json
hd_mag__-2.00-6.50.json
```

not a 98k-star catalog.

### Severity

**Medium — documentation/data mismatch.**

---

## E2. `hd_mag__-2.00-6.50.json` does not actually contain stars down to magnitude -2.00

Its observed range is approximately:

```text
-1.44 → 6.50
```

The filename says:

```text
-2.00 → 6.50
```

This is not necessarily a problem in itself because there may simply be no stars brighter than -1.44 in the catalog's chosen dataset.

However, the filename suggests a coverage guarantee that is not actually present.

### Severity

**Low — documentation/data-label issue.**

---

## E3. Some constellation names in the source data are misspelled

`asterisms.json` contains:

```text
Antila
Camelopardis
```

The standard constellation names are:

```text
Antlia
Camelopardalis
```

Because the application uses these strings as identifiers, changing them requires updating any dependent selections.

### Severity

**Low visually; important for data correctness.**

---

# F. Legacy / stale code problems

These do not necessarily affect the current application path, but the team should know they exist.

---

## F1. `src/svg.js` is an older SVG implementation

The current application uses:

```js
import { drawSVG } from './template'
```

The root-level `src/svg.js` is therefore not the main current SVG pipeline.

It contains its own:

- hull code
- scaling
- tab generation
- clipping
- SVG generation

Maintaining two implementations creates a major risk of someone fixing the wrong one.

### Recommendation

Mark it explicitly as legacy or remove it after confirming there are no external consumers.

---

## F2. `filter.py` is also effectively legacy

It references the old:

```text
src/catalogs/
```

layout.

It should not be treated as the current catalog-generation pipeline until repaired.

---

## F3. Several dependencies appear unused by the current runtime

The package declares libraries such as:

```text
async
eventemitter3
worker-loader
```

without an obvious active use in the current `src` pipeline.

Likewise, some custom methods such as `CurvePath.fromSvg()` appear unused.

This is not a correctness bug, but it makes modernization harder because it becomes unclear which pieces are actually required.

---

# G. Most important issues to fix first

If the team wants to turn this defect review into an action list, prioritize:

### Priority 1 — physical correctness

1. **Fix TI/Isodistant-TI scaling.**
2. **Fix tab geometry, especially `getTabAngle()`.**
3. **Fix `Line3.intersectLine()`.**
4. **Fix Bézier multi-intersection clipping.**
5. **Remove mutation of fold/cut state during SVG generation.**
6. **Add tests for geometry and SVG output.**

### Priority 2 — direct application failures

7. Fix `circumscribedRadius` spelling.
8. Fix magnitude bucket selection.
9. Fix magnitude `0` handling.
10. Fix preview async-object handling.
11. Make the Padding control actually work.

### Priority 3 — cleanup

12. Fix `filter.py`.
13. Remove/mark `src/svg.js` as legacy.
14. Clean duplicate Vue state.
15. Reconcile Node/webpack/package-lock versions.
16. Remove duplicate Three.js loading.
17. Correct stale README/data descriptions.

---

# H. What should NOT be "fixed" blindly

Some of the old code is complicated because it solves real geometry problems.

Do **not** replace these with simple alternatives without tests:

```text
projection of asterisms
polygon topology
edge adjacency
curve splitting
unfolding
tab overlap
SVG clipping
```

A replacement that looks correct in the 3D preview can still produce a physically wrong template.

---

# I. Recommended next step

Before changing the geometry, create a small regression suite around:

```text
1. Geometry counts
2. Face/edge adjacency
3. Face distances
4. Dihedral angles
5. Star → face projection
6. Asterism segmentation
7. Net/unfolding
8. Tab geometry
9. Scaling
10. SVG output
```

Then fix the defects in Priority 1 one at a time.

That gives the team a way to distinguish:

```text
"we improved the software"
```

from:

```text
"we changed the software and hope it still works."
```
