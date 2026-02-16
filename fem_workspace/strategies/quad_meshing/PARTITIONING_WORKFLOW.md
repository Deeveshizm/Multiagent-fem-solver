# Meshing Workflow

## Quad Meshing Approaches

| Geometry | Recommended Approach | Notes |
|----------|---------------------|-------|
| Simple rectangle | Transfinite (1 block) | Easy, structured grid |
| Rectangle + hole | **OCC Boolean + Algorithm 8** | Correct geometry, good quads |
| L-shape, complex | OCC Boolean + Algorithm 8 | Works for most geometries |

## ⚠️ CRITICAL RULES

- **NEVER use `geo.addSpline()` for straight edges** - creates curved boundaries!
- **USE OCC kernel** (`gmsh.model.occ`) for boolean operations
- **USE Algorithm 8 + RecombineAll** for quad meshing
- **USE `occ.addRectangle()` + `occ.addDisk()` + `occ.cut()`** for plate+hole

## Solver Compatibility Note

This workflow targets **DOLFINx 0.10.0** (FEniCSx). DOLFINx handles all quad meshes correctly.
The mesh is read directly from .msh files using `dolfinx.io.gmsh.read_from_msh()`.

---

## ⚠️ QUALITY EXTRACTION - IMPORTANT

**DO NOT use `gmsh.model.mesh.getElementQuality()`** - this method does not exist in Gmsh 4.15.0!

If you must use Gmsh's API (before meshio export), the correct method is:
```python
# Correct Gmsh 4.15.0 API for quality
elem_types, elem_tags, _ = gmsh.model.mesh.getElements(dim=2)
all_tags = []
for tags in elem_tags:
    all_tags.extend(tags)
qualities = gmsh.model.mesh.getElementQualities(all_tags)  # Note: PLURAL + requires element tags
min_quality = float(np.min(qualities))
avg_quality = float(np.mean(qualities))
```

**Recommended approach:** Use the meshio-based quality computation shown in the code templates below (after export). This is more reliable and doesn't depend on Gmsh API details.

---

## QUAD MESH: OCC Boolean + Algorithm 8 (Recommended for Rectangle + Hole)

This is the simplest and most reliable approach for plate-with-hole geometries.

