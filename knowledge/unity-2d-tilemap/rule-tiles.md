# Rule Tiles — Auto-Tiling

> Source: https://docs.unity3d.com/Packages/com.unity.2d.tilemap.extras@latest/manual/RuleTile.html

Rule Tiles select a sprite at paint-time based on neighbor cells, eliminating per-tile manual painting of edges and corners.

## Install

Package Manager — `com.unity.2d.tilemap.extras` (in addition to the standard `com.unity.2d.tilemap`).

## Create

`Assets > Create > 2D > Tiles > Rule Tile`. Drag the asset into the Tile Palette.

## Rule definition

Each rule has a 3x3 neighbor matrix:

```
[ NW ][ N  ][ NE ]      Each cell = Filled / Empty / Don't Care
[ W  ][ ME ][ E  ]      ME = the tile being painted
[ SW ][ S  ][ SE ]
```

Plus a sprite (or animation, or random pool) for that rule. The first matching rule wins (top-down evaluation).

## Standard ground tileset rules

```
Top empty + sides full         -> top edge sprite
Bottom empty + sides full      -> bottom edge sprite
Left empty + others full       -> left edge sprite
Right empty + others full      -> right edge sprite
Top + left empty               -> top-left corner
Top + right empty              -> top-right corner
Bottom + left empty            -> bottom-left corner
Bottom + right empty           -> bottom-right corner
All four neighbors empty       -> solo block
All four full                  -> interior fill
(else)                         -> fallback default
```

Order matters — corners must be evaluated before edges before interior, otherwise the wrong sprite wins.

## Animated Rule Tiles

`Assets > Create > 2D > Tiles > Animated Rule Tile` — same matching system but each rule can hold a frame sequence (water, lava, conveyor).

## Hexagonal & Isometric Rule Tiles

`com.unity.2d.tilemap.extras` provides `Hexagonal Rule Tile` and `Isometric Rule Tile` variants — they use 6 / 4 neighbors instead of 8 to match the grid topology.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Rule order interior-before-corner | Reorder; corners must match first |
| Hand-painting edges instead of using rules | Define the rule once; saves hours and keeps consistency |
| Mixed rule + manual paints on the same tilemap | Use different tilemaps for hand-decorated areas |
| Rule Tile on hexagonal grid using 8-neighbor logic | Use Hexagonal Rule Tile |
