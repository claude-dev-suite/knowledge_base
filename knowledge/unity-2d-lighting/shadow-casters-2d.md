# Shadow Casters 2D

> Source: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/2DShadows.html

In URP 2D, sprites do NOT cast shadows automatically. Add `ShadowCaster2D` components to opt into shadowing.

## Per-sprite Shadow Caster

```
On a sprite that should cast shadows:
  ShadowCaster2D
    Self Shadows  OFF (default)
    Casts Shadows ON
    Use Renderer Silhouette  ON  (auto from sprite alpha)
                              OFF -> Custom Shape with manually drawn polygon
    Sorting Layers           which Sorting Layers receive shadows
```

The light must have `Shadow Intensity > 0` to actually cast.

## Composite Shadow Caster — multi-part characters

A skeletal-rigged character has many SpriteRenderers (head, torso, arms, legs). Adding ShadowCaster2D to each produces incoherent shadow boundaries between parts. Use `CompositeShadowCaster2D`:

```
Knight (CompositeShadowCaster2D)
  Body  (ShadowCaster2D)
  Head  (ShadowCaster2D)
  Arms  (ShadowCaster2D)
  Legs  (ShadowCaster2D)
```

The composite merges children's silhouettes into one outline → clean character shadow.

## Self Shadows

Default OFF. Turn ON when you want a sprite to cast shadow back onto itself (a chimney casting shadow on the building's wall). On flat sprites, self-shadows usually cause z-fighting and look wrong.

## Sorting Layers field

Shadow Caster's Sorting Layers field controls which Sorting Layers it casts onto. Common: a "Background" caster only casts onto "Characters" and "World", not onto "FX" or "UI".

## Performance

Each shadow-casting light renders the casters' silhouettes into a stencil buffer — cost scales with `light_count * shadow_caster_count`. Cull aggressively:

| Technique | Effect |
|---|---|
| `enabled = false` on caster when off-screen | Skip caster |
| Light's `Shadow Intensity = 0` when not in dark zone | Skip the entire shadow pass for that light |
| Reduce light buffer resolution (Renderer2DData) | Half-res shadow accuracy, big perf win |

## Custom Shape

For sprites where the alpha silhouette doesn't match the desired shadow (e.g. a tree with leaves whose shadow should be a simpler ellipse), set `Use Renderer Silhouette = OFF` and edit the polygon in the inspector — fewer vertices = faster.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Self Shadows ON by default | OFF unless deliberately needed (causes z-fighting on flat sprites) |
| ShadowCaster2D on every body part of a skeletal character | CompositeShadowCaster2D on the parent + ShadowCaster2D on parts |
| `Use Renderer Silhouette = ON` for high-alpha-detail sprites (leaves) | Custom Shape with simplified polygon |
| All lights have Shadow Intensity > 0 | Only key lights cast; ambient/background lights have 0 |
