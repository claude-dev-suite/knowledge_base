# Terrain Hierarchy Walkthrough

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/tile-design`.
> Canonical source: RPG Maker autotile docs, Tiled terrain editor manual.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/tile-design/SKILL.md

## Concept

When more than two terrains can meet (grass / dirt / stone / sand / water),
you cannot paint every pairwise transition - that scales as N*(N-1)/2 sets of
~13 transition tiles. A hierarchy collapses this: terrains are ordered, only
adjacent levels in the hierarchy get authored transitions, and non-adjacent
pairs render through the chain.

## The hierarchy ordering rule

Order terrains from "lowest" to "highest". A higher terrain "covers" a lower
one. When a stone tile sits next to a grass tile, the rendering pipeline
discovers there is no grass-stone transition and falls back: it draws grass
under stone, then a dirt strip between (because dirt sits between them in the
hierarchy), then stone. Three render passes, all using authored transitions.

```
Level 0:  water     (lowest)
Level 1:  sand
Level 2:  grass
Level 3:  dirt
Level 4:  stone     (highest)
```

This requires: water-sand, sand-grass, grass-dirt, dirt-stone. Four
transition sets instead of ten.

## Authoring checklist per transition pair

For each adjacent pair `(A, B)` where A is lower, paint:

```
Quantity   Tile type         Notes
4          Cardinal edges    A on N, E, S, W (mirror to get all 4)
4          Outer corners     B convex into A
4          Inner corners     A wrapping B (concave)
1          T-junction        A on 3 sides, B fills - often skipped
1          Cross / interior  rare, sometimes auto-derived
~14 total  per pair          plus 3-5 variants of B interior
```

For 4 pairs that is ~56 transition tiles + 4 base interiors with variants =
roughly 80 painted tiles to cover 5 terrains with full hierarchy. Compared to
~130 tiles for full pairwise.

## Worked example: grass meets stone

Map cell wants grass NW, stone center, grass everywhere else. Hierarchy is
grass(2) -> dirt(3) -> stone(4).

```
Pass 1:  draw grass interior tile at center (level 2 base)
Pass 2:  draw dirt as if it sat at center, with grass-dirt transition
         (autotile rules see dirt center, grass on N/W/S/E -> outer corner)
Pass 3:  draw stone at center, with dirt-stone transition (same logic)

Visual result: stone island, thin dirt ring around it, grass beyond.
This is why stone "stains" appear on grass when designers don't think
about hierarchy distance.
```

If you don't want the dirt ring, paint a direct grass-stone transition (extra
13 tiles) OR change the level design so grass and stone never touch.

## Tiled terrain editor workflow

Tiled has a "Terrain" panel that automates hierarchy:

```
1. Open tileset, click Terrain tool.
2. Define terrain types in order: water, sand, grass, dirt, stone.
3. For each tile in tileset, paint its 4 corners with the terrain type
   that occupies that corner.
   (a tile that is "all grass" has all 4 corners marked grass)
   (a corner-tile NW has NW corner = stone, other 3 = grass)
4. Tiled now uses the corner data to pick the right tile when you paint
   a region. Conflicts resolve via hierarchy: paint stone over grass,
   the grass-dirt-stone chain renders if you painted the dirt tiles.
```

LDtk version: use IntGrid layers. Paint level integers (1=grass, 2=dirt, 3=stone).
Auto-Layer rules pattern-match `IntGrid value` against neighbors and choose
the visual tile. Hierarchy is implicit in the rule order.

## Decision: single tilemap layer vs stacked layers

Two implementation strategies:

```
Single layer with corner data:
  + cheap on draw calls (1 layer)
  + Tiled native
  - cannot do partial transparency between terrains
  - corner data fragile across exports

Stacked layers (one per terrain):
  + each terrain on its own layer; transitions between adjacent layers use alpha edges
  + cleaner for blending water-sand-shore
  + LDtk Auto-Layer-friendly
  - 5 terrains = 5 tilemap draw calls
  - z-sort discipline required
```

Modern 2D: stacked layers + LDtk Auto-Layer rules. The draw cost is trivial
on any modern GPU; the workflow win is huge.

## Pitfalls

- **Skipping intermediate transition**: dirt-stone undrawn means stone never
  blends to dirt; visual breaks.
- **Hierarchy reversal**: artist re-orders terrain levels mid-project, every
  transition tile must be re-mapped. Lock the order in week 1.
- **3-way meeting points**: grass + dirt + stone all adjacent to one cell.
  Most autotile rule sets don't handle this; the rendering picks one pair
  arbitrarily. Solutions: forbid 3-way junctions in level design, or paint
  explicit 3-way tiles (~6 per triple).
- **Corner-data tilesets that "look right" in Tiled but wrong in engine**:
  the engine importer reads tile flags; if Tiled uses bottom-up Y and your
  engine uses top-down, NW/SE corners flip. Test with a checkerboard tileset
  before painting all 80.
- **Variant frequency on transitions**: don't put variants on transition
  tiles - the pattern-match rule needs deterministic output. Variants belong
  on full-interior tiles only.
- **Forgetting 9-slice for cliff edges**: vertical cliffs where grass meets
  air at the top need a separate "cliff face" tile family, not just the
  grass-dirt transition. Plan a cliff hierarchy parallel to the terrain one.

## References

- Tiled terrain tool manual: https://doc.mapeditor.org/en/stable/manual/using-the-terrain-tool/
- LDtk Auto-Layer rules: https://ldtk.io/docs/general/auto-layers/
- RPG Maker MV autotile reference: https://blog.rpgmakerweb.com/tutorials/anatomy-of-an-autotile/
