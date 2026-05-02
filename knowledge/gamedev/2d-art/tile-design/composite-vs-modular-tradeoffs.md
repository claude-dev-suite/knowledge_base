# Composite vs Modular Tilesets - Tradeoffs

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/tile-design`.
> Canonical source: Dead Cells level design talks (Sebastien Benard), LDtk docs.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/tile-design/SKILL.md

## Concept

A modular tileset = every map cell is one tile. A composite tileset = base
modular layer plus larger non-grid sprites painted on top. Production
pipelines, collider authoring, and Z-sorting all change depending on which
ratio you pick. This article gives concrete cost numbers and a hybrid
template that scales.

## Pure modular

Every cell snaps to grid. Engine handles autotile blending. Collider derives
from per-tile shapes.

```
Cost per terrain (47-tile blob):    ~2-3 days hand-paint, or ~4h with Tilesetter
Memory:                             1 atlas, ~64KB at 16x16x47 tiles
Collision authoring:                per-tile shape in tileset editor
Z-sort:                             implicit (single layer)
Repetition:                         visible without 3-5 variants per interior
```

Strengths: cheap, deterministic, autotileable. Weakness: world feels grid-y.
Players see the cells.

## Pure composite (decoration overlay only)

Every visual is a free-floating sprite at arbitrary world position. No grid.

```
Cost per scene:                     5-10x modular (each scene hand-composed)
Memory:                             huge atlas of unique decor sprites
Collision authoring:                per-sprite polygon collider
Z-sort:                             explicit y-sort or manual z-order
Repetition:                         imperceptible (every screen unique)
```

Strengths: organic, painterly, no grid feel. Weakness: not autotileable; level
design becomes painting; swapping a tile-type globally is impossible.

## Hybrid: modular base + composite overlay (the standard)

Most modern indie 2D (Dead Cells, Hollow Knight, Hyper Light Drifter) layers:

```
Layer 0: modular ground tilemap (collision lives here)
Layer 1: modular overlay tilemap (cracks, edges, inset details, no collision)
Layer 2: composite decor sprites (trees, statues, broken pillars, multi-tile)
Layer 3: foreground sprites (vines hanging, bushes in front of player)
```

Collision lives ONLY on Layer 0. Layers 1-3 are visual. Z-sort: Layer 2 is
y-sorted with the player; Layer 3 is always above the player.

## Worked example: forest scene budget

Target: 1 forest biome in a metroidvania, 60 screens of content.

```
Pure modular budget:
  - 47 base grass tiles + 47 dirt + 47 stone + 5 variants of interior
    each = ~150 tiles painted hand
  - 60 screens designed: 6 days
  - Total: ~3 weeks for 1 artist

Pure composite budget:
  - 60 unique screens, each with ~30 hand-placed elements
  - ~5-10 hours per screen
  - Total: ~12-15 weeks - DEAD ON ARRIVAL for one artist

Hybrid budget:
  - Modular base: 47 grass + 47 dirt = ~94 tiles, 4 days
  - 30-40 decor sprites (tree trunks 3 variants, branches, stumps,
    rocks 5 sizes, ferns, mushrooms, fallen logs)
  - 60 screens designed using base + decor library: 2 days
  - Total: ~7-8 days
```

Hybrid hits the 80/20 sweet spot.

## Decor sprite size budget

```
Decor type          Size (px)      Count per scene    Memory cost
Small accent        8-16 px        20-40              tiny
Medium prop         32-48 px       8-15               low
Large landmark      64-128 px      2-4                medium
Hero prop           128-256 px     0-1 (signature)    notable
```

Cap landmarks at 4 per screen or the eye loses focal point. Stardew Valley
averages 1-2 hero props per screen (the well, the shipping bin).

## Z-sort for composite

For overlapping decor + character, two options:

```
Option A: Y-sort by sprite pivot Y
  pivot at sprite "feet" (bottom-center for character, base of trunk for tree)
  sorted layer index = floor(world_y_in_pixels)
  In Unity URP 2D: SpriteRenderer with "Custom Axis" = Y on Sorting Group
  In Godot: YSort node + each sprite's pivot at base

Option B: hand-assigned z-order
  designer picks z per asset
  fragile, breaks on unanticipated overlaps
```

Always go A unless the camera is fixed-angle and you can guarantee no novel
overlaps.

## Pitfalls

- **Collision on decor layer**: artists place a tree, player walks through it
  because the tree is on Layer 2 (no collision). Solution: drop a small
  invisible blocker on Layer 0 under each large decor.
- **Modular overlay misaligned**: Layer 1 inset details snap to Layer 0 grid;
  if you import overlay at half-pixel offset every detail looks 1px off.
  Lock both to the same import PPU.
- **Tree trunks Y-sort wrong**: pivot at center of sprite means tree appears
  to occlude the player when player is at trunk base. Pivot must be at trunk
  base.
- **Decor count creep**: artists keep adding "one more rock variant", atlas
  blows past 4096x4096, mobile build breaks. Set a hard cap at sprint start.
- **Hand-placed decor in random rooms**: if the game has procedural
  generation, decor placement must be rule-based (LDtk Auto-Layer rules
  trigger on IntGrid patterns) or the rooms feel sterile.
- **Foreground sprites occlude UI**: foreground is always-on-top in world
  space, but UI lives in screen space - keep them on different cameras.

## References

- Sebastien Benard "How to make a metroidvania" (LDtk creator, GDC talks)
- Hyper Light Drifter art breakdown (Heart Machine devlogs)
- LDtk Auto-Layer rules manual: https://ldtk.io/docs/general/auto-layers/
