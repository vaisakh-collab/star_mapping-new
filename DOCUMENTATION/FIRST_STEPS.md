# Stardome Star-Mapping Software
## First 10 Changes — Step-by-Step Implementation Guide

### Purpose

This document defines the **first ten actual changes** the team should make to the Stardome repository.

It is intentionally narrower than the full modernization plan.

The goal of these first ten changes is to:

1. establish a reproducible baseline;
2. make the project testable;
3. correct clear existing errors;
4. remove a few dangerous inconsistencies;
5. prepare the codebase for the later foundational modernization.

**Do not begin the major dependency upgrades yet.**

The first ten changes should make the old system safer to work on before we start replacing its foundations.

---

# Working rule for all ten changes

For every change:

```text
Create branch
   ↓
Make ONE focused change
   ↓
Run tests/build
   ↓
Run the manual check listed below
   ↓
Commit
   ↓
Only then begin the next change
```

Recommended commit style:

```text
01: establish reproducible baseline
02: remove duplicate Three.js runtime
03: add geometry regression tests
04: fix circumscribed radius scaling
...
```

If a change unexpectedly alters star positions, face assignments, unfolding, or SVG geometry, **stop and investigate before proceeding**.

---

# CHANGE 1 — Freeze the current working baseline

## Goal

Create a known reference point before changing the code.

We need to know:

> "What exactly did the old application do before we started modifying it?"

## Files

Primarily repository/Git configuration.

Add:

```text
.nvmrc
```

and eventually a lockfile if one is not already present.

## Steps

### 1. Create a baseline branch/tag

Start from the current working commit.

```bash
git checkout main
git pull
git checkout -b modernization/baseline
```

After confirming that the application is the current known-working version:

```bash
git tag stardome-baseline
```

### 2. Record the environment

Use the chosen Node LTS version from the modernization plan.

Check:

```bash
node --version
npm --version
```

Record them in the project's developer documentation.

### 3. Create `.nvmrc`

Use the project's agreed Node major version.

Example:

```text
24
```

### 4. Install exactly from the existing package definition

If a lockfile does not exist, generate one using the old dependency set **before changing dependency versions**.

Do not run a general dependency upgrade.

### 5. Record baseline output

Generate and save representative outputs:

```text
baseline/
    preview.png
    sample-net.svg
```

Also record:

- selected geometry;
- magnitude limit;
- selected asterisms;
- template size;
- scale dimension.

## Verification

The old application must:

```text
npm run build
```

and, if applicable:

```text
npm start
```

The generated output should be saved as the reference.

## Commit

```text
01: establish reproducible baseline
```

---

# CHANGE 2 — Remove the duplicate Three.js runtime

## Goal

The repository currently has a dangerous inconsistency:

`package.json` uses:

```text
three 0.88.0
```

while `src/index.html` directly loads:

```text
three.js version 85
```

This means the application can have **two different Three.js versions involved in the same page**.

That is especially dangerous because the project also modifies Three.js prototypes in `src/extensions/`.

## File

```text
src/index.html
```

## Current code

There is a script similar to:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/85/three.js"></script>
```

## Change

Remove the CDN Three.js script.

The application should use the Three.js version supplied by the npm dependency and bundled by Webpack.

Do **not** upgrade Three.js yet.

The objective of this change is only:

```text
two Three.js versions
        ↓
