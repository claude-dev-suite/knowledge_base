# Composite Collider on Tilemaps

> Source: https://docs.unity3d.com/Manual/class-CompositeCollider2D.html

Per-cell colliders on a 100x50 tilemap = 5000 small physics objects. Composite Collider collapses them into one polygon — cheap broadphase, fewer queries.

## Setup

```
Tilemap_Ground (GameObject)
  Tilemap                       (Grid child)
  TilemapRenderer
  TilemapCollider2D             Used By Composite = ON
  Rigidbody2D                   Body Type = Static
  CompositeCollider2D           Geometry Type = Polygons (or Outlines)
```

`Used By Composite = ON` on TilemapCollider2D is the trigger — it tells the composite to absorb the per-cell shapes.

`Body Type = Static` on Rigidbody2D — required, and tells the physics engine the geometry never moves.

## Geometry types

| Geometry | Use |
|---|---|
| **Outlines** | Hollow shape (perimeter only) — for one-way platforms with edge effectors |
| **Polygons** | Solid filled shape — standard for ground, walls |

## When the tilemap changes at runtime

`compositeCollider.GenerateGeometry()` after batch tile writes. Don't trigger geometry regen per-tile in a loop — batch then regen.

```csharp
[SerializeField] private CompositeCollider2D composite;
[SerializeField] private Tilemap tilemap;

public void DigBlock(Vector3Int cell) {
    tilemap.SetTile(cell, null);
}
public void CommitDig() {
    composite.GenerateGeometry();   // expensive — call once after batch
}
```

## Layer + tag advice

Put the colliding tilemap on a `Ground` layer, decorations on `BackgroundDeco` (no collider). The layer collision matrix culls Ground-vs-Ground pairs (tile self-collision is impossible by definition).

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Per-cell colliders without composite | Used By Composite ON + add CompositeCollider2D |
| Missing Static Rigidbody2D | Required — composite needs the rb to anchor |
| GenerateGeometry per tile change | Batch tile writes, then one regen |
| Painting collision and decoration on same tilemap | Two tilemaps: gameplay + cosmetic |