```python
# filename: generate_mesh.py
import gmsh
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.collections import PolyCollection
import json
import os

os.makedirs("mesh", exist_ok=True)

# ========== PARAMETERS ==========
Lx, Ly = 200, 100       # Plate dimensions (mm)
R = 20                   # Hole radius (mm)
cx, cy = 0, 0           # Hole center

fine_size = 2           # Mesh size near hole
coarse_size = 10        # Mesh size far from hole

# ========== GEOMETRY (OCC kernel) ==========
gmsh.initialize()
gmsh.model.add("plate_with_hole")
occ = gmsh.model.occ

# Create rectangle - guarantees straight edges!
plate = occ.addRectangle(-Lx/2, -Ly/2, 0, Lx, Ly)

# Create circular hole
hole = occ.addDisk(cx, cy, 0, R, R)

# Boolean subtraction: plate - hole
occ.cut([(2, plate)], [(2, hole)])
occ.synchronize()

# Get the resulting surface
surfaces = gmsh.model.getEntities(2)
surf_tag = surfaces[0][1]

# ========== PHYSICAL GROUPS ==========
boundaries = gmsh.model.getBoundary([(2, surf_tag)], oriented=False)
boundary_tags = [abs(b[1]) for b in boundaries]

left_curves, right_curves, top_curves, bottom_curves, hole_curves = [], [], [], [], []

for tag in boundary_tags:
    com = gmsh.model.occ.getCenterOfMass(1, tag)
    x, y = com[0], com[1]
    dist_from_center = np.sqrt(x**2 + y**2)
    
    if dist_from_center < R * 1.5:
        hole_curves.append(tag)
    elif abs(x - (-Lx/2)) < 1:
        left_curves.append(tag)
    elif abs(x - (Lx/2)) < 1:
        right_curves.append(tag)
    elif abs(y - (Ly/2)) < 1:
        top_curves.append(tag)
    elif abs(y - (-Ly/2)) < 1:
        bottom_curves.append(tag)

if left_curves:
    gmsh.model.addPhysicalGroup(1, left_curves, 1)
    gmsh.model.setPhysicalName(1, 1, "left")
if right_curves:
    gmsh.model.addPhysicalGroup(1, right_curves, 2)
    gmsh.model.setPhysicalName(1, 2, "right")
if top_curves:
    gmsh.model.addPhysicalGroup(1, top_curves, 3)
    gmsh.model.setPhysicalName(1, 3, "top")
if bottom_curves:
    gmsh.model.addPhysicalGroup(1, bottom_curves, 4)
    gmsh.model.setPhysicalName(1, 4, "bottom")
if hole_curves:
    gmsh.model.addPhysicalGroup(1, hole_curves, 5)
    gmsh.model.setPhysicalName(1, 5, "hole")

gmsh.model.addPhysicalGroup(2, [surf_tag], 100)
gmsh.model.setPhysicalName(2, 100, "domain")

# ========== MESH SIZE FIELD ==========
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "CurvesList", hole_curves)
gmsh.model.mesh.field.setNumber(1, "Sampling", 100)

gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", fine_size)
gmsh.model.mesh.field.setNumber(2, "SizeMax", coarse_size)
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", R * 3)
gmsh.model.mesh.field.setAsBackgroundMesh(2)

# ========== GENERATE QUAD MESH ==========
gmsh.option.setNumber("Mesh.Algorithm", 8)  # Frontal-Delaunay for quads
gmsh.option.setNumber("Mesh.RecombinationAlgorithm", 2)  # simple
gmsh.option.setNumber("Mesh.RecombineAll", 1)
gmsh.option.setNumber("Mesh.Smoothing", 100)
gmsh.option.setNumber("Mesh.MeshSizeFromPoints", 0)
gmsh.option.setNumber("Mesh.MeshSizeExtendFromBoundary", 0)
gmsh.option.setNumber("Mesh.MeshSizeFromCurvature", 0)

gmsh.model.mesh.generate(2)

# ========== QUALITY EXTRACTION (Gmsh API - before finalize) ==========
# Get element tags for quality computation
elem_types, elem_tags, _ = gmsh.model.mesh.getElements(dim=2)
all_tags = []
for tags in elem_tags:
    all_tags.extend(tags)

# Determine element type
elem_type_name = "unknown"
num_elements = len(all_tags)
for i, et in enumerate(elem_types):
    props = gmsh.model.mesh.getElementProperties(int(et))
    name = props[0].lower()
    if "quad" in name:
        elem_type_name = "quad"
    elif "triangle" in name:
        elem_type_name = "triangle"

# Get quality using CORRECT Gmsh 4.15.0 API
# NOTE: Use getElementQualities (PLURAL) with element tags as argument
qualities_gmsh = gmsh.model.mesh.getElementQualities(all_tags)
min_quality_gmsh = float(np.min(qualities_gmsh))
avg_quality_gmsh = float(np.mean(qualities_gmsh))

print(f"Gmsh quality: min={min_quality_gmsh:.4f}, avg={avg_quality_gmsh:.4f}")

gmsh.write("mesh/mesh.msh")
gmsh.finalize()

# ========== LOAD MESH FOR VISUALIZATION ==========
# Use meshio for visualization and secondary quality check
import meshio
m = meshio.read("mesh/mesh.msh")
pts = m.points[:, :2]  # 2D points only!

# Get cells (quads or triangles)
if "quad" in m.cells_dict:
    cells = m.cells_dict["quad"]
    cell_type = "quad"
else:
    cells = m.cells_dict["triangle"]
    cell_type = "triangle"

# ========== VISUALIZATION ==========
fig, ax = plt.subplots(figsize=(12, 6))
ax.set_aspect('equal')

if cell_type == "quad":
    quads = pts[cells]
    pc = PolyCollection(quads, facecolors='lightblue', edgecolors='black', linewidths=0.3)
    ax.add_collection(pc)
else:
    ax.triplot(pts[:, 0], pts[:, 1], cells, linewidth=0.3, color='black')

theta = np.linspace(0, 2*np.pi, 100)
ax.plot(R * np.cos(theta), R * np.sin(theta), 'r-', linewidth=1.5)
ax.set_xlim(-Lx/2 - 10, Lx/2 + 10)
ax.set_ylim(-Ly/2 - 10, Ly/2 + 10)
ax.set_xlabel("x (mm)")
ax.set_ylabel("y (mm)")
ax.set_title(f"Mesh: {len(cells)} {cell_type}s")
plt.tight_layout()
plt.savefig("mesh/mesh_visualization.png", dpi=150)
plt.close()

# ========== ALTERNATIVE QUALITY (meshio-based, for verification) ==========
def quad_quality(nodes):
    """Diagonal ratio quality metric for quads"""
    diag1 = np.linalg.norm(nodes[2] - nodes[0])
    diag2 = np.linalg.norm(nodes[3] - nodes[1])
    if max(diag1, diag2) == 0:
        return 0
    return min(diag1, diag2) / max(diag1, diag2)

def tri_quality(nodes):
    """Normalized shape ratio for triangles"""
    a = np.linalg.norm(nodes[1] - nodes[0])
    b = np.linalg.norm(nodes[2] - nodes[1])
    c = np.linalg.norm(nodes[0] - nodes[2])
    s = (a + b + c) / 2
    area = np.sqrt(max(0, s * (s-a) * (s-b) * (s-c)))
    denom = a*a + b*b + c*c
    return 4 * np.sqrt(3) * area / denom if denom > 0 else 0

if cell_type == "quad":
    qualities_meshio = [quad_quality(pts[q]) for q in cells]
else:
    qualities_meshio = [tri_quality(pts[t]) for t in cells]

# ========== WRITE QUALITY JSON ==========
# Use Gmsh quality values (more accurate for FEM)
quality_data = {
    "num_elements": num_elements,
    "num_nodes": len(pts),
    "element_type": cell_type,
    "meshing_method": "occ_boolean_algorithm8_recombine",
    "min_quality": round(min_quality_gmsh, 4),
    "avg_quality": round(avg_quality_gmsh, 4),
    "mesh_ready_for_solver": True,
    "fenics_compatible": True
}

with open("mesh/mesh_quality.json", "w") as f:
    json.dump(quality_data, f, indent=2)

# ========== WRITE MARKER MAP ==========
marker_map = {
    "left": 1,
    "right": 2,
    "top": 3,
    "bottom": 4,
    "hole": 5,
    "domain": 100
}

with open("mesh/marker_map.json", "w") as f:
    json.dump(marker_map, f, indent=2)

print(f"Generated {num_elements} {cell_type}s")
print(f"Quality: min={min_quality_gmsh:.4f}, avg={avg_quality_gmsh:.4f}")
print("Files written: mesh/mesh.msh, mesh/mesh_quality.json, mesh/marker_map.json, mesh/mesh_visualization.png")
```

