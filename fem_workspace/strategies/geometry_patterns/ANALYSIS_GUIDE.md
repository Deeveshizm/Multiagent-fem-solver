# Geometry Analysis Guide: How to Plan Blocks for ANY Shape

## Step-by-Step Decision Process

### 1. Identify the Base Shape

| Base Shape | Starting Blocks |
|------------|-----------------|
| Rectangle | 1 |
| L-shape | 3 |
| T-shape | 5 |
| U-shape | 5 |
| Ring/Annulus | 4 |

### 2. Count Internal Features

Each feature adds complexity:

| Feature | Additional Blocks |
|---------|------------------|
| Circular hole | +4 (minimum) |
| Rectangular cutout | +4 |
| Edge notch | +2 to +4 |
| Re-entrant corner | +1 (partition needed) |

### 3. Identify Singular Points

**Singular points** are where partitions MUST originate or terminate:

- Hole centers (conceptually) → radial partitions
- Re-entrant corners → partitions extend outward
- Sharp internal corners → stress concentration zones

```
Example - L-shape with hole:

    ┌──────────────┐
    │      ○       │  ← hole creates 4 singular junction points
    │              │
    ├──────┬───────┘
    │      │ ← re-entrant corner (singular)
    │      │
    └──────┘

Singular points: 5 (4 around hole + 1 at corner)
Minimum blocks: 3 (L-shape) + 4 (hole) = 7 blocks
```

### 4. Draw Partition Lines

**Rule:** Partitions should connect singular points to:
- Outer boundaries
- Other singular points
- Create 4-sided regions

```
Good partition:              Bad partition:
┌─────┬─────┐               ┌───────────┐
│     │     │               │     │     │
├──○──┼─────┤               │  ○──┼─────│  ← doesn't reach boundary
│     │     │               │     │     │
└─────┴─────┘               └───────────┘
```

### 5. Verify 4-Sided Blocks

Count edges for each block. If any has ≠4:
- Add more partition lines, OR
- Merge/split existing partitions

### 6. Decision: Partition or Fallback?

```
                    Geometry Analysis
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        Simple geometry            Complex geometry
        (≤4 blocks, no holes       (>6 blocks, multiple holes,
         or 1 hole)                 non-aligned features)
              │                         │
              ▼                         ▼
        TRY PARTITIONING          USE FALLBACK
        (transfinite)             (Algorithm 8)
              │                         │
              ▼                         ▼
        If fails twice ──────────► Fallback
```

## Quick Reference: Block Counts

| Geometry | Blocks | Complexity | Recommendation |
|----------|--------|------------|----------------|
| Rectangle | 1 | Easy | Partition |
| Rectangle + 1 hole | 4 | Medium | Partition |
| Rectangle + 2 holes | 8+ | Hard | Fallback |
| L-shape | 3 | Medium | Partition |
| L-shape + hole | 7+ | Hard | Fallback |
| T-shape | 5 | Medium-Hard | Partition or Fallback |
| Notched plate | 4-5 | Medium | Partition or Fallback |
| Arbitrary polygon | Varies | Hard | Fallback |
| Curved boundaries | Varies | Hard | Fallback |

## Node Count Propagation

When blocks share edges, node counts propagate:

```
Block A          Block B
┌──────┐        ┌──────┐
│      │        │      │
│  Na  │ shared │  Nb  │
│      │  edge  │      │
└──────┘   ↓    └──────┘
           │
    N_shared must work for BOTH blocks
    (must equal opposite in A AND opposite in B)
```

**Constraint Chain Example:**

```
Block 1: N_top = 10, so N_bottom = 10
Block 2 shares bottom with Block 1's top
Block 2: N_top = 10 (inherited), so N_bottom = 10
... continues through all connected blocks
```

If constraints conflict → need more blocks or different topology.

## Red Flags (Use Fallback Instead)

- More than 8 blocks needed
- Holes at different Y positions
- Curved outer boundaries
- Very thin sections (high aspect ratio)
- Complex boolean operations needed
- Previous partitioning attempts failed

## Summary Checklist

Before implementing:
- [ ] Identified base shape
- [ ] Counted all features (holes, notches, corners)
- [ ] Located all singular points
- [ ] Planned partition lines
- [ ] Verified all blocks are 4-sided
- [ ] Checked node count constraints are satisfiable
- [ ] Decided: partition vs fallback

If unsure → **Use fallback (Algorithm 8)** — it's reliable.
