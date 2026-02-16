# Fallback: Frontal-Delaunay for Quads (Algorithm 8)

**Use this when:** Partitioning has failed twice

## When to Use Fallback

- Partitioning topology is too complex
- Opposite edge constraints can't be satisfied
- Time pressure requires working mesh

## Implementation

```python
import gmsh

gmsh.initialize()
gmsh.model.add("fallback_mesh")
occ = gmsh.model.occ

# 1. Build geometry using OCC (boolean operations work)
plate = occ.addRectangle(x_min, y_min, 0, width, height)
hole = occ.addDisk(cx, cy, 0, radius, radius)
occ.cut([(2, plate)], [(2, hole)])
occ.synchronize()

# 2. Add physical groups for boundary conditions
# ... (classify curves by position)

# 3. Set mesh algorithm to Frontal-Delaunay for Quads
gmsh.option.setNumber("Mesh.Algorithm", 8)              # Key setting!
gmsh.option.setNumber("Mesh.RecombinationAlgorithm", 3) # Blossom full-quad
gmsh.option.setNumber("Mesh.RecombineAll", 1)           # Force all quads
gmsh.option.setNumber("Mesh.Smoothing", 100)            # Polish mesh

# 4. Control mesh size
gmsh.option.setNumber("Mesh.CharacteristicLengthMin", min_size)
gmsh.option.setNumber("Mesh.CharacteristicLengthMax", max_size)

# 5. Optional: Refine near features
gmsh.model.mesh.field.add("Distance", 1)
gmsh.model.mesh.field.setNumbers(1, "CurvesList", [hole_curves])
gmsh.model.mesh.field.add("Threshold", 2)
gmsh.model.mesh.field.setNumber(2, "InField", 1)
gmsh.model.mesh.field.setNumber(2, "SizeMin", fine_size)
gmsh.model.mesh.field.setNumber(2, "SizeMax", coarse_size)
gmsh.model.mesh.field.setNumber(2, "DistMin", 0)
gmsh.model.mesh.field.setNumber(2, "DistMax", transition)
gmsh.model.mesh.field.setAsBackgroundMesh(2)

# 6. Generate
gmsh.model.mesh.generate(2)
gmsh.write("mesh/mesh.msh")
gmsh.finalize()
```

## Key Differences from Partitioning

| Aspect | Partitioning | Fallback (Algo 8) |
|--------|--------------|-------------------|
| Topology control | Full | None |
| Node count control | Exact | Approximate |
| Element alignment | Perfect | Good |
| Reliability | ~30% | ~90% |
| Code complexity | High | Low |

## mesh_quality.json for Fallback

```json
{
  "num_elements": 1234,
  "element_type": "quad",
  "meshing_method": "frontal_delaunay_fallback",
  "min_quality": 0.45,
  "avg_quality": 0.72,
  "mesh_ready_for_solver": true,
  "fenics_compatible": true,
  "note": "Used fallback after partitioning failed"
}
```
