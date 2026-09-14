# Stardome Star-Mapping Software
## Critical Features, Pipelines, and Team Reference

**Purpose:** This document records the parts of the existing software that the team must understand before modifying, modernizing, or extending it for the physical Stardome projection workflow.

**Repository examined:** `star_mapping-new-main`  
**Scope:** `src/` and its runtime role in the application.

---

# 1. What the software actually does

This application is best understood as a **celestial-coordinate → polyhedral projection → unfolded physical template** engine.

It is not merely a star-map viewer.

The central transformation is:

```text
Astronomical catalog
(RA / Dec / magnitude)
        |
        v
3D direction on celestial sphere
        |
        v
Intersection with selected polyhedron
        |
        v
Polyhedron face / polygon + 3D position
        |
        +----------------------+
        |                      |
        v                      v
     Stars                 Asterism lines
        |                      |
        +----------+-----------+
                   |
                   v
          Polyhedral topology
          + net/unfolding
                   |
                   v
          2D physical template
                   |
                   v
             SVG output
```

The software therefore has three fundamentally different coordinate spaces:

1. **Celestial space** — right ascension and declination.
2. **3D polyhedral space** — points, faces, edges, and folds.
3. **2D template/SVG space** — the flat physical cutting/projection template.

Keeping these spaces conceptually separate is essential when modifying the code.

---

# 2. Critical runtime pipeline

## 2.1 Application/UI layer

### Main files
- `app.js`
- `index.html`
- `components/preview.js`

`app.js` is the main controller for user selections and application state.

It determines things such as:

- selected polyhedron
- magnitude filtering
- selected asterisms/constellations
- net/template options
- projection parameters

It does **not** perform the core geometric projection itself.

Instead it eventually calls the projection pipeline through `project.js`.

### Team rule

When changing the UI, do not assume that changing a displayed option changes geometry directly. Trace:

```text
UI state
 -> computed query
 -> project()
 -> projection/topology
 -> template
```

---

# 3. Astronomical data pipeline

## 3.1 `database.js`

This is the catalog/data-access layer.

It loads star and asterism data from the JSON assets and normalizes them into objects used by the projection code.

Important assets include:

- `assets/asterisms.json`
- `assets/hd_asterisms.json`
- `assets/hd_mag__-2.00-6.50.json`
- `assets/hd.json`

The main magnitude catalog currently provides thousands of stars across approximately the `-2.00` to `6.50` magnitude range.

The asterism data identifies stars connected by constellation/asterism lines.

### Important implementation detail

Catalog loading is cached through promises, so repeated projections do not necessarily re-download the same data.

### Team risks

The repository contains multiple catalog files and not all are necessarily active at runtime.

Before replacing or deleting an asset, trace its imports from `database.js`.

---

# 4. Celestial coordinates → 3D vectors

## 4.1 `catalogs.js`

This is one of the most important files in the entire project.

It combines:

- star catalog data
- asterism definitions
- astronomical coordinate conversion
- star-shape creation
- calls into projection algorithms

A star's right ascension and declination are converted into a normalized Three.js `Vector3`.

Conceptually:

```text
RA + Dec
   |
   v
unit vector from origin
```

That vector represents a **direction toward the star on the celestial sphere**.

### Why this matters

Once the star has become a 3D direction, the rest of the system no longer needs to reason about RA/Dec for the geometric projection.

---

# 5. Celestial sphere → selected polyhedron

## 5.1 `projections/vector.js`

This module finds where a celestial direction intersects the selected polyhedron.

Conceptually:

```text
origin
  |
  | ray along star direction
  v
polyhedron face
```

The result identifies:

```text
polygonId
3D intersection point
```

The software therefore knows not only *where* the star lands in 3D, but **which physical panel/face receives it**.

### Critical invariant

Every projected astronomical object must remain associated with the correct polygon.

If this mapping is wrong, later unfolding cannot fix it.

---

# 6. Star rendering/projection

The application does not simply put a point at the star location.

Depending on the star, it can create a small geometric representation such as:

- four-point star
- circle

The star shape is then transformed and projected onto the polyhedron.

The system also converts an angular size into a geometric size, rather than treating star size as an arbitrary screen pixel value.

### Important consequence

Star appearance is tied to geometry.

Changing:

- star angular size
- polyhedron size
- projection scale

can change the resulting physical SVG.

---

# 7. Asterism/constellation pipeline

Asterisms are represented as connections between catalog stars.

Conceptually:

