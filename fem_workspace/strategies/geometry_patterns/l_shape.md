# Geometry Pattern: L-Shape

## Block Layout: 3 Blocks Minimum

```
    ┌─────────────┐
    │             │
    │   Block 1   │
    │             │
    ├──────┬──────┘
    │      │
    │  B2  │   ← re-entrant corner (stress concentration)
    │      │
    └──────┘
    
    Or alternatively:
    
    ┌─────────────┐
    │      │      │
    │  B1  │  B2  │
    │      │      │
    ├──────┼──────┘
    │      │
    │  B3  │
    │      │
    └──────┘
```

**Blocks needed:** 3

## Dimensions Reference

```
         W1
    ┌───────────┐
    │           │ H1
    │           │
    ├─────┬─────┘
    │     │
 H2 │     │ W2
    │     │
    └─────┘
```

## Implementation (3-Block Version)

```python
#filename.py
import gmsh

gmsh.initialize()
gmsh.model.add("L_shape")
geo = gmsh.model.geo

# Parameters
W1, H1 = 150, 50   # top horizontal arm
W2, H2 = 50, 100   # bottom vertical arm

# Points (counter-clockwise from bottom-left)
p1 = geo.addPoint(0, 0, 0)           # bottom-left
p2 = geo.addPoint(W2, 0, 0)          # bottom-right of vertical arm
p3 = geo.addPoint(W2, H2, 0)         # re-entrant corner (SINGULAR POINT)
p4 = geo.addPoint(W1, H2, 0)         # right end of horizontal arm
p5 = geo.addPoint(W1, H1 + H2, 0)    # top-right
p6 = geo.addPoint(0, H1 + H2, 0)     # top-left

# Additional point for partitioning (extends re-entrant corner vertically)
p7 = geo.addPoint(W2, H1 + H2, 0)    # partition point on top edge

# Outer boundary curves
c1 = geo.addLine(p1, p2)    # bottom
c2 = geo.addLine(p2, p3)    # right side of vertical arm
c3 = geo.addLine(p3, p4)    # bottom of horizontal arm (internal step)
c4 = geo.addLine(p4, p5)    # right side of horizontal arm
c5 = geo.addLine(p5, p7)    # top edge part 1
c6 = geo.addLine(p7, p6)    # top edge part 2
c7 = geo.addLine(p6, p1)    # left side

# Partition line (from re-entrant corner to top)
c_partition = geo.addLine(p3, p7)

geo.synchronize()

# ===== BLOCK 1: Top-left (4 edges) =====
# Edges: c6, c7, part of left?, c_partition
# Need to split c7...

# Actually, let's reconsider. With partition from p3 to p7:
# Block 1: c6 (p7→p6), c7 (p6→p1), c1 (p1→p2), c2 (p2→p3), c_partition (p3→p7)
# That's 5 edges! Need different partition.

gmsh.finalize()

# ===== CORRECTED: 3-Block with Horizontal Partition =====

gmsh.initialize()
gmsh.model.add("L_shape_v2")
geo = gmsh.model.geo

# Points
p1 = geo.addPoint(0, 0, 0)
p2 = geo.addPoint(W2, 0, 0)
p3 = geo.addPoint(W2, H2, 0)        # re-entrant corner
p4 = geo.addPoint(W1, H2, 0)
p5 = geo.addPoint(W1, H1 + H2, 0)
p6 = geo.addPoint(0, H1 + H2, 0)
p7 = geo.addPoint(0, H2, 0)         # partition point on left edge

# Curves
c_bottom = geo.addLine(p1, p2)
c_right_lower = geo.addLine(p2, p3)
c_step = geo.addLine(p3, p4)
c_right_upper = geo.addLine(p4, p5)
c_top = geo.addLine(p5, p6)
c_left_upper = geo.addLine(p6, p7)
c_left_lower = geo.addLine(p7, p1)

# Partition line (horizontal, from left edge to re-entrant corner)
c_partition = geo.addLine(p7, p3)

geo.synchronize()

# ===== BLOCK 1: Upper horizontal arm (4 edges) =====
loop1 = geo.addCurveLoop([c_step, c_right_upper, c_top, c_left_upper, -c_partition])
# That's 5 edges again!

# THE ISSUE: L-shape with re-entrant corner naturally creates 5-sided regions
# SOLUTION: Need to add another partition point

gmsh.finalize()

# ===== FINAL CORRECT VERSION =====

gmsh.initialize()
gmsh.model.add("L_shape_correct")
geo = gmsh.model.geo

# Points  
p1 = geo.addPoint(0, 0, 0)
p2 = geo.addPoint(W2, 0, 0)
p3 = geo.addPoint(W2, H2, 0)        # re-entrant corner
p4 = geo.addPoint(W1, H2, 0)
p5 = geo.addPoint(W1, H1 + H2, 0)
p6 = geo.addPoint(0, H1 + H2, 0)

# Partition points
p7 = geo.addPoint(0, H2, 0)         # on left edge at height H2
p8 = geo.addPoint(W2, H1 + H2, 0)   # on top edge at x = W2

# Curves - outer boundary split
c1 = geo.addLine(p1, p2)      # bottom
c2 = geo.addLine(p2, p3)      # right-lower
c3 = geo.addLine(p3, p4)      # step (horizontal internal)
c4 = geo.addLine(p4, p5)      # right-upper
c5 = geo.addLine(p5, p8)      # top-right
c6 = geo.addLine(p8, p6)      # top-left
c7 = geo.addLine(p6, p7)      # left-upper
c8 = geo.addLine(p7, p1)      # left-lower

# Partition lines
c_h = geo.addLine(p7, p3)     # horizontal partition
c_v = geo.addLine(p3, p8)     # vertical partition

geo.synchronize()

# ===== BLOCK 1: Top-left rectangle (4 edges) =====
loop1 = geo.addCurveLoop([c6, c7, -c_h, c_v])  # Nope, wrong connectivity

# Let me trace carefully:
# Block 1 (top-left): p8 → p6 (c6), p6 → p7 (c7), p7 → p3 (c_h), p3 → p8 (c_v)
# That's: c6, c7, c_h, c_v - but need to check directions

loop1 = geo.addCurveLoop([-c6, -c7, c_h, c_v])  
# p8→p6 is -c6 (c6 goes p5→p8)... this is getting confusing

# CLEANER APPROACH: Define curves with clear from→to naming
gmsh.finalize()
```