---

## DOLFINx 0.10.0 Solver Integration

The solver reads the mesh directly from .msh file. **No XDMF conversion needed.**

```python
# In solver code (DOLFINx 0.10.0)
from dolfinx.io import gmsh as gmsh_io  # NOTE: module is 'gmsh', not 'gmshio'
from mpi4py import MPI

# read_from_msh returns a tuple - unpack carefully
# DOLFINx 0.10.0 returns: (mesh, cell_tags, facet_tags)
mesh, cell_tags, facet_tags = gmsh_io.read_from_msh("mesh/mesh.msh", MPI.COMM_WORLD, gdim=2)

# Load marker map
import json
with open("mesh/marker_map.json", "r") as f:
    markers = json.load(f)

# Use markers for boundary conditions
left_marker = markers["left"]      # 1
right_marker = markers["right"]    # 2
hole_marker = markers["hole"]      # 5
```

---

## QUAD MESH: Transfinite (Simple Rectangle Only)

Follow these 8 steps. Every quad mesh using partitioning must complete ALL steps.

---

## STEP 1: Analyze Geometry

**What to document:**
- Outer boundary shape and dimensions
- Internal features (holes, cutouts)
- Corner types (90°, re-entrant, curved)

**Example output:**
```
Geometry: Rectangular plate 200mm × 100mm
Features: Central circular hole, radius 20mm at (0, 0)
Corners: 4 outer corners (90°), hole creates internal boundary
```