```text
Star A -------- Star B
```

The line is projected using the 3D locations of the endpoints.

## 7.1 `projections/line.js`

This is a critical geometric algorithm.

It determines how a line on the celestial sphere/polyhedron crosses polygon boundaries.

A single astronomical line may therefore become:

```text
Face A:  A -> X
Face B:  X -> Y
Face C:  Y -> B
```

This segmentation is necessary because the eventual physical template is made of separate polygon panels.

### Team rule

Do not replace this with a simple 2D line-drawing operation unless the topology/clipping consequences are fully understood.

---

# 8. Curves and star-shape clipping

## 8.1 `projections/path.js`
## 8.2 `projections/curve.js`

These handle projected paths/curves, particularly star shapes.

A star shape begins as local geometry and is transformed toward the corresponding celestial direction before being projected.

The code also has to handle geometry crossing polygon boundaries.

This is why the system contains curve splitting and intersection logic.

---

# 9. Custom Three.js extensions

## Directory
`extensions/`

The project extends Three.js classes/prototypes with additional operations.

Examples include custom operations for:

- `Vector3`
- `Face3`
- `Triangle`
- `Line3`
- `CurvePath`
- `CubicBezierCurve3`

Important operations include:

- line/curve intersections
- curve splitting
- conversion helpers
- extraction of line segments

## Why this is critical

The rest of the application relies on these methods.

They are part of the project's **implicit internal geometry API**, even though they are implemented as extensions to old Three.js classes.

### Major modernization warning

The project uses an old Three.js architecture based on APIs such as:

```text
THREE.Geometry
THREE.Face3
```

Modern Three.js versions do not preserve this architecture.

Therefore:

> Upgrading Three.js is a migration project, not a dependency-version bump.

A future team member must first understand/reimplement the geometry API used by these extensions.

---

# 10. Polyhedron geometry

## Directory
`geometry/`

Important files include:

- `truncated-icosahedron.js`
- `isodistant-ti.js`
- `hierarchical-mesh.js`
- `nets.js`

---

# 11. Truncated Icosahedron

## `geometry/truncated-icosahedron.js`

The software generates a truncated icosahedron mathematically using the golden ratio.

The conventional topology has:

```text
12 pentagons
20 hexagons
60 vertices
90 edges
32 faces
```

This geometry is especially important for the Stardome because it resembles the panel structure required by the physical projection concept.

### Team rule

Treat the geometry generator as a mathematical definition, not merely a rendering mesh.

Changing vertex positions can affect:

- face planes
- edge lengths
- dihedral angles
- projected star positions
- unfolding
- SVG dimensions
- physical assembly

---

# 12. Isodistant Truncated Icosahedron

## `geometry/isodistant-ti.js`

This is a particularly important custom feature.

The software constructs a truncated icosahedron and solves for a truncation parameter so that the pentagonal and hexagonal face planes have equal distance from the center.

The implementation searches for the truncation value numerically and then normalizes the geometry.

Conceptually:

```text
distance(center, pentagon planes)
                =
distance(center, hexagon planes)
```

### Why this matters

For a projection system, equal face-plane distance can be physically useful because the projector-to-panel geometry becomes more uniform.

### Important distinction

This is not merely a visual variant of the standard truncated icosahedron.

It is a different geometric constraint and must be treated as such.

---

# 13. Polyhedral topology

## Directory
`topology/`

This layer converts the underlying triangular Three.js mesh into meaningful polygon/edge topology.

This is one of the most critical architectural layers.

The renderer may represent a surface as triangles, but the physical template needs:

```text
pentagons
hexagons
shared edges
neighbor relationships
```

---

# 14. Polygon reconstruction

## `topology/polygons.js`

Triangles belonging to the same plane are grouped into actual polygon faces.

A polygon contains information such as:

- plane
- points
- triangles
- center
- index

It also provides point-in-polygon testing.

### Why this matters

A projected star must be assigned to the correct polygon.

A line must know which polygon segment it belongs to.

The net generator must know which polygon is connected to which.

All of those depend on this layer.

---

# 15. Edge reconstruction and adjacency

## `topology/edges.js`

Edges are reconstructed and linked between neighboring polygons.

An edge carries information about:

- its endpoints
- owning polygon
- neighboring/shared polygon
- direction/vector
- previous/next edge
- geometric line
- dihedral angle

The most important concept is:

```text
Polygon A
    |
    | shared edge
    |
Polygon B
```

