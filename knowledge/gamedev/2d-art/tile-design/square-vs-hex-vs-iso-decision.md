# Square vs Hex vs Isometric - Decision Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/tile-design`.
> Canonical source: Red Blob Games hex grid reference (https://www.redblobgames.com/grids/hexagons/), Amit Patel iso (https://www.redblobgames.com/grids/isometric/).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/tile-design/SKILL.md

## Concept

Grid choice locks in art cost, gameplay rules, and the engine code path for the
remainder of the project. Switching from square to hex after week 4 means
re-painting every transitional tile, rewriting movement, redoing pathfinding
weights, and re-authoring every multi-tile prop. The decision is a 5-axis
trade: art budget, movement model, range/area math, depth perception, and
tooling support.

## Decision matrix

| Axis                       | Square (ortho)             | Hex (flat or pointy)     | Staggered iso          | True iso (Z-sort)      |
|----------------------------|----------------------------|--------------------------|------------------------|------------------------|
| Art cost (base interior)   | 1x                         | 1x                       | 2x (2-3 facings)       | 3-4x (4 facings)       |
| Transition tiles           | 47 (RPGMaker)              | ~13 per neighbor pair    | ~47 plus depth seams   | ~47 per facing         |
| Diagonal movement          | Ambiguous (sqrt(2) hack)   | All neighbors equidistant| Ambiguous              | Ambiguous              |
| Range/AoE math             | Chebyshev or Manhattan     | Cube/axial - clean       | Chebyshev on offset    | Chebyshev on offset    |
| Pathfinding cost           | A* with 4 or 8 dirs        | A* with 6 dirs           | A* + Y-sort            | A* + Z-sort + occluder |
| Depth/parallax feel        | Flat                       | Flat                     | Pseudo-3D              | Strong pseudo-3D       |
| Mod/UGC friendly           | Easiest                    | Medium                   | Hard                   | Hardest                |
| Engine native support      | Unity/Godot/Tiled/LDtk all | Tiled/Godot, LDtk weak   | Tiled/Godot/Unity      | Manual y-sort everywhere|

## When square wins

- Platformers: gravity is axis-aligned, slopes are rotational variants of squares.
- 8-bit/16-bit homage: visual signal of retro is square pixels.
- Top-down twin-stick shooters: aim is continuous, grid is just collision.
- Rule of thumb: if movement is real-time and pixel-precise, square.

## When hex wins

- Turn-based tactics where line-of-sight, range rings, and area-of-effect are
  central mechanics. Cube coordinates `(x+y+z=0)` make distance the trivial
  `max(|dx|,|dy|,|dz|)`. With square + Chebyshev, a "5-tile fireball" looks
  like a square, which feels wrong for a circle.
- Strategy/4X with adjacency-based bonuses (Civ tile yields).
- Board-game-feel UI without diagonal awkwardness.
- Pointy-top hex if you want east-west reading; flat-top if you want
  north-south columns (mountain ranges, river systems read better).

## When isometric wins

- City-builder, dungeon-crawler, tactics RPG that wants visible depth without
  3D tooling.
- Verticality matters: Z-stacked floors, ramps, walls that occlude.
- Choose **staggered iso** (a.k.a. dimetric) for cheaper authoring: tile is
  stored as 2:1 diamond on a rectangular grid, sorted by row.
- Choose **true iso** if multiple props at same Y need explicit z-sort.

## Worked decision: a tactics RPG with flying units

Constraints: turn-based, range = "all tiles within 4", flying ignores terrain.

```
Square + Chebyshev:  range-4 = 9x9 square. AoE shapes feel angular.
Square + Manhattan:  range-4 = diamond. Diagonals cost 2 - feels weird.
Hex + cube coords:   range-4 = clean hex ring. Distance = (|dq|+|dr|+|ds|)/2.
                     6 movement directions, all equidistant. Pick this.
Iso (any):           depth feels great but you pay 2-3x art and z-sort.
                     Overkill for a flat tactical battlefield.
```

Verdict: pointy-top hex with axial coords `(q, r)`. Movement uses
`(+1,0), (-1,0), (0,+1), (0,-1), (+1,-1), (-1,+1)`. Distance:
`(|dq| + |dr| + |dq+dr|) / 2`.

## Pitfalls in conversion

- **Hex offset vs axial coordinate confusion**: store offset for the editor
  (rectangular array), convert to axial/cube for math. Mixing them breaks
  pathfinding silently.
- **Square diagonal cost = 1**: enables players to move 41% faster on
  diagonals. Either disallow diagonals, charge sqrt(2), or bite the bullet.
- **Iso tile pivot mismatch**: half the team draws pivot at top of diamond,
  other half at bottom-center. Standardize: bottom-center is canonical for
  staggered iso (matches y-sort feet).
- **Hex tilesets in Aseprite**: tilemap mode is square-grid only. You must
  author hex tiles as offset rectangles and let the engine offset rows/cols.
  See engine docs for stamp mode.
- **Mixing iso props with ortho UI**: floating UI cards at iso angle look
  drunk. Keep UI screen-space, art world-space.
- **Late-stage hex switch**: nearly impossible. Lock the decision in
  pre-production.

## References

- Red Blob Games hex guide: https://www.redblobgames.com/grids/hexagons/
- Red Blob Games isometric: https://www.redblobgames.com/grids/isometric/
- Civilization V hex transition postmortem (Soren Johnson, GDC 2010)
- Tiled hex docs: https://doc.mapeditor.org/en/stable/manual/using-the-terrain-tool/
