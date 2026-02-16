# Geometry Pattern: Plate with Multiple Holes

## Block Layout: 4 Blocks per Hole

```
Two holes example (8 blocks):

    ┌─────┬─────┬─────┬─────┐
    │  1  │  2  │  5  │  6  │
    ├──○──┼─────┼──○──┼─────┤
    │  4  │  3  │  8  │  7  │
    └─────┴─────┴─────┴─────┘
       hole1       hole2
```

**Rule:** Each circular hole adds 4 blocks minimum.

## Complexity Warning

Multiple holes create **constraint propagation chains**:
- Node counts on shared edges between hole regions must match
- Holes at different y-positions make this very complex
- Consider using **fallback (Algorithm 8)** for 2+ holes

## Implementation: 2 Holes (Horizontal Arrangement)

```python
#filename.py
import gmsh
import numpy as np

gmsh.initialize()
gmsh.model.add("two_holes")
geo = gmsh.model.geo

# Parameters
Lx, Ly = 300, 100
R = 15  # hole radius
hole1_x, hole1_y = -75, 0   # left hole center
hole2_x, hole2_y = 75, 0    # right hole center

# Outer corners
p_bl = geo.addPoint(-Lx/2, -Ly/2, 0)
p_br = geo.addPoint(Lx/2, -Ly/2, 0)
p_tr = geo.addPoint(Lx/2, Ly/2, 0)
p_tl = geo.addPoint(-Lx/2, Ly/2, 0)

# Boundary partition points (where hole radials meet boundary)
p_top_h1 = geo.addPoint(hole1_x, Ly/2, 0)
p_bot_h1 = geo.addPoint(hole1_x, -Ly/2, 0)
p_top_h2 = geo.addPoint(hole2_x, Ly/2, 0)
p_bot_h2 = geo.addPoint(hole2_x, -Ly/2, 0)

# Midpoint between holes (partition)
mid_x = (hole1_x + hole2_x) / 2
p_top_mid = geo.addPoint(mid_x, Ly/2, 0)
p_bot_mid = geo.addPoint(mid_x, -Ly/2, 0)

# Hole 1 cardinal points
p_c1 = geo.addPoint(hole1_x, hole1_y, 0)  # center
p_h1_e = geo.addPoint(hole1_x + R, hole1_y, 0)
p_h1_n = geo.addPoint(hole1_x, hole1_y + R, 0)
p_h1_w = geo.addPoint(hole1_x - R, hole1_y, 0)
p_h1_s = geo.addPoint(hole1_x, hole1_y - R, 0)

# Hole 2 cardinal points
p_c2 = geo.addPoint(hole2_x, hole2_y, 0)
p_h2_e = geo.addPoint(hole2_x + R, hole2_y, 0)
p_h2_n = geo.addPoint(hole2_x, hole2_y + R, 0)
p_h2_w = geo.addPoint(hole2_x - R, hole2_y, 0)
p_h2_s = geo.addPoint(hole2_x, hole2_y - R, 0)

# ... (curves and surfaces would follow similar pattern to single hole)
# This gets VERY complex with many curves and constraints

geo.synchronize()
gmsh.finalize()

print("""
For 2+ holes, the partitioning complexity explodes:
- 8 blocks minimum
- 16+ curves
- Complex node count constraints

RECOMMENDATION: Use Algorithm 8 fallback for multi-hole geometries.
""")
```

## Recommended: Use Fallback for Multiple Holes

```python
#filename.py
import gmsh

gmsh.initialize()
gmsh.model.add("multi_hole")
occ = gmsh.model.occ

# Build with OCC (boolean operations)
plate = occ.addRectangle(-Lx/2, -Ly/2, 0, Lx, Ly)
hole1 = occ.addDisk(hole1_x, hole1_y, 0, R, R)
hole2 = occ.addDisk(hole2_x, hole2_y, 0, R, R)

# Cut holes from plate
occ.cut([(2, plate)], [(2, hole1), (2, hole2)])
occ.synchronize()

# Use Algorithm 8 (reliable for complex geometry)
gmsh.option.setNumber("Mesh.Algorithm", 8)
gmsh.option.setNumber("Mesh.RecombinationAlgorithm", 3)
gmsh.option.setNumber("Mesh.RecombineAll", 1)
gmsh.option.setNumber("Mesh.Smoothing", 100)

# Refine near holes
hole_curves = [...]  # get hole boundary curves
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "CurvesList", hole_curves)
gmsh.model.mesh.field.setNumber(1, "Sampling", 200)

gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", R/5)
gmsh.model.mesh.field.setNumber(2, "SizeMax", Lx/20)
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", R*3)
gmsh.model.mesh.field.setAsBackgroundMesh(2)

gmsh.model.mesh.generate(2)
gmsh.write("mesh/mesh.msh")
gmsh.finalize()
```

## Decision Guide

| Holes | Recommended Approach |
|-------|---------------------|
| 0 | Simple transfinite |
| 1 | 4-block O-grid partitioning |
| 2+ (aligned) | 8+ block partitioning (complex) OR fallback |
| 2+ (non-aligned) | **Use fallback (Algorithm 8)** |

## Why Fallback is Often Better for Multi-Hole

1. **Constraint explosion:** N holes → 4N blocks → complex node matching
2. **Error-prone:** One wrong node count breaks everything
3. **Diminishing returns:** Quality gain vs complexity cost
4. **Algorithm 8 handles it:** Produces good quads automatically