---

## STEP 2: Identify Singular Points

**Definition:** Points where mesh topology must change (elements ≠ 4 meeting)

**Common singular points:**
- Hole centers (conceptually)
- Re-entrant corners (L-shapes)
- Where partition lines meet

**Example output:**
```
Singular points:
- 4 outer corners: (-100,-50), (100,-50), (100,50), (-100,50)
- 4 O-grid junctions around hole (where blocks meet)
```

---

## STEP 3: Plan Block Topology

**Rule:** Each hole requires minimum 4 blocks (O-grid pattern)

**Draw your block layout:**
```
For plate with central hole (4-block O-grid):

        top
    ┌────┬────┐
    │ B1 │ B2 │
    │    │    │
left├────○────┤right   ○ = hole
    │    │    │
    │ B4 │ B3 │
    └────┴────┘
       bottom

Blocks: 4
Partition lines: 1 horizontal (through hole), 1 vertical (through hole)
```

---

## STEP 4: Draw Partition Lines

**List each partition line:**
| Line | From | To | Purpose |
|------|------|-----|---------|
| H1 | hole-left | left-edge | Horizontal partition |
| H2 | hole-right | right-edge | Horizontal partition |
| V1 | hole-top | top-edge | Vertical partition |
| V2 | hole-bottom | bottom-edge | Vertical partition |

**In Gmsh:**
```python
# Partition lines from hole to boundary
line_h1 = geo.addLine(pt_hole_left, pt_boundary_left)
line_h2 = geo.addLine(pt_hole_right, pt_boundary_right)
line_v1 = geo.addLine(pt_hole_top, pt_boundary_top)
line_v2 = geo.addLine(pt_hole_bottom, pt_boundary_bottom)
```

---

## STEP 5: Verify All Blocks Are 4-Sided

**⚠️ CRITICAL: Every block MUST have exactly 4 boundary curves**

**Verify each block:**
```
Block B1 (top-left quadrant):
  Edge 1: top-left portion of outer boundary
  Edge 2: vertical partition (V1)
  Edge 3: hole arc (top-left quarter)
  Edge 4: horizontal partition (H1)
  Total: 4 edges ✓

Block B2 (top-right quadrant):
  Edge 1: top-right portion of outer boundary  
  Edge 2: horizontal partition (H2)
  Edge 3: hole arc (top-right quarter)
  Edge 4: vertical partition (V1)
  Total: 4 edges ✓

[Continue for all blocks...]
```

**If any block has ≠ 4 edges:** STOP and fix topology before proceeding.

---

## STEP 6: Assign Node Counts (CRITICAL!)

**THE GOLDEN RULE:**
> Opposite edges in each block MUST have the SAME number of nodes

**Create constraint table:**
```
| Curve | N | Opposite | Block | Verified |
|-------|---|----------|-------|----------|
| top_outer_left | 15 | hole_arc_top_left | B1 | ✓ |
| left_outer_top | 10 | partition_V1_upper | B1 | ✓ |
| hole_arc_top_left | 15 | top_outer_left | B1 | ✓ |
| partition_H1 | 10 | partition_V1_upper | B1 | ✓ |
...
```

**Common node count strategy:**
- Hole quarter-arcs: N_arc (e.g., 12 nodes)
- Radial partitions (hole to boundary): N_radial (e.g., 15 nodes)
- Outer edge segments: Must match opposite (either N_arc or another outer segment)