## Recommended: Cleaner 3-Block Implementation

```python
#filename.py
import gmsh
import numpy as np

gmsh.initialize()
gmsh.model.add("L_shape")
geo = gmsh.model.geo

# L-shape parameters
W_total = 150    # total width
H_total = 150    # total height  
W_notch = 100    # notch width (removed from top-right)
H_notch = 100    # notch height

# This creates an L like:
#  ┌────┐
#  │    │
#  │    └────┐
#  │         │
#  └─────────┘

# 6 corner points of L-shape
p1 = geo.addPoint(0, 0, 0)
p2 = geo.addPoint(W_total, 0, 0)
p3 = geo.addPoint(W_total, H_total - H_notch, 0)
p4 = geo.addPoint(W_total - W_notch, H_total - H_notch, 0)  # re-entrant
p5 = geo.addPoint(W_total - W_notch, H_total, 0)
p6 = geo.addPoint(0, H_total, 0)

# Partition points
p7 = geo.addPoint(W_total - W_notch, 0, 0)  # on bottom edge
p8 = geo.addPoint(0, H_total - H_notch, 0)  # on left edge

# Outer curves
c_b1 = geo.addLine(p1, p7)   # bottom-left
c_b2 = geo.addLine(p7, p2)   # bottom-right  
c_r = geo.addLine(p2, p3)    # right
c_step = geo.addLine(p3, p4) # step (re-entrant)
c_inner = geo.addLine(p4, p5)# inner vertical
c_t = geo.addLine(p5, p6)    # top
c_l1 = geo.addLine(p6, p8)   # left-upper
c_l2 = geo.addLine(p8, p1)   # left-lower

# Partition lines
c_pv = geo.addLine(p7, p4)   # vertical partition (through re-entrant)
c_ph = geo.addLine(p8, p4)   # horizontal partition

geo.synchronize()

# ===== 3 BLOCKS (each has exactly 4 edges) =====

# Block 1: Bottom-left quadrilateral
# p1 → p7 → p4 → p8 → p1
loop1 = geo.addCurveLoop([c_b1, c_pv, -c_ph, -c_l2])
surf1 = geo.addPlaneSurface([loop1])

# Block 2: Bottom-right quadrilateral  
# p7 → p2 → p3 → p4 → p7
loop2 = geo.addCurveLoop([c_b2, c_r, c_step, -c_pv])
surf2 = geo.addPlaneSurface([loop2])

# Block 3: Top-left quadrilateral
# p8 → p4 → p5 → p6 → p8
loop3 = geo.addCurveLoop([c_ph, c_inner, c_t, c_l1])
surf3 = geo.addPlaneSurface([loop3])

# ===== NODE COUNTS (opposites must match) =====

# Block 1: c_b1 opposite to c_ph, c_pv opposite to c_l2
N1 = 15  # bottom-left / horizontal-partition
N2 = 20  # vertical-partition / left-lower

gmsh.model.mesh.setTransfiniteCurve(c_b1, N1)
gmsh.model.mesh.setTransfiniteCurve(c_ph, N1)   # opposite
gmsh.model.mesh.setTransfiniteCurve(c_pv, N2)
gmsh.model.mesh.setTransfiniteCurve(c_l2, N2)   # opposite

# Block 2: c_b2 opposite to c_step, c_r opposite to c_pv
N3 = 20  # bottom-right / step
N4 = 15  # right / vertical-partition (must equal N2? Check!)

gmsh.model.mesh.setTransfiniteCurve(c_b2, N3)
gmsh.model.mesh.setTransfiniteCurve(c_step, N3)
gmsh.model.mesh.setTransfiniteCurve(c_r, N2)    # shares c_pv, so must be N2

# Block 3: c_ph opposite to c_t, c_inner opposite to c_l1
N5 = 10  # inner / left-upper

gmsh.model.mesh.setTransfiniteCurve(c_t, N1)    # opposite to c_ph
gmsh.model.mesh.setTransfiniteCurve(c_inner, N5)
gmsh.model.mesh.setTransfiniteCurve(c_l1, N5)

# ===== GRADING (refine at re-entrant corner) =====
gmsh.model.mesh.setTransfiniteCurve(c_pv, N2, "Bump", 0.2)  # fine at both ends
gmsh.model.mesh.setTransfiniteCurve(c_ph, N1, "Bump", 0.2)

# ===== GENERATE =====
for surf in [surf1, surf2, surf3]:
    gmsh.model.mesh.setTransfiniteSurface(surf)
    gmsh.model.mesh.setRecombine(2, surf)

gmsh.model.mesh.generate(2)
gmsh.write("mesh/mesh.msh")
gmsh.finalize()
```

## Key Insight for L-Shapes

The re-entrant corner is a **singular point** where partitions must meet. You need:
1. One partition line from re-entrant corner to bottom edge
2. One partition line from re-entrant corner to left edge

This creates exactly 3 four-sided blocks.

## Node Count Constraint Table

| Block | Edge 1 | N | Edge 2 (opposite) | N |
|-------|--------|---|-------------------|---|
| B1 | c_b1 | 15 | c_ph | 15 |
| B1 | c_pv | 20 | c_l2 | 20 |
| B2 | c_b2 | 20 | c_step | 20 |
| B2 | c_r | 20 | c_pv | 20 |
| B3 | c_ph | 15 | c_t | 15 |
| B3 | c_inner | 10 | c_l1 | 10 |

Note: Shared edges (c_pv, c_ph) propagate constraints between blocks!