one Three.js version
```

## Why do this now?

The custom files:

```text
src/extensions/*.js
```

modify Three.js classes.

If different parts of the application receive different Three.js constructors/prototypes, behavior can become extremely difficult to reason about.

## Verification

Check:

- application starts;
- 3D preview still appears;
- folding animation still works;
- star geometry still appears;
- SVG generation still works.

## Commit

```text
02: remove duplicate Three.js runtime
```

---

# CHANGE 3 — Add the first automated regression tests

## Goal

The current repository has:

```text
npm test
→ Error: no test specified
```

Before modifying mathematical code, we need tests.

Do not try to test everything.

Start with **small deterministic mathematical functions**.

## Recommended test framework

Use a modern JavaScript test runner appropriate for the eventual toolchain, preferably one that will also work after the later Vite migration.

The exact test framework can be selected by the team, but the tests should be runnable with:

```bash
npm test
```

## First tests

Create:

```text
tests/
```

Start with tests for:

### A. Vector conversion

Test known RA/Dec values through the coordinate conversion function.

### B. Polygon membership

Test:

```text
point inside polygon
point outside polygon
point on/near boundary
```

### C. Basic topology

For the truncated icosahedron verify:

```text
32 faces
90 edges
60 vertices
```

and:

```text
12 pentagons
20 hexagons
```

### D. Basic scaling

Test a known geometry and expected scale factor.

## Important

At this stage, **do not write tests that encode obviously wrong behavior**.

If a function appears wrong, first determine whether the test should describe the intended mathematical behavior.

## Verification

```bash
npm test
```

must now run successfully.

The tests should pass against the baseline code, except for any tests deliberately exposing a confirmed bug.

## Commit

```text
03: add initial geometry regression tests
```

---

# CHANGE 4 — Fix the Circumscribed Radius scaling bug

## Goal

Fix a definite application error.

The UI requests:

```text
circumscribedRadius
```

but `src/template/scaling.js` exports:

```text
cicrcumscribedRadius
```

The spelling differs.

Therefore the selected scaling function cannot be found by the name supplied by the UI.

## Files

```text
src/template/scaling.js
```

Possibly:

```text
src/template/index.js
```

only if the import/call path needs adjustment.

## Current problem

The UI contains:

```html
<option value="circumscribedRadius">
```

while the function is:

```js
export const cicrcumscribedRadius = ...
```

## Change

Rename:

```js
cicrcumscribedRadius
```

to:

```js
circumscribedRadius
```

Do not change the mathematics in this change.

## Add regression test

Select:

```text
Scale dimension = Circumscribed Radius
```

and verify that SVG generation does not throw an undefined-function error.

Also test that the resulting scale matches the intended formula for a known geometry.

## Verification

Test at least:

```text
Dodecahedron
Circumscribed Radius
size = known value
```

and confirm that:

- generation completes;
- SVG is produced;
- reported dimensions are finite.

## Commit

```text
04: fix circumscribed radius scaling
```

---

# CHANGE 5 — Make the Padding control actually affect the template

## Goal

The UI exposes:

```text
Padding (centimeters)
```

and stores:

```js
netOptions.padding
```

but `template/index.js` currently contains a hard-coded padding calculation and explicitly comments that it should eventually come from `netOptions.padding`.

This means the UI value does not control the actual template padding.

## Files

```text
src/template/index.js
src/app.js
```

## Current behavior

The template generator calculates padding approximately as:

```js
const padding = tabHeight * 1.25
```

while the UI provides:

```js
netOptions.padding
```

## Change

Replace the hard-coded value with the configured option.

Conceptually:

```js
const padding = netOptions.padding
```

But first verify the units.

The UI labels padding as:

```text
centimeters
```

while the geometry before final scaling is not necessarily in centimeters.

Therefore the implementation must distinguish:

```text
requested physical padding
```

from:

```text
pre-scaled model-space padding
```

### Recommended implementation

Apply padding in the same coordinate system as the final size calculation.

If `netOptions.padding` is intended to mean **final SVG/physical centimeters**, convert it consistently with the selected scale.

Do not simply substitute the value without checking the order of scaling.

## Test

Generate two otherwise identical templates:

```text
padding = 0
padding = 2
```

The second must visibly have greater outer margin.

Also test:

```text
padding = 5
```

and verify that the change is proportional.

## Commit

```text
05: connect template padding to net options
```

---

# CHANGE 6 — Fix the `.Length` typo in SVG filename generation

## Goal

Fix a definite JavaScript error in the SVG save-path logic.

## File

```text
src/template/index.js
```

## Current code

The code uses:

```js
polygons.length > selectedPolygons.Length
```

JavaScript arrays use:

```js
.length
```

not:

```js
.Length
```

## Change

Replace:

```js
selectedPolygons.Length
```

with:

```js
selectedPolygons.length
```

## Why this matters

The expression determines whether the generated SVG filename represents:

```text
full
```

or:

```text
polygons-...
```

The typo can cause the wrong filename classification.

## Test

Generate:

1. all polygons;
2. a subset of polygons.

Check the generated filename.

The output should distinguish the cases correctly.

## Commit

```text
06: fix polygon count property typo
```

---

# CHANGE 7 — Correct the star-catalog magnitude bucket condition

## Goal

Fix the catalog-file selection logic.

## File

```text
src/database.js
```

## Current code

The bucket selection is:

```js
.filter(({min, max}) => min < magnitude || magnitude > max)
```

For a normal range check, the intended condition is almost certainly:

```text
min <= magnitude <= max
```

which corresponds to:

```js
min <= magnitude && magnitude <= max
```

## Why this matters

The current condition uses:

```text
OR
```

where a range-selection operation generally requires:

```text
AND
```

The current implementation can select a bucket when the requested magnitude is outside the bucket's range.

The problem is partly hidden because the current repository has a single broad bucket.

## Change

Replace the condition with a proper range check.

For example:

```js
.filter(({min, max}) => min <= magnitude && magnitude <= max)
```

Use the project's intended boundary convention consistently.

## Important edge case

If no magnitude is present in the query, determine what the intended behavior should be.

Do not accidentally turn:

```text
loadStarCatalog({})
```

into:

```text
load no stars
```

The existing default behavior must be tested.

## Tests

Test:

```text
magnitude = -2
magnitude = 0
magnitude = 4.75
magnitude = 6.5
magnitude > 6.5
```

and verify the correct catalog files are selected.

## Commit

```text
07: fix star catalog magnitude range selection
```

---

# CHANGE 8 — Make `Line3.intersectLine()` check both line segments

## Goal

Fix a geometry robustness problem in the custom Three.js extension.

## File

```text
src/extensions/line.js
```

## Current logic

`intersectLine()` creates an intersection point and checks the parameter of the **second** line.

It does not independently verify that the resulting point lies within the first line segment.

The method is therefore asymmetric.

## Why this matters

The method is used by projection and clipping algorithms.

If the intersection lies on the infinite extension of the first line but outside its segment, the method can return an invalid intersection.

That can produce:

```text
incorrect polygon boundary crossing
incorrect curve clipping
incorrect segmentation
```

## Change

After calculating the intersection point:

```js
const t = line.closestPointToPointParameter(point)
```

also calculate the parameter for the current `this` line.

Conceptually:

```js
const tThis = this.closestPointToPointParameter(point)
const tOther = line.closestPointToPointParameter(point)
```

Require both to be within the segment bounds.

Use a consistent tolerance policy rather than relying on exact floating-point equality.

## Important

Do not change the public return type without checking all callers.

Search:

```bash
grep -R "intersectLine(" src
```

and update callers only if necessary.

## Tests

Test:

```text
segments intersect
segments do not intersect
infinite lines intersect but segments do not
parallel segments
endpoint intersection
```

## Commit

```text
08: validate both segments in line intersections
```

---

# CHANGE 9 — Fix multi-intersection Bézier clipping

## Goal

Correct the multi-intersection branch in template clipping.

## File

```text
src/template/index.js
```

## Current code pattern

The code obtains:

```js
const intersections = curve.intersectLine(edge)
```

The custom Bézier extension returns:

```js
[{t}, {t}, ...]
```

not:

```js
[t, t, ...]
```

But the multi-intersection branch treats the array as though its elements were numeric `t` values.

The code effectively does:

```js
intersections.map((t, i, intersections) => {
    ...
    curve.splitAt(t0)
    ...
    remaining.splitAt(t)
})
```

where `t0` and `t` are objects rather than numbers.

## Change

Explicitly extract the parameter values.

Conceptually:

```js
const ts = intersections.map(({t}) => t)
```

Then operate on:

```js
ts
```

instead of the original intersection objects.

## Important

The custom extension deliberately returns objects because callers elsewhere need the edge/intersection information.

Do **not** change the return type globally just to fix this one caller.

Fix the caller.

## Also check ordering

The Bézier extension sorts its intersection results by `t`.

The clipping code should preserve that ordering.

## Tests

Construct a curve/edge case with:

```text
two intersections
```

and verify that the curve is divided into the expected sections.

Also test:

```text
zero intersections
one intersection
two intersections
three intersections
```

where geometrically possible.

## Commit

```text
09: fix multi-intersection Bézier clipping
```

---

# CHANGE 10 — Clean up duplicated/shadowed application state

## Goal

Remove state that is duplicated between `data` and `computed`.

## File

```text
src/app.js
```

## Current situation

The Vue instance contains values such as:

```js
data: {
    connectedStars: [],
    starQuery: {...},
    asterismQuery: {...}
}
```

while it also defines computed properties:

```js
computed: {
    starQuery() {...},
    asterismQuery() {...},
    connectedStars() {...}
}
```

The computed versions derive their values from current UI state.

The data versions are therefore stale/duplicated state.

## Change

Keep the values that are genuinely source state:

```text
selectedAsterisms
filters
selectedGeometry
netOptions
availableAsterisms
availableStars
```

and derive:

```text
connectedStars
starQuery
asterismQuery
```

from computed properties.

Remove duplicate data fields only after confirming that no code relies on them as mutable state.

## Also inspect

The `updateStars()` method currently creates its own `connectedStars` calculation even though a computed property exists.

Refactor it so the same source of truth is used.

Conceptually:

```js
updateStars() {
    return catalogs.loadStarCatalog(this.starQuery)
        .then(stars => {
            this.availableStars = stars
        })
}
```

Do not introduce circular dependencies between:

```text
computed starQuery
→ updateStars
→ computed starQuery
```

The method should simply consume the current query.

## Why this is important

The projection pipeline should have one clear source of truth:

```text
UI state
    ↓
computed query
    ↓
projection
```

rather than:

```text
UI state
  ↘
   mutable duplicate state
  ↘
   computed state
```

## Tests/manual checks

Change the magnitude value and verify:

- available stars update;
- connected constellation stars remain included;
- projection updates;
- no stale values remain.

Change selected asterisms and verify:

- connected stars update;
- asterism lines update;
- projection updates.

## Commit

```text
10: remove duplicated application query state
```

---

# 11. What should exist after these ten changes

After completing the ten changes, the repository should have:

```text
1. reproducible baseline
2. one Three.js runtime
3. working test command
4. fixed circumscribed-radius option
5. working padding control
6. corrected polygon-count logic
7. corrected magnitude bucket logic
8. safer line intersections
9. corrected Bézier multi-intersection clipping
10. cleaner application state
```

The project should still use the **old dependency versions** at this point.

That is intentional.

---

# 12. What we do NOT do in these first ten changes

Do not yet:

```text
❌ upgrade Three.js
❌ upgrade Vue
❌ replace Webpack
❌ replace Babel
❌ rewrite topology
❌ rewrite projection algorithms
❌ redesign the net algorithm
❌ rewrite the SVG generator
❌ convert the whole project to TypeScript
```

Those are later stages.

First we establish a trustworthy foundation.

---

# 13. After Change 10

The next stage should be:

```text
Changes 1–10
      ↓
baseline + bug fixes + tests
      ↓
review
      ↓
geometry abstraction
      ↓
topology modernization
      ↓
projection modernization
      ↓
Three.js migration
```

At that point we can begin removing the application's dependence on:

```text
THREE.Geometry
THREE.Face3
prototype modifications
```

without flying blind.

---

# 14. Final checklist

Before moving beyond the first ten changes:

- [ ] Baseline tag exists.
- [ ] Node version is documented.
- [ ] Baseline SVG/reference output exists.
- [ ] Only one Three.js runtime is loaded.
- [ ] `npm test` runs successfully.
- [ ] Geometry regression tests exist.
- [ ] Circumscribed Radius works.
- [ ] Padding changes the generated template.
- [ ] Full/subset SVG filenames are correct.
- [ ] Magnitude bucket selection is correct.
- [ ] Line intersections are segment-safe.
- [ ] Multi-intersection Bézier clipping works.
- [ ] `app.js` has one source of truth for projection queries.
- [ ] The build still succeeds.
- [ ] Representative SVG output has been compared with the baseline.

Only after this checklist passes should the team begin the larger dependency and architecture migration.

---

# Summary

The first ten changes deliberately move from **safety → correctness → testability → small geometry fixes → cleaner application state**.

The progression is:

```text
REPRODUCIBILITY
      ↓
RUNTIME CONSISTENCY
      ↓
TESTING
      ↓
CLEAR BUG FIXES
      ↓
GEOMETRY ROBUSTNESS
      ↓
STATE CLEANUP
      ↓
READY FOR MODERNIZATION
```

This gives the team a controlled starting point without simultaneously changing the mathematical engine and the technology stack.
