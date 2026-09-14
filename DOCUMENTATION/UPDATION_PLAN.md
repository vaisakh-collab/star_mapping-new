# Stardome Star-Mapping Software
## Project Update & Modernization Plan

**Purpose:** A practical, ordered plan for bringing the existing Stardome software up to date while preserving its mathematical and physical behavior.

This is deliberately a **plan of execution**, not a replacement for the detailed architecture documentation or the error register.

---

# 1. The guiding principle

We should **modernize the existing system incrementally**, not rewrite it from scratch.

The software contains specialized geometry for:

- celestial projection
- polyhedron topology
- truncated-icosahedron geometry
- line/curve intersections
- unfolding
- tabs
- SVG generation

These are the parts we cannot afford to accidentally change while modernizing the surrounding technology.

Therefore:

> **First make the existing behavior testable. Then update the foundations. Then update the higher-level geometry. Finally update the UI/build system.**

Every stage should leave the project in a buildable, testable state.

---

# 2. Current dependency situation

The repository's `package.json` currently uses a very old stack.

## 2.1 Runtime dependencies

| Current | Current version | Recommended direction |
|---|---:|---|
| `three` | 0.88.0 | **0.186.0** eventually |
| `vue` | 2.1.6 | **Vue 3.5.42** eventually |
| `vue-async-computed` | 3.1.2 | **4.0.1**, or preferably remove by redesigning async state |
| `bezier-js` | 2.2.3 | **6.1.4**, but requires API migration |
| `convexhull-js` | 1.0.0 | **No maintained upgrade; evaluate replacement/removal** |
| `debounce` | 1.0.0 | **3.0.0** |
| `eventemitter3` | 2.0.3 | **5.0.4** |
| `lodash` | 4.17.4 | **4.18.1** |
| `parse-svg-path` | 0.1.2 | **0.2.0** |
| `sift` | 3.2.6 | **17.1.3** |
| `async` | 2.4.0 | **3.2.6**, but appears unused directly |
| `threestyle` | 0.2.1 | **Evaluate/remove; only preview.js uses it** |

The versions above are current package versions checked during preparation of this plan. They should be pinned in the project's lockfile after compatibility testing rather than blindly using floating versions. Three.js is currently 0.186.0, Vue is 3.5.42, and the Node.js LTS line is 24.21.0. citeturn0search4turn2search8turn4search0

---

# 3. Development/build dependencies

| Current | Current version | Recommended direction |
|---|---:|---|
| `webpack` | 2.5.0 | **5.111.0**, or eventually replace with **Vite 8.3.0** |
| `webpack-dev-server` | 2.4.5 | **6.0.0** if staying on Webpack |
| `babel-core` | 6.17.0 | Replace with **`@babel/core` 8.0.5** |
| `babel-loader` | 6.2.5 | **10.1.1** |
| `babel-preset-es2015` | 6.16.0 | Remove; use **`@babel/preset-env` 8.0.2** |
| `babel-plugin-transform-object-rest-spread` | 6.23.0 | Remove if no longer needed by the chosen Babel configuration |
| `babel-polyfill` | 6.23.0 | Remove; use targeted/polyfill strategy if actually required |
| `css-loader` | 0.28.7 | **7.1.5**, but reassess because current webpack supports native CSS handling |
| `style-loader` | 0.19.0 | **4.0.0** |
| `extract-text-webpack-plugin` | 2.1.2 | **Remove; use `mini-css-extract-plugin` 2.10.2** |
| `file-loader` | 1.1.5 | **Remove; use Webpack 5 asset modules / Vite assets** |
| `html-loader` | 0.4.5 | **5.1.0**, if still required |
| `html-webpack-plugin` | 2.28.0 | **5.6.8** |
| `worker-loader` | 0.8.0 | **Remove/rework using native Webpack 5/Vite worker support** |
| `eslint` | 3.7.1 | **10.10.0** |
| `eslint-plugin-react` | 7.4.0 | **Remove unless React code is introduced; the project is Vue** |
| `whatwg-fetch` | 2.0.1 | **3.6.20**, or remove for modern browser targets |

These are not all direct "change version X to version Y" operations. Several packages are obsolete and should be replaced by the facilities of the modern toolchain.