The same physical edge is represented from both faces' perspectives.

This adjacency is what makes unfolding possible.

---

# 16. Per-edge dihedral angles

A significant feature of the current repository is that dihedral information is associated with individual edges.

This matters especially for truncated icosahedra.

A simplistic implementation might assume:

```text
one dihedral angle for entire polyhedron
```

but different edge types can have different relationships.

The current topology therefore has the ability to reason about the actual local fold.

### Team rule

Do not replace per-edge dihedral information with a global constant without proving that the target geometry permits it.

---

# 17. Net generation

## `geometry/nets.js`

The net system determines which faces remain connected in the flat template and which edges are treated as cuts.

There are explicit net arrangements for some standard solids and a generic/naive net-generation path for other geometries.

Conceptually:

```text
3D polyhedron
     |
     v
face adjacency graph
     |
     v
choose spanning tree / connections
     |
     v
flat net
```

### Important distinction

A mathematically valid unfolding is not necessarily a good physical template.

A net can:

- overlap itself
- become very wide
- waste material
- create awkward tabs
- be difficult to assemble

Therefore net optimization is a separate future engineering problem.

---

# 18. Hierarchical mesh / unfolding

## `geometry/hierarchical-mesh.js`

This builds a hierarchy of polygon nodes.

The hierarchy represents the folding relationships between faces.

Conceptually:

```text
root face
   |
   +-- adjacent face
          |
          +-- next face
```

Each child can rotate around its shared edge.

This allows the preview to show the net being folded/unfolded.

### Critical relationship

The hierarchical mesh is not just a visualization convenience.

Its transforms are later used by the template-generation stage to obtain the flat 2D positions.

---

# 19. Central orchestration

## `project.js`

This is the central coordinator.

It connects the major subsystems:

```text
selected geometry
      |
      v
Topology
      |
      +----> projected stars
      |
      +----> projected asterisms
      |
      v
Hierarchical mesh
      |
      v
template/SVG
```

If the team needs to understand the whole application quickly, `project.js` is one of the best files to trace.

---

# 20. Preview pipeline

## `components/preview.js`

This creates the interactive Three.js preview.

It is responsible for things such as:

- renderer
- camera
- interaction
- raycasting
- polygon highlighting
- animation
- visualizing the hierarchical mesh

The preview therefore provides a visual verification layer before SVG generation.

### Important limitation

A visually correct 3D preview does **not automatically prove** that the physical SVG is correct.

The SVG pipeline performs additional transformations, scaling, clipping, and tab operations.

---

# 21. Unfolded template → SVG

## Directory
`template/`

Important files include:

- `index.js`
- `hull.js`
- `scaling.js`
- `tabs.js`
- `directives.js`
- `svg.js`

---

# 22. `template/index.js`

This is the main template-generation pipeline.

It takes the unfolded polygon transformations and converts them into a flat representation suitable for SVG.

Conceptually:

```text
3D face transforms
       |
       v
unfolded 2D coordinates
       |
       v
orientation
       |
       v
scaling
       |
       v
tabs / overlaps
       |
       v
SVG elements
```

---

# 23. Orientation and convex hull

## `template/hull.js`

The hull logic determines the outer boundary of the unfolded template.

The system also calculates an orientation transformation so the resulting template has a sensible page orientation.

This is important because the physical template should not be randomly rotated just because of how the root face happened to be selected.

---

# 24. Physical scaling

## `template/scaling.js`

Scaling can be based on physical/geometric quantities such as:

- edge length
- inscribed radius
- circumscribed radius
- template width
- template height

### Critical distinction

There is a difference between:

```text
mathematical polyhedron scale
```

and:

```text
SVG/page scale
```

and:

```text
physical Stardome/projector scale
```

These should not be conflated.

When modifying scaling, document exactly which quantity is being held constant.

---

# 25. Tabs and physical assembly

## `template/tabs.js`

This module generates the tabs used to physically assemble the net.

It considers:

- edge geometry
- tab height
- overlap
- neighboring polygon
- orientation
- usable tab region

The tab system is therefore part of the physical manufacturing pipeline, not merely decoration.

### Critical requirement

Any change to face geometry or net topology can change tab geometry.

Tabs must therefore be regression-tested whenever:

- geometry changes
- net changes
- edge definitions change
- scaling changes

---

# 26. Cross-face clipping

This is one of the most mathematically complex parts of the software.

A star or asterism can cross a polygon boundary.

The system therefore needs to:

