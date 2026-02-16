# Geometry Pattern: Rectangular Plate with Circular Hole

## ⚠️ CRITICAL: Use OCC Kernel for This Geometry

The GEO kernel makes it easy to accidentally create **curved boundaries** instead of straight rectangles. The OCC kernel with boolean operations **guarantees correct geometry**.

---

## COMMON MISTAKE: Splines Create Curves, Not Corners!

```python
# ❌ WRONG - This creates a CURVED boundary, not a 90° corner!
spline = geo.addSpline([p_top_mid, p_corner, p_right_mid])
```

**What happens:** `addSpline([A, B, C])` creates a smooth curve through all 3 points. The corner point B becomes a control point, not a sharp corner.

**Result:** Outer boundary is wavy/curved instead of rectangular:
```
Expected:          Actual (with spline):
┌──────────┐       ╭──────────╮
│          │       │          │
│    ○     │       │    ○     │
│          │       │          │
└──────────┘       ╰──────────╯
```

---

## ✅ CORRECT APPROACH: OCC Boolean Operations

Use the OCC kernel to create a rectangle, create a disk, and subtract:

```python
import gmsh
import meshio
import numpy as np

gmsh.initialize()
gmsh.model.add("plate_with_hole")

# USE OCC KERNEL - guarantees straight boundaries
occ = gmsh.model.occ

# Parameters
Lx, Ly = 200, 100  # plate dimensions (mm)
R = 20             # hole radius (mm)
cx, cy = 0, 0      # hole center

# Create rectangle (returns surface tag)
plate = occ.addRectangle(-Lx/2, -Ly/2, 0, Lx, Ly)

# Create circular hole
hole = occ.addDisk(cx, cy, 0, R, R)

# Boolean subtraction: plate - hole
occ.cut([(2, plate)], [(2, hole)])
occ.synchronize()

# Get the resulting surface
surfs = gmsh.model.getEntities(2)
surf_tag = surfs[0][1]
```

**Why this works:** 
- `occ.addRectangle()` creates a true rectangle with straight edges
- `occ.addDisk()` creates a true circle
- `occ.cut()` performs exact boolean subtraction
- No manual curve construction = no mistakes

---

## Mesh Refinement Near Hole

```python
# Identify hole boundary curves (they have small center-of-mass distance)
boundaries = gmsh.model.getBoundary([(2, surf_tag)], oriented=False)
hole_curves = []
outer_curves = []

for dim, tag in boundaries:
    com = gmsh.model.occ.getCenterOfMass(dim, abs(tag))
    dist = np.sqrt(com[0]**2 + com[1]**2)
    if dist < R * 1.5:
        hole_curves.append(abs(tag))
    else:
        outer_curves.append(abs(tag))

# Distance field from hole
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "CurvesList", hole_curves)

# Threshold field: fine near hole, coarse far away
gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", 2)    # fine size near hole
gmsh.model.mesh.field.setNumber(2, "SizeMax", 10)   # coarse size far away
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", R * 3)
gmsh.model.mesh.field.setAsBackgroundMesh(2)
```

---

## Physical Groups for Boundary Conditions

```python
# Classify boundaries by position
left_curves, right_curves, top_curves, bottom_curves = [], [], [], []

for tag in outer_curves:
    com = gmsh.model.occ.getCenterOfMass(1, tag)
    if abs(com[0] - (-Lx/2)) < 1:
        left_curves.append(tag)
    elif abs(com[0] - (Lx/2)) < 1:
        right_curves.append(tag)
    elif abs(com[1] - (Ly/2)) < 1:
        top_curves.append(tag)
    elif abs(com[1] - (-Ly/2)) < 1:
        bottom_curves.append(tag)

# Assign physical groups
gmsh.model.addPhysicalGroup(1, left_curves, 1)
gmsh.model.setPhysicalName(1, 1, "left")
gmsh.model.addPhysicalGroup(1, right_curves, 2)
gmsh.model.setPhysicalName(1, 2, "right")
gmsh.model.addPhysicalGroup(1, top_curves, 3)
gmsh.model.setPhysicalName(1, 3, "top")
gmsh.model.addPhysicalGroup(1, bottom_curves, 4)
gmsh.model.setPhysicalName(1, 4, "bottom")
gmsh.model.addPhysicalGroup(1, hole_curves, 5)
gmsh.model.setPhysicalName(1, 5, "hole")
gmsh.model.addPhysicalGroup(2, [surf_tag], 100)
gmsh.model.setPhysicalName(2, 100, "domain")
```

---

## Generate Quad Mesh

```python
# Algorithm 8 (Frontal-Delaunay for quads) + Recombination
gmsh.option.setNumber("Mesh.Algorithm", 8)
gmsh.option.setNumber("Mesh.RecombinationAlgorithm", 2)  # simple
gmsh.option.setNumber("Mesh.RecombineAll", 1)
gmsh.option.setNumber("Mesh.Smoothing", 100)

gmsh.model.mesh.generate(2)
gmsh.write("mesh/mesh.msh")
gmsh.finalize()
```

---

## Export for FEniCS/DOLFINx

Keep the .msh file - DOLFINx reads it directly via gmshio. No XDMF conversion needed.
```python
# Write marker map for solver
marker_map = {
    "left_edge": 1,
    "right_edge": 2,
    "top_edge": 3,
    "bottom_edge": 4,
    "hole_boundary": 5,
    "domain": 100
}

import json
with open("mesh/marker_map.json", "w") as f:
    json.dump(marker_map, f, indent=2)

# Mesh file is already written by gmsh.write("mesh/mesh.msh")
# Solver reads it with: gmshio.read_from_msh("mesh/mesh.msh", MPI.COMM_WORLD, gdim=2)
```

---

## mesh_quality.json

```json
{
  "num_elements": <int>,
  "element_type": "quad",
  "meshing_method": "occ_boolean_algorithm8_recombine",
  "min_quality": <float>,
  "avg_quality": <float>,
  "mesh_ready_for_solver": true,
  "fenics_compatible": true
}
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Use OCC kernel (`gmsh.model.occ`) |
| 2 | Create rectangle with `occ.addRectangle()` |
| 3 | Create hole with `occ.addDisk()` |
| 4 | Subtract with `occ.cut()` |
| 5 | Set up refinement near hole |
| 6 | Use Algorithm 8 + RecombineAll for quads |
| 7 | Export with meshio (2D points!) |

**DO NOT use `geo.addSpline()` for outer boundaries!**