**In Gmsh:**
```python
N_arc = 12      # Nodes along each quarter of hole
N_radial = 15   # Nodes from hole to outer boundary

# Hole arcs
gmsh.model.mesh.setTransfiniteCurve(hole_arc_1, N_arc)
gmsh.model.mesh.setTransfiniteCurve(hole_arc_2, N_arc)
gmsh.model.mesh.setTransfiniteCurve(hole_arc_3, N_arc)
gmsh.model.mesh.setTransfiniteCurve(hole_arc_4, N_arc)

# Partition lines (radial)
gmsh.model.mesh.setTransfiniteCurve(partition_h1, N_radial)
gmsh.model.mesh.setTransfiniteCurve(partition_h2, N_radial)
gmsh.model.mesh.setTransfiniteCurve(partition_v1, N_radial)
gmsh.model.mesh.setTransfiniteCurve(partition_v2, N_radial)

# Outer edges - must match opposites!
gmsh.model.mesh.setTransfiniteCurve(outer_top_left, N_arc)    # Opposite to hole_arc
gmsh.model.mesh.setTransfiniteCurve(outer_top_right, N_arc)
# ... etc
```

---

## STEP 7: Apply Grading (Refinement)

**Where to refine:**
- Near holes (stress concentration)
- Near re-entrant corners
- Near load application points

**Grading options:**
```python
# Uniform (default)
gmsh.model.mesh.setTransfiniteCurve(curve, N)

# Progression (refine toward start of curve)
# coef > 1: elements grow toward end
gmsh.model.mesh.setTransfiniteCurve(curve, N, "Progression", 1.15)

# Bump (refine at both ends)
gmsh.model.mesh.setTransfiniteCurve(curve, N, "Bump", 0.2)
```

**Example - refine toward hole:**
```python
# Partition lines from hole to boundary - fine at hole end
gmsh.model.mesh.setTransfiniteCurve(partition_h1, N_radial, "Progression", 1.2)
gmsh.model.mesh.setTransfiniteCurve(partition_h2, N_radial, "Progression", 0.83)  # 1/1.2, reversed
```

---

## STEP 8: Generate Mesh & Validate

**Apply transfinite to all surfaces:**
```python
for surf in [surf_b1, surf_b2, surf_b3, surf_b4]:
    gmsh.model.mesh.setTransfiniteSurface(surf)
    gmsh.model.mesh.setRecombine(2, surf)

gmsh.model.mesh.generate(2)
```

**Validation checklist:**
```python
# 1. Check for quads
assert "quad" in mesh.cells_dict, "No quads generated!"

# 2. Check for unwanted triangles
assert "triangle" not in mesh.cells_dict, "Triangles found - partitioning failed!"

# 3. Check quality
assert min_quality > 0.2, f"Quality too low: {min_quality}"

# 4. Visual inspection
# Save PNG and verify structured pattern
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────┐
│  PARTITIONING RULES                             │
├─────────────────────────────────────────────────┤
│  ✓ Every block = exactly 4 edges                │
│  ✓ Opposite edges = same node count             │
│  ✓ Call setTransfiniteCurve on ALL curves       │
│  ✓ Call setTransfiniteSurface on ALL surfaces   │
│  ✓ Call setRecombine(2, tag) on ALL surfaces    │
├─────────────────────────────────────────────────┤
│  MINIMUM BLOCKS                                 │
├─────────────────────────────────────────────────┤
│  Rectangle (no hole): 1 block                   │
│  Rectangle + 1 hole: 4 blocks                   │
│  Rectangle + 2 holes: 8+ blocks                 │
│  L-shape: 3 blocks                              │
├─────────────────────────────────────────────────┤
│  QUALITY EXTRACTION (Gmsh 4.15.0)               │
├─────────────────────────────────────────────────┤
│  ✗ WRONG: getElementQuality(2)                  │
│  ✓ RIGHT: getElementQualities(element_tags)     │
└─────────────────────────────────────────────────┘
```

---

## Common Errors and Fixes