1. detect boundary crossing;
2. find the intersection;
3. split the path/curve;
4. assign the correct segment to the correct face;
5. unfold the segment;
6. account for tabs/overlap;
7. output the appropriate SVG path.

For Bézier curves, the custom curve extensions provide operations such as splitting and intersection.

### Team warning

This logic should be considered **high-risk code**.

Do not refactor it aggressively until a regression test suite exists.

---

# 27. SVG serialization

## `template/directives.js`
## `template/svg.js`

These are comparatively simple compared with the geometry engine.

They convert internal geometric objects into SVG primitives such as:

- paths
- polygons
- lines

The key point is that by the time these files execute, the difficult geometric decisions should already have been made.

---

# 28. Important legacy/duplicate code

The repository contains an older SVG-related implementation outside the newer template system.

The active path used by `project.js` goes through the `template` module.

Therefore:

> Do not assume every file in `src/` is part of the current runtime.

Before deleting or modernizing a file, search its imports/usages.

Likewise, multiple astronomical data assets exist; trace actual imports before changing them.

---

# 29. Critical data structures

The team should become comfortable with these concepts.

## Star

Conceptually:

```text
id
rightAscension
declination
magnitude
```

After projection:

```text
polygonId
3D point
projected shape/path
```

## Polygon

Conceptually:

```text
plane
points
triangles
center
edges
```

## Edge

Conceptually:

```text
endpoints
line
direction
owning polygon
shared polygon
next
previous
dihedral
```

## Projected line/path

Conceptually:

```text
source astronomical object
+
3D geometry
+
polygon ownership
+
transformed/unfolded geometry
```

The exact object shapes should be checked before changing APIs, but these relationships are the important conceptual model.

---

# 30. The three coordinate systems team members must not confuse

## A. Celestial coordinates

```text
RA
Dec
```

These describe direction on the celestial sphere.

## B. Polyhedron coordinates

```text
Vector3
Point
Plane
Face
Edge
```

These describe geometry on the 3D projection surface.

## C. Template coordinates

```text
2D x/y
SVG units
physical dimensions
```

These describe the flat sheet/panel template.

### Golden rule

When debugging an incorrect star position, first ask:

> At which coordinate-system transition did the error appear?

Do not immediately modify the final SVG code.

---

# 31. End-to-end example: one star

For a star such as Sirius:

```text
Catalog entry
    |
    v
RA / Dec
    |
    v
vectorFromAngles()
    |
    v
3D unit direction
    |
    v
projectVector()
    |
    v
polyhedron face + intersection point
    |
    v
star shape / angular size
    |
    v
projected path
    |
    v
face topology
    |
    v
unfolding transform
    |
    v
2D SVG coordinates
    |
    v
<path ...>
```

This is the basic star pipeline.

---

# 32. End-to-end example: constellation line

```text
Asterism catalog
    |
    v
star A + star B
    |
    v
3D directions / projected positions
    |
    v
line projection
    |
    v
boundary intersections
    |
    v
segments assigned to faces
    |
    v
unfolding
    |
    v
clipping / tabs
    |
    v
SVG paths
```

---

# 33. What the team should treat as "core intellectual property"

The most valuable/least replaceable parts of the existing implementation are not the Vue UI or Webpack configuration.

They are:

1. celestial coordinate conversion;
2. vector-to-polyhedron projection;
3. polygon/edge topology reconstruction;
4. line intersection and segmentation;
5. Bézier curve intersection/splitting;
6. hierarchical unfolding;
7. net generation;
8. physical scaling;
9. tab generation;
10. cross-face clipping;
11. final SVG transformation.

These are the parts we should preserve and test before attempting major rewrites.

---

# 34. What can be modernized more safely

Lower-risk areas include:

- UI styling;
- Vue components, provided state contracts remain intact;
- build tooling;
- dependency management;
- catalog loading architecture;
- code organization;
- documentation;
- tests;
- logging/debugging;
- SVG serialization, once its input contracts are documented.

Higher-risk areas include:

- Three.js replacement;
- geometry representation;
- topology algorithms;
- projection mathematics;
- unfolding;
- curve intersection;
- clipping;
- tabs.

---

# 35. Recommended modernization strategy

Do this incrementally.

## Phase 1 — Freeze the baseline

Before modifying geometry:

```text
Current repository
      |
      v
known working build
      |
      v
sample generated SVGs
      |
      v
sample screenshots/output
```

Keep representative outputs for regression comparison.

