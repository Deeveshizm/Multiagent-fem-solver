# Geometry Pattern: Simple Rectangle (No Holes)

## The Easiest Case: 1 Block

```
    ┌─────────────────────┐
    │                     │
    │                     │
    │      1 BLOCK        │
    │                     │
    │                     │
    └─────────────────────┘
```

**Blocks needed:** 1

## Implementation

```python
import gmsh

gmsh.initialize()
gmsh.model.add("rectangle")
geo = gmsh.model.geo

# Parameters
Lx, Ly = 200, 100  # dimensions
x0, y0 = -Lx/2, -Ly/2  # bottom-left corner

# 4 corner points
p1 = geo.addPoint(x0, y0, 0)        # bottom-left
p2 = geo.addPoint(x0 + Lx, y0, 0)   # bottom-right
p3 = geo.addPoint(x0 + Lx, y0 + Ly, 0)  # top-right
p4 = geo.addPoint(x0, y0 + Ly, 0)   # top-left

# 4 edges
c_bottom = geo.addLine(p1, p2)
c_right = geo.addLine(p2, p3)
c_top = geo.addLine(p3, p4)
c_left = geo.addLine(p4, p1)

# 1 surface (exactly 4 edges ✓)
loop = geo.addCurveLoop([c_bottom, c_right, c_top, c_left])
surf = geo.addPlaneSurface([loop])

geo.synchronize()

# Physical groups for BCs
gmsh.model.addPhysicalGroup(1, [c_left], 1)
gmsh.model.setPhysicalName(1, 1, "left")
gmsh.model.addPhysicalGroup(1, [c_right], 2)
gmsh.model.setPhysicalName(1, 2, "right")
gmsh.model.addPhysicalGroup(1, [c_top], 3)
gmsh.model.setPhysicalName(1, 3, "top")
gmsh.model.addPhysicalGroup(1, [c_bottom], 4)
gmsh.model.setPhysicalName(1, 4, "bottom")
gmsh.model.addPhysicalGroup(2, [surf], 100)

# Node counts (opposites must match!)
Nx = 40  # nodes along x (bottom & top)
Ny = 20  # nodes along y (left & right)

gmsh.model.mesh.setTransfiniteCurve(c_bottom, Nx)
gmsh.model.mesh.setTransfiniteCurve(c_top, Nx)     # opposite to bottom
gmsh.model.mesh.setTransfiniteCurve(c_left, Ny)
gmsh.model.mesh.setTransfiniteCurve(c_right, Ny)   # opposite to left

# Generate structured mesh
gmsh.model.mesh.setTransfiniteSurface(surf)
gmsh.model.mesh.setRecombine(2, surf)
gmsh.model.mesh.generate(2)

gmsh.write("mesh/mesh.msh")
gmsh.finalize()
```

## Adding Grading (Optional)

Refine toward one edge (e.g., left side for fixed BC):

```python
# Progression > 1: elements grow toward end
# Progression < 1: elements shrink toward end
gmsh.model.mesh.setTransfiniteCurve(c_bottom, Nx, "Progression", 1.1)
gmsh.model.mesh.setTransfiniteCurve(c_top, Nx, "Progression", 1.1)
```

## Verification Checklist

- [ ] 1 block with exactly 4 edges
- [ ] Nx (bottom) = Nx (top)
- [ ] Ny (left) = Ny (right)
- [ ] setTransfiniteSurface called
- [ ] setRecombine called
