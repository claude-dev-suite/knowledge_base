# Normal Map Pixel Art Pipeline - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/seamless-textures`.
> Canonical source: Sprite Lamp documentation; Owlboy normal map workflow
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/seamless-textures/SKILL.md

## Concept

Normal maps add **dynamic 3D-ish lighting** to flat 2D sprites. Each
pixel encodes a 3D surface normal in RGB (R=X, G=Y, B=Z, with #8080FF
= flat). Engine 2D Lights respect normal maps to compute per-pixel
shading. Authoring options range from full hand-paint (highest quality,
slowest) to procedural generation from diffuse (fastest, lowest
quality).

## Walkthrough / mechanics

### Normal map color encoding

```
Pixel = (R, G, B):
  R = (X + 1) / 2 * 255    X axis component
  G = (Y + 1) / 2 * 255    Y axis component
  B = Z / 1 * 255          Z axis component (must be positive)

Flat surface (normal pointing at camera):
  X=0, Y=0, Z=1 -> (128, 128, 255) = #8080FF
```

| Direction | Normal RGB | Hex |
|-----------|------------|-----|
| Flat (up at camera) | (0, 0, 1) | #8080FF |
| Tilted left | (-1, 0, 0.5) | #006080 (cyan-ish) |
| Tilted right | (1, 0, 0.5) | #FF6080 (magenta-ish) |
| Tilted up (top toward camera) | (0, 1, 0.5) | #80FF80 (green-ish) |
| Tilted down | (0, -1, 0.5) | #800080 (purple-ish) |

### Authoring options

#### 1. Sprite Lamp (semi-automatic, $15)

Provide:
- Diffuse texture
- 4 lighting passes: top-lit, bottom-lit, left-lit, right-lit (you
  paint each by re-shading the diffuse with a specific light direction)

Sprite Lamp computes the normal map from the difference between
lighting passes. Quality: very good. Used by Owlboy, Dust: An
Elysian Tail.

Time: ~30 min per sprite (4 lighting passes + import).

#### 2. Sprite DLight (procedural, $15)

Provide:
- Diffuse texture only

Sprite DLight uses height-from-shading heuristic: bright pixels =
"out", dark = "in". Generates normal that bumps brightness up toward
camera, darkness away. Quality: medium, often "good enough".

Time: 30 seconds per sprite.

#### 3. Materialize (free, FOSS)

Cross-platform tool. Generates normal + height + AO + roughness from a
diffuse. Less specialized for pixel art (designed for 3D PBR), but
works.

Time: ~5 min per sprite (manual tweaking).

#### 4. Hand-painted (highest quality, slowest)

Paint the normal map directly. Time: equal to the diffuse painting time
(maybe more). Used for hero sprites where quality matters most.

Workflow:
1. Open diffuse.
2. New layer named "normal".
3. Fill with #8080FF (flat).
4. For each surface that should appear tilted: paint with the
   appropriate hue (see table above).
5. Use Aseprite "Convolution Matrix" filter for blur to smooth
   transitions.

## Worked example

Authoring a normal map for a 32x32 mushroom sprite (Sprite Lamp
workflow):

1. Diffuse already done (mushroom on white bg, 32x32).
2. Duplicate diffuse 4x. For each copy, paint shading as if light
   comes from:
   - Top: shadow on bottom of cap, bright on top
   - Bottom: shadow on top of cap, bright on bottom
   - Left: shadow on right side, bright on left
   - Right: shadow on left, bright on right
3. Save 5 PNGs (diffuse + 4 lighting) and feed to Sprite Lamp.
4. Sprite Lamp outputs normal_map.png and ambient_occlusion.png.
5. Import normal_map.png into engine; assign as Sprite Renderer's
   "Secondary Texture: _NormalMap" in Unity URP 2D.

For pure pixel art at 32x32: pixel-by-pixel resolution may look
"noisy". Apply gentle blur to normal map (1-2 px Gaussian) to smooth
transitions.

## Pixel art specific concerns

### Pixel-by-pixel normals = noisy
A 16x16 sprite has 256 pixels. Each pixel having a slightly different
normal = noisy lighting that flickers as light moves.

Solution: blur the normal map (gentle 1-2 px). Or paint flat regions
of the sprite with uniform #8080FF (and only paint normal variation at
silhouette / form edges).

### Z-component (B channel)
Always positive. Use 128-255 (no values below 128). Negative Z would be
"surface facing away from camera" which doesn't make sense for sprites.

### Sub-pixel motion artifacts
If sprite moves at sub-pixel speed AND has normal map AND has 2D
Light, the lighting can shimmer between frames. Anchor sprite position
to integer pixel.

## Engine integration

### Unity URP 2D
1. Add Normal Map Render Feature to URP Renderer Data.
2. Sprite Renderer → Sprite Editor → Custom Outline / Custom Physics
   Shape (for shadow casting).
3. In the Sprite Importer's Secondary Textures, add entry: name
   "_NormalMap", drag your normal map.
4. Light 2D will respect the normal map.

### Godot 4
1. Sprite2D → Material → New ShaderMaterial → custom canvas shader
   sampling normal_texture.
2. Or use built-in Light2D + Normal Map texture parameter on
   CanvasItemMaterial.

### Custom (raylib, SDL, etc.)
Standard tangent-space normal lighting in fragment shader:
```glsl
vec3 N = texture(normalMap, uv).rgb * 2.0 - 1.0;
vec3 L = lightPos.xyz - fragPos.xyz;
float lambert = max(dot(N, normalize(L)), 0.0);
fragColor = diffuse * (ambient + lambert * lightColor);
```

## Common pitfalls

- **G channel inverted (DirectX vs OpenGL convention)**: light comes
  from "wrong" direction. Most engines have a flip toggle. Flip Y in
  the importer or in the shader.
- **Z channel < 128**: pixels render as black or strange artifact.
  Always Z >= 128.
- **Normal map blurry while diffuse is sharp**: misaligned dimensions.
  Both must be the same pixel resolution.
- **Atlas without normal map atlas**: sprite atlas is the wrong size
  for normal lookup. Both atlases must have identical layout
  (TexturePacker can pack a "secondary atlas" matching layout).
- **Hand-painted normal that doesn't look right**: probably mixed up
  R-X axis. Test with a single light and verify direction.
- **Sub-pixel motion artifacts**: integer-only sprite positioning.

## References

- Sprite Lamp official: https://www.spritelamp.com/
- Sprite DLight: https://2deegamestudio.com/spritedlight.html
- Materialize FOSS: https://boundingboxsoftware.com/materialize/
- Owlboy normal map workflow blog post (D-Pad Studio)
- Unity URP 2D Renderer: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/2d-lighting.html