## Phase 2 — Add tests around mathematics

Test:

- RA/Dec → vector
- point → polygon
- line → polygon segments
- polygon adjacency
- dihedral angles
- unfolding transforms
- scaling
- tab generation

## Phase 3 — Fix obvious defects

Only after establishing the baseline, address clearly identifiable bugs and stale code.

## Phase 4 — Improve the Stardome-specific geometry

This is where the isodistant truncated icosahedron and physical projector constraints can be developed properly.

## Phase 5 — Modernize infrastructure

Upgrade build tooling and dependencies while preserving the mathematical behavior.

## Phase 6 — Replace obsolete Three.js abstractions

Only after the internal geometry contracts are understood and tested.

---

# 36. Recommended debugging method

When something is wrong in the final output, debug from left to right:

```text
DATA
  ↓
COORDINATES
  ↓
3D PROJECTION
  ↓
TOPOLOGY
  ↓
UNFOLDING
  ↓
SCALING
  ↓
TABS/CLIPPING
  ↓
SVG
```

For example:

### Star appears on wrong face

Check:

```text
RA/Dec
→ vector
→ polygon intersection
```

Do not start with SVG.

### Star is on correct face but wrong location

Check:

```text
vector projection
→ face-local coordinates
→ unfolding transform
```

### Star is correct until it reaches an edge

Check:

```text
curve intersection
→ splitting
→ cross-face clipping
```

### Whole template is the wrong size

Check:

```text
scaling.js
```

### Faces fold incorrectly

Check:

```text
edges.js
→ dihedral
→ hierarchical-mesh.js
```

### Tabs are wrong

Check:

```text
net
→ shared edges
→ tabs.js
```

---

# 37. Critical invariants

These should eventually become automated tests.

### Geometry

- expected vertex count
- expected face count
- expected edge count
- face-planarity
- correct adjacency

### Truncated icosahedron

```text
12 pentagons
20 hexagons
60 vertices
90 edges
32 faces
```

### Isodistant variant

Verify:

```text
pentagon plane distance
≈
hexagon plane distance
```

within a defined numerical tolerance.

### Projection

A known star should consistently map to the same face/position.

### Topology

Every shared edge must connect exactly two faces.

### Unfolding

Connected faces must share the expected 2D edge after unfolding.

### SVG

SVG output must be deterministic for identical input.

---

# 38. Team mental model

Every developer joining this project should be able to answer these questions before making major changes:

### Astronomy

- What are RA and Dec?
- How are they converted into a 3D direction?
- What does magnitude represent here?

### Geometry

- What is a polygon versus a triangle in this project?
- How are edges shared?
- What is a dihedral angle?
- Why can't we assume one dihedral angle for every truncated-icosahedron edge?

### Projection

- How does a celestial direction become a point on a face?
- What happens when an asterism crosses a face boundary?
- Why do star shapes need curve intersection/splitting?

### Unfolding

- What is the net?
- What determines which edges are cuts?
- How is a child face rotated around a shared edge?

### Physical template

- What does scaling mean?
- What are tabs and overlaps?
- Why can a visually correct preview still produce an incorrect physical template?

### Software architecture

- Which files are active?
- Which files are legacy?
- Which Three.js APIs are custom extensions?
- Where does data become geometry?
- Where does geometry become SVG?

---

# 39. The most important files to study first

If the team does not have time to read everything, use this order:

```text
1. app.js
2. project.js
3. database.js
4. catalogs.js

5. topology/polygons.js
6. topology/edges.js

7. projections/vector.js
8. projections/line.js
9. projections/path.js

10. geometry/truncated-icosahedron.js
11. geometry/isodistant-ti.js
12. geometry/hierarchical-mesh.js
13. geometry/nets.js

14. template/index.js
15. template/tabs.js
16. template/scaling.js

17. extensions/cubic-bezier-curve.js
18. extensions/curve-path.js
19. components/preview.js
```

That reading order follows the actual data flow rather than the directory structure.

---

# 40. One-page architecture reference

