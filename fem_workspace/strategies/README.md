# Meshing Strategies

## ⚠️ CRITICAL RULES

- ❌ **NEVER use `geo.addSpline()` for straight edges** → Creates curved boundaries!
- ✅ **USE OCC kernel** (`gmsh.model.occ`) for boolean operations
- ✅ **USE `occ.addRectangle()` + `occ.addDisk()` + `occ.cut()`** for plate+hole
- ✅ **USE Algorithm 8 + RecombineAll** for quad meshing

---

## Quick Decision

```
What geometry?
  │
  ├─ Simple rectangle (no holes)
  │   └─ Use TRANSFINITE QUADS (1 block)
  │      (See: geometry_patterns/rectangle.md)
  │
  └─ Rectangle + hole / Complex
      └─ Use OCC BOOLEAN + ALGORITHM 8 (quads)
         (See: geometry_patterns/plate_with_hole.md)
         (See: quad_meshing/PARTITIONING_WORKFLOW.md)
```

---

## Strategy Files

### Main Workflow
| File | Description |
|------|-------------|
| `quad_meshing/PARTITIONING_WORKFLOW.md` | **Main reference** - includes BOTH quad and triangle workflows |

### Geometry Patterns  
| File | Geometry | Recommended Method |
|------|----------|---------------------|
| `geometry_patterns/rectangle.md` | Simple rectangle | Transfinite quads |
| `geometry_patterns/plate_with_hole.md` | Rectangle + hole | **OCC Boolean + Algorithm 8** |
| `geometry_patterns/l_shape.md` | L-shape | OCC Boolean + Algorithm 8 |

---

## Element Type Decision Table

| Geometry | Element | Method | Complexity |
|----------|---------|--------|------------|
| Rectangle (no hole) | Quad | Transfinite | Easy |
| Rectangle + 1 hole | **Quad** | OCC + Algorithm 8 | Easy |
| Rectangle + 2+ holes | **Quad** | OCC + Algorithm 8 | Easy |
| L-shape | **Quad** | OCC + Algorithm 8 | Easy |
| Complex | **Quad** | OCC + Algorithm 8 | Easy |

**Note:** If solver (FEniCS 2019) gives "non-orderable quads" error, use DOLFINx or switch to triangles.

---

## Physical Group Convention

| Marker | Boundary |
|--------|----------|
| 1 | left |
| 2 | right |
| 3 | top |
| 4 | bottom |
| 5 | hole |
| 100 | domain (surface) |