For example, `extract-text-webpack-plugin` is deprecated and explicitly recommends `mini-css-extract-plugin`; `babel-preset-es2015` is deprecated; and modern `babel-loader` requires Babel 7/8 rather than the project's Babel 6 setup. citeturn2search10turn2search0turn3search5turn1search14

---

# 4. Recommended platform

Use:

```text
Node.js 24 LTS
npm 11.x
```

Node 24 is currently the LTS line. Node 26 is currently the Current line, so Node 24 is the safer baseline for a long-running college project. citeturn4search0turn4search11

Create a project-level version declaration, for example:

```text
.nvmrc
```

containing the chosen Node major version.

---

# 5. Important dependency decisions

## 5.1 Three.js

Target:

```text
0.88.0
   ↓
0.186.0
```

This is **not** a simple dependency update.

The project currently depends heavily on old Three.js concepts such as:

- `Geometry`
- `Face3`
- custom prototype extensions
- old geometry APIs

Therefore Three.js must be migrated only after the internal geometry layer has been isolated and tested.

The current Three.js release is 0.186.0. citeturn0search4

---

## 5.2 Vue

Target:

```text
Vue 2.1.6
   ↓
Vue 3.5.42
```

Vue 2 reached end-of-life on December 31, 2023. Vue's official migration documentation recommends a compatibility build for gradual migration. citeturn0search0turn0search1

Do **not** jump directly from Vue 2.1.6 to Vue 3 in one commit.

Recommended route:

```text
Vue 2.1.6
    ↓
Vue 2.7
    ↓
@vue/compat
    ↓
Vue 3
```

Vue 2.7 is the final Vue 2 minor release and provides several APIs that make the migration easier. citeturn0search2

---

## 5.3 Build system

The current project uses:

```text
Webpack 2
+
Babel 6
+
old loaders
```

The long-term target should be:

```text
Vite
+
Vue 3
```

Vue's current recommendations favor Vite over Vue CLI/older build tooling. citeturn0search13

However, **do not migrate to Vite at the beginning**.

First stabilize the source code and geometry.

---

# 6. The actual update order

The project should be updated in the following order.

```text
PHASE 0  Freeze + baseline
   ↓
PHASE 1  Correct obvious errors
   ↓
PHASE 2  Create tests
   ↓
PHASE 3  Modernize basic JavaScript/data layer
   ↓
PHASE 4  Modernize geometry primitives
   ↓
PHASE 5  Modernize topology
   ↓
PHASE 6  Modernize projection
   ↓
PHASE 7  Modernize unfolding/net system
   ↓
PHASE 8  Modernize SVG/template system
   ↓
PHASE 9  Modernize Vue UI
   ↓
PHASE 10 Modernize build system
   ↓
PHASE 11 Remove legacy code/dependencies
   ↓
PHASE 12 Final validation for physical Stardome use
```

This order is intentional.

---

# 7. PHASE 0 — Freeze the current system

Before updating anything:

## Record

- current commit
- current Node/npm environment
- current dependency versions
- current generated SVGs
- current 3D preview behavior
- current supported polyhedra
- current constellation selections

Create a directory such as:

```text
tests/
fixtures/
```

and save representative outputs.

### Minimum baseline cases

Test at least:

1. one simple polyhedron;
2. truncated icosahedron;
3. isodistant truncated icosahedron;
4. bright stars;
5. magnitude-filtered stars;
6. asterism lines crossing face boundaries;
7. stars near polygon boundaries;
8. SVG with tabs.

The goal is to have something to compare against after every major migration.

---

# 8. PHASE 1 — Correct obvious errors

Before modernizing dependencies, fix errors that are clearly wrong.

Use the separate **Errors & Logical Issues** document as the checklist.

Examples include:

- broken `Circumscribed Radius` option;
- padding that does not actually affect output;
- incorrect property casing such as `.Length` vs `.length`;
- stale/incorrect UI state;
- obviously incorrect conditions;
- dead configuration;
- known runtime errors.

### Rule

Do not mix a bug fix with a large dependency migration if it can be avoided.

Make each correction a separate commit.

This makes it possible to distinguish:

```text
old behavior
vs.
bug fix
vs.
dependency migration
```

---

# 9. PHASE 2 — Build the test foundation

This is the most important phase before touching complex geometry.

The current repository effectively has no useful test suite.

Replace:

