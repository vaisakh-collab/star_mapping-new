# Stardome Star-Mapping Software
## Team Quick Reference

This is the **short working documentation** for the Stardome team.

A more detailed technical document exists as a fallback. Use that document when you need implementation-level details, mathematical explanations, or file-by-file analysis.

---

# 1. What does the software do?

The software takes **astronomical star data** and turns it into a **flat SVG template for a polyhedral Stardome projection**.

The basic pipeline is:

```text
Star data
   ↓
RA / Dec
   ↓
3D position on celestial sphere
   ↓
Projection onto polyhedron
   ↓
Faces / edges / asterisms
   ↓
Unfold into a flat net
   ↓
Add tabs and clipping
   ↓
SVG template
```

The 3D preview shows the same geometry before it is flattened.

---

# 2. The five main parts

Think of the software as five layers.

### 1. Data

**Main files:** `database.js`, `assets/*.json`

Contains:

- stars
- magnitudes
- asterisms/constellation connections

---

### 2. Projection

**Main files:** `catalogs.js`, `projections/`

Converts astronomical coordinates into geometry.

```text
RA + Dec
   ↓
3D direction
   ↓
point on a polyhedron face
```

This is where the star/asterism data becomes spatial data.

---

### 3. Polyhedron

**Main files:** `geometry/`, `topology/`

Defines and understands the physical shape.

It knows:

- faces
- edges
- neighboring faces
- face geometry
- dihedral angles

The truncated icosahedron and the custom **isodistant truncated icosahedron** are especially important to this project.

---

### 4. Unfolding

**Main files:** `geometry/nets.js`, `geometry/hierarchical-mesh.js`

Takes the 3D polyhedron and unfolds it into a flat net.

```text
       3D
      /       /____       ↓
    _________
   |   |   |
   |___|___|
```

This is what makes the output usable as a physical template.

---

### 5. SVG / Template

**Main files:** `template/`

Produces the final flat SVG.

It handles:

- positioning
- scaling
- tabs
- overlaps
- clipping
- SVG paths

---

# 3. The central file

## `project.js`

If someone asks:

> "Where does everything come together?"

The answer is **`project.js`**.

It connects:

```text
polyhedron
   +
star data
   +
asterisms
   +
topology
   +
unfolding
   +
SVG generation
```

When tracing how a feature travels through the system, start here.

---

# 4. The most important files

You do **not** need to memorize every file.

Start with these:

```text
app.js
project.js
database.js
catalogs.js

projections/
geometry/
topology/
template/
```

Then inspect individual files only when working on that part of the system.

---

# 5. How a star reaches the final SVG

A simplified example:

```text
Sirius
  ↓
catalog data
  ↓
RA / Dec
  ↓
3D vector
  ↓
polyhedron intersection
  ↓
specific face + position
  ↓
unfold that face
  ↓
2D position
  ↓
SVG path
```

If Sirius appears in the wrong place, determine **which step first becomes wrong**.

---

# 6. How constellation lines work

A constellation line is not simply drawn directly on the SVG.

It goes through:

```text
Star A + Star B
       ↓
3D line
       ↓
polyhedron
       ↓
split at face boundaries
       ↓
unfold
       ↓
SVG
```

This is why the projection code is more complicated than an ordinary star-map application.

---

# 7. The geometry is critical

The polyhedron is not just a visual model.

Changing its geometry can affect:

- star positions
- constellation lines
- face boundaries
- unfolding
- tabs
- SVG dimensions

For the Stardome work, **geometry changes must be treated as high-impact changes**.

---

# 8. The isodistant truncated icosahedron

The project contains a custom version of the truncated icosahedron.

Its important property is:

```text
pentagon face distance from center
             =
hexagon face distance from center
```

This is one of the project's important Stardome-specific features.

Do not remove or replace it without understanding why it exists.

---

# 9. Three.js is old

The project uses a very old Three.js architecture.

It also adds custom methods to Three.js geometry classes.

Therefore:

> Do not simply change the Three.js version to the latest version.

A Three.js upgrade is a **migration task** and should be done only after the geometry behavior is protected by tests.

---

# 10. What is safe to change first?

Generally lower risk:

```text
UI
documentation
catalog selection
build tooling
data loading
obvious bugs
```

Higher risk:

```text
polyhedron geometry
topology
projection mathematics
unfolding
curve intersection
clipping
tabs
Three.js geometry layer
```

---

# 11. How we should work on the project

The recommended approach is:

```text
Understand existing behavior
          ↓
Make a small change
          ↓
Test
          ↓
Compare output
          ↓
Keep the change
          ↓
Move to the next part
```

Avoid:

```text
Old code
   ↓
rewrite everything
```

The existing geometry contains a lot of specialized logic that would be difficult to reproduce from scratch.

---

# 12. The debugging rule

When something goes wrong, follow the pipeline:

```text
DATA
 ↓
3D PROJECTION
 ↓
TOPOLOGY
 ↓
UNFOLDING
 ↓
SCALING
 ↓
TABS / CLIPPING
 ↓
SVG
```

Find the **first stage where the result becomes wrong**.

Do not immediately modify the final SVG code.

---

# 13. What every team member should understand

Before making major changes, everyone should know:

- What RA and Dec represent
- How a star becomes a 3D vector
- How that vector hits a polyhedron face
- What a polygon and shared edge are
- What a net/unfolding is
- Why constellation lines may need to be split
- What tabs are for
- Why the SVG is the final physical output

That is enough to start working effectively.

---

# 14. One diagram to remember

```text
                  STAR CATALOG
                       │
                       ▼
                    RA / Dec
                       │
                       ▼
                 3D DIRECTIONS
                       │
                       ▼
              POLYHEDRON PROJECTION
                       │
              ┌────────┴────────┐
              ▼                 ▼
            STARS           ASTERISMS
              │                 │
              └────────┬────────┘
                       ▼
                    TOPOLOGY
                       │
                       ▼
                   UNFOLDING
                       │
                       ▼
                TABS + CLIPPING
                       │
                       ▼
                      SVG
                       │
                       ▼
              PHYSICAL STARDOME
```

---

## Quick rule of thumb

**`catalogs.js` knows about stars.**  
**`projections/` knows where things land.**  
**`topology/` knows how faces connect.**  
**`geometry/` knows the polyhedron and unfolding.**  
**`template/` knows how to turn it into a physical SVG.**  
**`project.js` ties them together.**

For detailed implementation information, refer to the full technical documentation.