```text
┌──────────────────────────────────────────────┐
│                 UI / Vue                     │
│ app.js / components/preview.js / index.html │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│                 DATA                         │
│ database.js                                   │
│ stars + asterisms + magnitude filtering       │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│          CELESTIAL COORDINATES               │
│ catalogs.js                                   │
│ RA/Dec → 3D unit vectors                     │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│              PROJECTION                      │
│ projections/vector.js                        │
│ projections/line.js                          │
│ projections/path.js                          │
│ projections/curve.js                         │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│             TOPOLOGY                         │
│ polygons.js / edges.js                       │
│ faces + adjacency + dihedral angles           │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│          NET / UNFOLDING                     │
│ nets.js / hierarchical-mesh.js               │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│          PHYSICAL TEMPLATE                   │
│ template/index.js                            │
│ hull.js / scaling.js / tabs.js               │
│ clipping + overlaps                          │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│                    SVG                       │
│ directives.js / svg.js                       │
└──────────────────────────────────────────────┘
```

---

# 41. Bottom line for the Stardome team

**Do not think of this as an old frontend project that needs to be rewritten.**

Think of it as an old **geometry engine with a frontend attached to it**.

The frontend can eventually be replaced.

The build system can eventually be replaced.

Old Three.js can eventually be replaced.

But before doing that, the team needs to preserve and understand the mathematical contracts:

```text
celestial coordinates
        ↓
3D directions
        ↓
polyhedron intersection
        ↓
face topology
        ↓
cross-face projection
        ↓
unfolding
        ↓
physical scaling
        ↓
tabs / clipping
        ↓
SVG
```

That pipeline is the heart of the software.

The safest long-term strategy is therefore **incremental modernization with regression tests**, not a clean-slate rewrite.

---

# Appendix A — `src/` inventory

The uploaded repository contains the following `src` files/directories examined for this documentation:

- `app.js` (3,987 bytes)
- `assets/asterisms.json` (12,818 bytes)
- `assets/hd.json` (79,295 bytes)
- `assets/hd_asterisms.json` (81,753 bytes)
- `assets/hd_mag__-2.00-6.50.json` (1,077,033 bytes)
- `assets/materials.three.css` (669 bytes)
- `assets/style.css` (1,349 bytes)
- `catalogs.js` (3,989 bytes)
- `components/preview.js` (3,894 bytes)
- `database.js` (3,882 bytes)
- `extensions/cubic-bezier-curve.js` (2,514 bytes)
- `extensions/curve-path.js` (3,544 bytes)
- `extensions/face.js` (120 bytes)
- `extensions/index.js` (122 bytes)
- `extensions/line.js` (1,002 bytes)
- `extensions/triangle.js` (126 bytes)
- `extensions/vector.js` (108 bytes)
- `geometry/hierarchical-mesh.js` (1,611 bytes)
- `geometry/isodistant-ti.js` (9,036 bytes)
- `geometry/nets.js` (2,294 bytes)
- `geometry/truncated-icosahedron.js` (6,704 bytes)
- `index.html` (2,949 bytes)
- `project.js` (6,279 bytes)
- `projections/async.js` (1,496 bytes)
- `projections/curve.js` (3,319 bytes)
- `projections/index.js` (170 bytes)
- `projections/line.js` (3,493 bytes)
- `projections/path.js` (1,315 bytes)
- `projections/vector.js` (437 bytes)
- `shapes/circle.js` (648 bytes)
- `shapes/star.js` (812 bytes)
- `svg.js` (17,632 bytes)
- `template/directives.js` (809 bytes)
- `template/hull.js` (1,134 bytes)
- `template/index.js` (15,712 bytes)
- `template/scaling.js` (1,483 bytes)
- `template/svg.js` (850 bytes)
- `template/tabs.js` (3,515 bytes)
- `topology/Topology.js` (2,604 bytes)
- `topology/edges.js` (1,527 bytes)
- `topology/index.js` (59 bytes)
- `topology/polygons.js` (2,126 bytes)

---

# Appendix B — terminology

| Term | Meaning in this project |
|---|---|
| RA | Right Ascension; astronomical angular coordinate |
| Dec | Declination; astronomical angular coordinate |
| Vector3 | Three-dimensional direction/point representation |
| Polygon | Actual polyhedron face, potentially made from multiple triangles |
| Edge | Boundary shared by polygon faces |
| Dihedral angle | Angle relationship between adjacent face planes |
| Topology | Connectivity/adjacency structure of faces and edges |
| Net | Flat arrangement of connected polyhedron faces |
| Unfolding | Transforming the 3D connected faces into a flat template |
| Asterism | Defined set of stars connected by lines |
| Tab | Extra physical material used to join panels |
| SVG | Final vector template representation |

---

**Operational recommendation:** Treat this document as the team's architectural map. Before changing any core geometry code, identify which stage of the pipeline the change belongs to and establish a regression test for the current behavior.