```text
npm test
→ intentional failure
```

with actual tests.

## Start with mathematical unit tests

Test:

### Coordinates

```text
RA/Dec → Vector3
```

### Polyhedron

```text
vertices
faces
edges
face planes
```

### Topology

```text
face → neighboring faces
edge → shared face
```

### Projection

```text
vector → face + point
```

### Lines

```text
line → face segments
```

### Unfolding

```text
3D face relationship → 2D relationship
```

### Scaling

```text
requested physical size → correct output dimensions
```

### SVG

```text
same input → deterministic output
```

Do not attempt a full end-to-end test suite immediately.

Start with the mathematical foundations.

---

# 10. PHASE 3 — Modernize the basic data layer

Work from the simplest code upward.

Recommended order:

```text
assets/*.json
      ↓
database.js
      ↓
catalogs.js
```

Tasks:

- clean up data loading;
- remove unused catalog paths;
- replace obsolete fetch/polyfill assumptions;
- simplify asynchronous code;
- clarify star/asterism data structures;
- document the catalog schema;
- add validation for malformed data;
- replace suspicious filtering logic;
- make magnitude filtering explicit and testable.

Do not change the astronomical meaning of the data while doing this.

---

# 11. PHASE 4 — Modernize geometry primitives

Next work on the lowest-level geometry.

Recommended order:

```text
extensions/
      ↓
geometry/truncated-icosahedron.js
geometry/isodistant-ti.js
shapes/
```

The main objective is to stop the application from depending on obsolete Three.js geometry classes.

### Current problem

The project relies on:

```text
Geometry
Face3
Vector3 prototype extensions
Line3 prototype extensions
CurvePath prototype extensions
CubicBezierCurve3 prototype extensions
```

### Desired architecture

Instead of modifying library prototypes:

```text
THREE.Vector3.prototype.someMethod = ...
```

prefer project-owned functions/classes:

```text
geometry/
  vector-utils.js
  line-utils.js
  curve-utils.js
  polygon.js
  edge.js
```

This creates a clear boundary between:

```text
Three.js
```

and:

```text
Stardome geometry
```

This is the key preparation for the eventual Three.js upgrade.

---

# 12. PHASE 5 — Modernize topology

Once geometry primitives are stable:

```text
topology/polygons.js
topology/edges.js
topology/Topology.js
```

Update these next.

The target should be a library-independent topology model.

The topology system should know:

```text
faces
edges
vertices
neighbors
planes
dihedral angles
```

without depending on deprecated Three.js mesh classes.

### Critical test

For the truncated icosahedron:

```text
12 pentagons
20 hexagons
60 vertices
90 edges
32 faces
```

For the isodistant version, also test the equal-distance condition.

---

# 13. PHASE 6 — Modernize projection

Only after topology is stable:

```text
projections/vector.js
projections/line.js
projections/curve.js
projections/path.js
catalogs.js
```

Update:

```text
star → face
asterism → face segments
curve → face pieces
```

### Important

This phase must preserve the mathematical behavior.

Do not combine:

```text
projection redesign
+
Three.js upgrade
+
Vue migration
```

in one step.

---

# 14. PHASE 7 — Modernize unfolding

Next:

```text
geometry/nets.js
geometry/hierarchical-mesh.js
```

The long-term objective is to separate:

```text
unfolding mathematics
```

from:

```text
Three.js scene graph
```

The net itself should be represented as data:

```text
face A
 ├── face B
 ├── face C
 └── face D
```

with explicit transforms.

Then Three.js can merely visualize that data.

This makes the physical template generator independent of the renderer.

---

# 15. PHASE 8 — Modernize the SVG/template pipeline

Next:

```text
template/index.js
template/hull.js
template/scaling.js
template/tabs.js
template/directives.js
template/svg.js
```

This stage should preserve:

- physical dimensions;
- face placement;
- tabs;
- overlaps;
- star paths;
- asterism paths.

### Important

The SVG generator should eventually be usable **without Three.js**.

Ideally:

```text
geometry model
     ↓
2D template model
     ↓
SVG
```

Three.js should not be required merely to generate the final physical SVG.

---

# 16. PHASE 9 — Modernize the Vue application

Only after the computational core is stable.

Current:

```text
Vue 2.1.6
vue-async-computed
```