| Error Message | Cause | Fix |
|--------------|-------|-----|
| "Surface X is not transfinite" | Surface doesn't have 4 curves | Check Step 5 |
| "Incompatible transfinite constraints" | Opposite edges have different N | Check Step 6 |
| Triangles in output | setRecombine not called | Add setRecombine(2, surf) |
| Mesh has gaps | Shared edges not connected | Use same point tags |
| `getElementQuality` not found | Wrong API method | Use `getElementQualities(tags)` |
| Quality shows 0.0 | Quality extraction failed silently | Use correct Gmsh API or meshio method |

---
---

# TRIANGLE WORKFLOW (Alternative/Fallback)

## When to Use Triangles

- Very complex boundaries where Algorithm 8 struggles
- When quad quality is poor
- Specific solver requirements

## Complete Triangle Mesh Code

```python
# filename: generate_mesh.py
import gmsh
import numpy as np
import matplotlib.pyplot as plt
import json
import os

os.makedirs("mesh", exist_ok=True)

# ========== PARAMETERS ==========
Lx, Ly = 200, 100       # Plate dimensions (mm)
R = 20                   # Hole radius (mm)
cx, cy = 0, 0           # Hole center

fine_size = 2           # Mesh size near hole
coarse_size = 10        # Mesh size far from hole

# ========== GEOMETRY (OCC kernel) ==========
gmsh.initialize()
gmsh.model.add("plate_with_hole")
occ = gmsh.model.occ

# Create rectangle and hole
plate = occ.addRectangle(-Lx/2, -Ly/2, 0, Lx, Ly)
hole = occ.addDisk(cx, cy, 0, R, R)

# Cut hole from plate
result = occ.cut([(2, plate)], [(2, hole)])
occ.synchronize()

# Get the resulting surface
surfaces = gmsh.model.getEntities(2)
surf_tag = surfaces[0][1]

# ========== PHYSICAL GROUPS ==========
# Get all boundary curves
boundaries = gmsh.model.getBoundary([(2, surf_tag)], oriented=False)
boundary_tags = [abs(b[1]) for b in boundaries]

# Classify curves by position
left_curves = []
right_curves = []
top_curves = []
bottom_curves = []
hole_curves = []

for tag in boundary_tags:
    com = gmsh.model.occ.getCenterOfMass(1, tag)
    x, y = com[0], com[1]
    
    # Check if it's on the hole (distance from center ≈ R)
    dist_from_center = np.sqrt(x**2 + y**2)
    if dist_from_center < R * 1.5:
        hole_curves.append(tag)
    elif abs(x - (-Lx/2)) < 1:
        left_curves.append(tag)
    elif abs(x - (Lx/2)) < 1:
        right_curves.append(tag)
    elif abs(y - (Ly/2)) < 1:
        top_curves.append(tag)
    elif abs(y - (-Ly/2)) < 1:
        bottom_curves.append(tag)

# Assign physical groups
if left_curves:
    gmsh.model.addPhysicalGroup(1, left_curves, 1)
    gmsh.model.setPhysicalName(1, 1, "left")
if right_curves:
    gmsh.model.addPhysicalGroup(1, right_curves, 2)
    gmsh.model.setPhysicalName(1, 2, "right")
if top_curves:
    gmsh.model.addPhysicalGroup(1, top_curves, 3)
    gmsh.model.setPhysicalName(1, 3, "top")
if bottom_curves:
    gmsh.model.addPhysicalGroup(1, bottom_curves, 4)
    gmsh.model.setPhysicalName(1, 4, "bottom")
if hole_curves:
    gmsh.model.addPhysicalGroup(1, hole_curves, 5)
    gmsh.model.setPhysicalName(1, 5, "hole")

gmsh.model.addPhysicalGroup(2, [surf_tag], 100)
gmsh.model.setPhysicalName(2, 100, "domain")

# ========== MESH SIZE FIELD (Refine near hole) ==========
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "CurvesList", hole_curves)
gmsh.model.mesh.field.setNumber(1, "Sampling", 100)

gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", fine_size)
gmsh.model.mesh.field.setNumber(2, "SizeMax", coarse_size)
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", R * 3)

gmsh.model.mesh.field.setAsBackgroundMesh(2)

# ========== GENERATE TRIANGLE MESH ==========
gmsh.option.setNumber("Mesh.Algorithm", 6)  # Frontal-Delaunay
gmsh.option.setNumber("Mesh.MeshSizeFromPoints", 0)
gmsh.option.setNumber("Mesh.MeshSizeExtendFromBoundary", 0)
gmsh.option.setNumber("Mesh.MeshSizeFromCurvature", 0)

gmsh.model.mesh.generate(2)

# ========== QUALITY EXTRACTION (Gmsh API) ==========
elem_types, elem_tags, _ = gmsh.model.mesh.getElements(dim=2)
all_tags = []
for tags in elem_tags:
    all_tags.extend(tags)

num_elements = len(all_tags)
qualities = gmsh.model.mesh.getElementQualities(all_tags)
min_quality = float(np.min(qualities))
avg_quality = float(np.mean(qualities))

gmsh.write("mesh/mesh.msh")
gmsh.finalize()

# ========== VISUALIZATION ==========
import meshio
m = meshio.read("mesh/mesh.msh")
pts = m.points[:, :2]
tri_cells = m.cells_dict.get("triangle", [])

if len(tri_cells) == 0:
    raise RuntimeError("No triangles generated!")

fig, ax = plt.subplots(figsize=(12, 6))
ax.set_aspect('equal')
ax.triplot(pts[:, 0], pts[:, 1], tri_cells, linewidth=0.3, color='black')
ax.plot(R * np.cos(np.linspace(0, 2*np.pi, 100)), 
        R * np.sin(np.linspace(0, 2*np.pi, 100)), 'r-', linewidth=1)
ax.set_xlabel("x (mm)")
ax.set_ylabel("y (mm)")
ax.set_title(f"Triangle Mesh: {num_elements} elements")
ax.set_xlim(-Lx/2 - 10, Lx/2 + 10)
ax.set_ylim(-Ly/2 - 10, Ly/2 + 10)
plt.tight_layout()
plt.savefig("mesh/mesh_visualization.png", dpi=150)
plt.close()

# ========== WRITE QUALITY JSON ==========
quality_data = {
    "num_elements": num_elements,
    "num_nodes": len(pts),
    "element_type": "triangle",
    "meshing_method": "frontal_delaunay_triangles",
    "min_quality": round(min_quality, 4),
    "avg_quality": round(avg_quality, 4),
    "mesh_ready_for_solver": True,
    "fenics_compatible": True
}

with open("mesh/mesh_quality.json", "w") as f:
    json.dump(quality_data, f, indent=2)

# ========== WRITE MARKER MAP ==========
marker_map = {
    "left": 1,
    "right": 2,
    "top": 3,
    "bottom": 4,
    "hole": 5,
    "domain": 100
}

with open("mesh/marker_map.json", "w") as f:
    json.dump(marker_map, f, indent=2)

print("Mesh generation complete!")
print(f"Elements: {num_elements} triangles")
print(f"Quality: min={min_quality:.4f}, avg={avg_quality:.4f}")
```

## DOLFINx Triangle Mesh Loading

```python
# In solver code (DOLFINx 0.10.0)
from dolfinx.io import gmsh as gmsh_io
from mpi4py import MPI

mesh, cell_tags, facet_tags = gmsh_io.read_from_msh("mesh/mesh.msh", MPI.COMM_WORLD, gdim=2)
```

## Triangle Workflow Summary

1. Use OCC kernel for boolean operations
2. `occ.cut()` to subtract hole from plate
3. Classify boundaries by position
4. Set up Distance + Threshold field for refinement
5. Use Algorithm 6 (Frontal-Delaunay)
6. **DO NOT recombine** - keep triangles
7. Use `getElementQualities(tags)` for quality (NOT `getElementQuality`)
8. DOLFINx reads .msh directly - no XDMF needed
