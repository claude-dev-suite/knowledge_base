# Procedural Tilemap Generation

> Source: https://docs.unity3d.com/ScriptReference/Tilemaps.Tilemap.SetTilesBlock.html

For runtime-generated levels (caves, dungeons, terrain), use `SetTilesBlock` for batch writes — `SetTile` per cell is orders of magnitude slower for large maps.

## SetTilesBlock — the only fast path

```csharp
public void GenerateChunk(Tilemap map, Vector3Int origin, int width, int height, TileBase[] grid) {
    // grid is a flat array of size width*height; layout: y * width + x
    var bounds = new BoundsInt(origin.x, origin.y, 0, width, height, 1);
    map.SetTilesBlock(bounds, grid);
}
```

Build the array once, write once. For a 256x256 cave, this is ~50x faster than per-cell `SetTile`.

## Cellular automata caves

```csharp
public TileBase[] GenerateCave(int w, int h, int seed, float fillPct, int smoothIterations) {
    var rng = new System.Random(seed);
    var grid = new bool[w * h];
    for (int i = 0; i < grid.Length; i++) grid[i] = rng.NextDouble() < fillPct;

    for (int it = 0; it < smoothIterations; it++) {
        var next = new bool[grid.Length];
        for (int y = 1; y < h - 1; y++)
        for (int x = 1; x < w - 1; x++) {
            int n = 0;
            for (int dy = -1; dy <= 1; dy++)
            for (int dx = -1; dx <= 1; dx++)
                if (dx != 0 || dy != 0) if (grid[(y+dy) * w + (x+dx)]) n++;
            next[y * w + x] = n >= 5;        // majority becomes wall
        }
        grid = next;
    }

    var tiles = new TileBase[w * h];
    for (int i = 0; i < grid.Length; i++) tiles[i] = grid[i] ? wallTile : floorTile;
    return tiles;
}
```

5 iterations of smoothing — cave-like rooms with rough but coherent shapes. Tune `fillPct` (0.4–0.5) and iterations (3–7).

## Chunked streaming

For huge worlds: divide into 64x64 chunks, generate / load on player proximity, unload distant ones. Use Addressables to ship per-chunk Tile palettes if their content varies.

```csharp
void Update() {
    var playerCell = WorldToCell(player.position);
    foreach (var chunkCoord in NeighborsWithin(playerCell, radius: 3)) {
        if (!loadedChunks.ContainsKey(chunkCoord)) LoadChunk(chunkCoord);
    }
    foreach (var loaded in loadedChunks.Keys.ToList()) {
        if (Distance(loaded, playerCell) > unloadRadius) UnloadChunk(loaded);
    }
}
```

## Composite regen

After procgen, regenerate the composite collider once:

```csharp
groundComposite.GenerateGeometry();
```

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `SetTile` in a per-cell loop | `SetTilesBlock` |
| Regenerating composite on every tile write | Batch + single regen |
| No chunking on huge worlds | Stream chunks based on player proximity |
| Random seed not exposed | Always allow seed for reproducibility |
