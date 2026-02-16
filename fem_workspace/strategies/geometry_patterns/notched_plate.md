# Geometry Pattern: Notched Plate (Edge Notch)

## Common Notch Types

```
Single edge notch:        Double edge notch:       V-notch:
┌───────────────┐         ┌───────────────┐        ┌───────────────┐
│               │         │               │        │               │
├───┐       ┌───┤         ├───┐       ┌───┤        │\             /│
│   │       │   │         │   │       │   │        │ \           / │
│   │       │   │         │   │       │   │        │  \    ↓    /  │
├───┘       └───┤         ├───┘       └───┤        │   \  notch/   │
│               │         │               │        │    \     /    │
└───────────────┘         └───────────────┘        └─────\───/─────┘
```

## Single Edge Notch: 3 Blocks

```
    ┌───────────────────┐
    │                   │
    │      Block 1      │
    │                   │
    ├─────┬───────┬─────┤
    │     │       │     │
    │ B2  │ notch │ B3  │
    │     │       │     │
    └─────┴───────┴─────┘
```

**Key insight:** The notch creates 2 re-entrant corners (singular points).

## Implementation

```python
#filename.py
import gmsh

gmsh.initialize()
gmsh.model.add("notched_plate")
geo = gmsh.model.geo

# Parameters
Lx, Ly = 200, 100
notch_width = 20
notch_depth = 30
notch_x = Lx / 2  # centered on bottom edge

# Outer points
p1 = geo.addPoint(0, 0, 0)
p2 = geo.addPoint(Lx, 0, 0)
p3 = geo.addPoint(Lx, Ly, 0)
p4 = geo.addPoint(0, Ly, 0)

# Notch points
notch_left = (Lx - notch_width) / 2
notch_right = (Lx + notch_width) / 2

p5 = geo.addPoint(notch_left, 0, 0)       # notch bottom-left
p6 = geo.addPoint(notch_right, 0, 0)      # notch bottom-right
p7 = geo.addPoint(notch_left, notch_depth, 0)   # notch top-left (re-entrant)
p8 = geo.addPoint(notch_right, notch_depth, 0)  # notch top-right (re-entrant)

# Partition points (extend re-entrant corners to top)
p9 = geo.addPoint(notch_left, Ly, 0)
p10 = geo.addPoint(notch_right, Ly, 0)

# Outer boundary curves
c_b1 = geo.addLine(p1, p5)    # bottom-left
c_b2 = geo.addLine(p6, p2)    # bottom-right
c_r = geo.addLine(p2, p3)     # right
c_t1 = geo.addLine(p3, p10)   # top-right
c_t2 = geo.addLine(p10, p9)   # top-middle
c_t3 = geo.addLine(p9, p4)    # top-left
c_l = geo.addLine(p4, p1)     # left

# Notch curves
c_n1 = geo.addLine(p5, p7)    # notch left wall
c_n2 = geo.addLine(p7, p8)    # notch bottom (inside)
c_n3 = geo.addLine(p8, p6)    # notch right wall

# Partition lines (from notch corners to top)
c_p1 = geo.addLine(p7, p9)    # left partition
c_p2 = geo.addLine(p8, p10)   # right partition

geo.synchronize()

# ===== 3 BLOCKS =====

# Block 1: Main body above notch
loop1 = geo.addCurveLoop([c_t2, c_t3, c_l, c_b1, c_n1, c_p1])
# That's 6 edges! Need different approach.

# PROBLEM: With partitions from notch to top, Block 1 still has >4 edges
# SOLUTION: Need 4 blocks, not 3

gmsh.finalize()

# ===== CORRECTED: 5-Block Layout =====

gmsh.initialize()
gmsh.model.add("notched_plate_v2")
geo = gmsh.model.geo

# Same points as before...
# Add partition that splits the top block

# Block layout:
#     ┌─────┬─────┬─────┐
#     │  1  │  2  │  3  │
#     ├─────┼─────┼─────┤
#     │  4  │notch│  5  │
#     └─────┴─────┴─────┘

# This gives 5 blocks, each with 4 edges
# (The "notch" area is not a block - it's the cutout)

# ... implementation continues
```

## Recommended Approach for Notches

Due to complexity, often better to use fallback:

```python
#filename.py
import gmsh

gmsh.initialize()
gmsh.model.add("notched")
occ = gmsh.model.occ

# Create plate with notch using boolean
plate = occ.addRectangle(0, 0, 0, Lx, Ly)
notch = occ.addRectangle(notch_left, 0, 0, notch_width, notch_depth)
occ.cut([(2, plate)], [(2, notch)])
occ.synchronize()

# Algorithm 8 handles this well
gmsh.option.setNumber("Mesh.Algorithm", 8)
gmsh.option.setNumber("Mesh.RecombinationAlgorithm", 3)
gmsh.option.setNumber("Mesh.RecombineAll", 1)

# Refine at notch tip (stress concentration)
# ... distance field setup

gmsh.model.mesh.generate(2)
```

## Stress Concentration Reminder

Notch tip is a **severe stress concentration**. Always refine mesh there:

```python
#filename.py
# Refine near notch corners
corner_pts = [p7, p8]  # re-entrant corners
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "PointsList", corner_pts)

gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", 0.5)   # very fine
gmsh.model.mesh.field.setNumber(2, "SizeMax", 5)
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", 20)
```
