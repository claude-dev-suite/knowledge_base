# Normal-Mapped Sprites

> Source: https://docs.unity3d.com/Manual/SecondaryTextures.html

Normal-mapped sprites pop with bumpy 3D-looking lighting from 2D Lights — great for hero objects, atmospheric scenes, lit dynamic decals.

## Author the normal map

Tools:

| Tool | Use |
|---|---|
| **Sprite Lamp** | Standalone editor — paint shading direction, derive normal map |
| **NormalPainter** (Unity asset) | Paint normals directly in the editor |
| **Aseprite** | Export normal map via the dedicated workflow (per-layer light direction) |
| **CrazyBump / Substance Sampler** | Photo-derived normal maps |

Standard convention: normal maps in **OpenGL** orientation (Green = up). Unity reads them as `+Y` up.

## Hook the normal map to a sprite

Sprites in Unity support **Secondary Textures** — extra textures shipped alongside the main sprite, accessible by shader by name.

```
Sprite Editor → Secondary Textures → +
  Name:    _NormalMap
  Texture: drag your normal-map texture
```

The default Sprite-Lit shader (URP 2D Renderer's default for sprites) reads `_NormalMap` automatically.

## Material setup

```
Sprite Renderer
  Material: Sprite-Lit-Default (URP)
  (no manual texture assignment — comes via the sprite's Secondary Texture)
```

For custom Sprite-Lit shaders, sample the secondary texture in Shader Graph:

```
Sample Texture 2D node
  Texture: (passed via Secondary Texture name "_NormalMap")
  Type:    Normal
  Space:   Tangent
```

## Light interaction

For each Light 2D you want to interact with the normal map:

```
Light 2D
  Use Normal Map  ON
  Normal Map Quality
    Disabled
    Accurate    (default)
    Fast        (lower per-pixel cost)
```

`Fast` halves the per-pixel work — usually visually fine for ambient lighting; reserve `Accurate` for hero sprites under direct lights.

## Lit vs Unlit shader choice

| Sprite | Material |
|---|---|
| Background tilemaps (don't react to lights) | `Sprite-Unlit-Default` — cheapest, no lighting cost |
| Characters / hero objects (do react) | `Sprite-Lit-Default` |
| Effects (additive, fully lit by their own color) | `Sprite-Unlit-Default` with HDR color |

Mix freely — lit and unlit sprites coexist in the same scene; only lit ones consume the lighting pass.

## Performance

Lit sprites with normal maps cost ~2x more per pixel than unlit. Profile fillrate before lighting up everything. Mobile target: keep `Use Normal Map` to a small subset.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Wrong normal map orientation (DirectX vs OpenGL) | Re-export with Y inverted, or set Texture Type = Normal Map and tick "Flip Green" |
| Normal map not assigned via Secondary Texture | Use Sprite Editor → Secondary Textures (NOT material slot) |
| Lit shader on background tilemaps | Switch to Unlit for cheap rendering |
| Very high-detail normal maps on small sprites | Aliasing — use lower-res normals or smooth them |