Target:

```text
Vue 3
Composition API where useful
```

The Vue layer should primarily:

- collect user input;
- display status;
- request projections;
- display the preview;
- initiate SVG generation.

The computational geometry should remain outside Vue.

### Migration path

```text
Vue 2.1
   ↓
Vue 2.7
   ↓
@vue/compat
   ↓
fix compatibility warnings
   ↓
Vue 3
   ↓
remove @vue/compat
```

This follows the migration approach documented by Vue. citeturn0search1turn0search2

---

# 17. PHASE 10 — Modernize the build system

After the source code is modern enough:

## Current

```text
Webpack 2
Babel 6
old loaders
```

## Preferred

```text
Vite
Vue 3 plugin
modern ES modules
```

A modern Vite/Vue setup currently uses Vite 8.x and `@vitejs/plugin-vue` 6.x. citeturn4search14turn4search5

### Remove/rework

- `webpack.config.js`
- CommonsChunkPlugin
- extract-text-webpack-plugin
- file-loader
- worker-loader
- old Babel configuration
- old webpack dev-server configuration

Only remove them **after their functionality has been replaced and tested**.

---

# 18. PHASE 11 — Remove legacy code

Only now should the team delete obsolete code.

Candidates include:

- old SVG implementation;
- unused dependencies;
- unused catalog assets;
- old Three.js compatibility code;
- dead Webpack configuration;
- unused React ESLint plugin;
- old polyfills;
- duplicate utilities.

### Rule

Never delete something merely because it looks old.

First prove:

```text
No imports
+
No runtime use
+
No build use
+
No required historical reference
```

Then remove it.

---

# 19. PHASE 12 — Stardome validation

The final stage is not just:

```text
npm run build
```

The software must be validated against the **physical Stardome requirements**.

Check:

### Geometry

- correct panel dimensions;
- correct face distances;
- correct edge lengths;
- correct dihedral angles.

### Projection

- known stars land correctly;
- constellation lines remain continuous;
- boundary-crossing objects behave correctly.

### Template

- net does not unexpectedly overlap;
- tabs are usable;
- physical dimensions are correct;
- SVG scale is correct.

### Physical test

Produce at least one physical prototype.

Then compare:

```text
software geometry
      ↕
physical assembly
```

before declaring the new pipeline production-ready.

---

# 20. Suggested Git strategy

Do not update the project in one giant branch.

Use a sequence such as:

```text
main
 │
 ├── baseline-tests
 │
 ├── fix-obvious-errors
 │
 ├── modernize-data-layer
 │
 ├── geometry-abstraction
 │
 ├── topology-modernization
 │
 ├── projection-modernization
 │
 ├── unfolding-modernization
 │
 ├── svg-modernization
 │
 ├── vue3-migration
 │
 └── build-modernization
```

Merge each stage only when its tests pass.

---

# 21. What NOT to update together

Avoid these combinations:

### Bad

```text
Three.js
+
Vue
+
Webpack
+
projection mathematics
```

all at once.

### Better

```text
geometry abstraction
→ test
→ Three.js migration
→ test
```

then:

```text
Vue migration
→ test
```

then:

```text
build migration
→ test
```

---

# 22. Dependency migration table — final target

A reasonable long-term target is:

| Area | Current | Target |
|---|---|---|
| Node | very old/unspecified | **24 LTS** |
| Three.js | 0.88.0 | **0.186.0** |
| Vue | 2.1.6 | **3.5.42** |
| Build | Webpack 2 | **Vite 8.3.x** |
| Babel | Babel 6 | **Babel 8 / or minimize Babel usage** |
| Bezier.js | 2.2.3 | **6.1.4** |
| Lodash | 4.17.4 | **4.18.1** |
| Sift | 3.2.6 | **17.1.3** |
| EventEmitter3 | 2.0.3 | **5.0.4** |
| debounce | 1.0.0 | **3.0.0** |
| parse-svg-path | 0.1.2 | **0.2.0** |
| vue-async-computed | 3.1.2 | **4.0.1 or remove** |
| convexhull-js | 1.0.0 | **replace/remove if practical** |
| threestyle | 0.2.1 | **replace/remove** |
| async | 2.4.0 | **3.2.6 or remove if unused** |
| fetch polyfill | whatwg-fetch 2.0.1 | **3.6.20 or remove for modern browsers** |

Some packages should **not** be upgraded in isolation because they are being replaced by the new architecture.

---

# 23. The most important migration boundary

The project should eventually look conceptually like:

```text
                ┌─────────────────────┐
                │     Vue 3 UI        │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Application / State │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Stardome Core       │
                │                     │
                │ astronomy           │
                │ projection          │
                │ geometry            │
                │ topology            │
                │ unfolding           │
                │ template            │
                └───────┬─────┬───────┘
                        │     │
              ┌─────────┘     └─────────┐
              ▼                         ▼
        Three.js preview             SVG output
```

The **Stardome Core** should eventually be as independent as possible from Vue and Three.js.

That is the most valuable architectural improvement we can make.

---

# 24. Priority order for the actual team

If resources are limited, follow this exact order:

### Priority 1
Make the existing application reproducibly build and run.

### Priority 2
Fix confirmed errors.

### Priority 3
Create mathematical regression tests.

### Priority 4
Separate Stardome geometry from Three.js.

### Priority 5
Modernize the geometry/topology layer.

### Priority 6
Upgrade Three.js.

### Priority 7
Modernize projection/unfolding/template code.

### Priority 8
Migrate Vue.

### Priority 9
Move from Webpack to Vite.

### Priority 10
Clean up/remove legacy code.

---

# 25. Definition of "updated"

The project should **not** be considered updated merely because:

```text
package.json contains recent versions
```

It is updated when:

- dependencies are current and supported;
- the project builds on a supported Node LTS;
- tests cover the mathematical core;
- geometry behavior is verified;
- projection behavior is verified;
- SVG output is verified;
- the Vue UI works;
- the 3D preview works;
- the physical template works;
- known errors are fixed;
- obsolete code has been removed;
- the team understands the resulting architecture.

---

# 26. Final roadmap

```text
CURRENT REPOSITORY
       │
       ▼
0. BASELINE
   freeze working output
       │
       ▼
1. ERRORS
   fix confirmed bugs
       │
       ▼
2. TESTS
   protect mathematical behavior
       │
       ▼
3. DATA
   clean database/catalog layer
       │
       ▼
4. GEOMETRY
   remove dependence on old Three.js geometry APIs
       │
       ▼
5. TOPOLOGY
   modern face/edge model
       │
       ▼
6. PROJECTION
   modern star/asterism projection
       │
       ▼
7. UNFOLDING
   modern net + fold model
       │
       ▼
8. SVG
   modern physical-template generator
       │
       ▼
9. VUE
   Vue 2 → Vue 3
       │
       ▼
10. BUILD
    Webpack/Babel → Vite/modern tooling
       │
       ▼
11. CLEANUP
    remove legacy code/dependencies
       │
       ▼
12. PHYSICAL VALIDATION
    test on actual Stardome
       │
       ▼
UPDATED STARDOME SOFTWARE
```

## The central rule

**Never modernize a higher layer while the lower layer it depends on is still unstable.**

In particular:

```text
Do not upgrade Vue to solve geometry problems.
Do not upgrade Three.js before understanding the geometry API.
Do not rewrite projection while changing topology.
Do not trust SVG output until the geometry is tested.
```

The modernization should proceed **from mathematical foundations upward**, with the UI and build system coming later.

---

## Sources for current dependency/platform targets

- Three.js current npm release: 0.186.0. citeturn0search4
- Vue current npm release: 3.5.42. citeturn2search8
- Vue 3 migration/compatibility guidance. citeturn0search0turn0search1
- Vue's current recommendation of Vite for Vue 3 tooling. citeturn0search13
- Vite current npm release: 8.3.0. citeturn4search14
- `@vitejs/plugin-vue` current release: 6.0.8. citeturn4search5
- Node.js 24 LTS / current release status. citeturn4search0turn4search11
- Webpack current release and Webpack 5 line. citeturn1search5turn1search15
- Babel current core/preset-env releases. citeturn3search5turn3search1
- Current Bezier.js, Lodash, Sift, EventEmitter3, debounce, and parse-svg-path releases. citeturn1search7turn1search4turn1search8turn1search3turn1search16turn1search1
- Deprecated/replacement guidance for the old CSS extraction and Babel packages. citeturn2search10turn2search0
