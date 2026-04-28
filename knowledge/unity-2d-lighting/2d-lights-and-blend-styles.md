# 2D Lights & Blend Styles

> Source: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/2DLightBlendStyles.html

URP's 2D Renderer ships five Light 2D types. Blend Styles define how lights compose with the unlit base.

## Light 2D types

| Type | What |
|---|---|
| **Global Light 2D** | Ambient color + intensity for the entire scene |
| **Freeform** | Polygon-shaped light (room interior, lava pool) |
| **Point** | Circular falloff (torch, glow) — Inner/Outer radius for falloff |
| **Sprite** | Light shape derived from a sprite (artist-defined falloff) |
| **Parametric** | N-sided polygon (legacy; prefer Freeform) |

Common setup: **one Global Light per scene** (low intensity for darkness) + **per-object Lights** to highlight key elements.

## Blend Styles — Renderer2DData

`Renderer2DData` defines up to 4 blend styles. Each has:

| Setting | Effect |
|---|---|
| **Blend Mode** | `Multiply` (darkens unlit areas — standard) or `Additive` (brightens) |
| **Render Texture Scale** | 0.5 = half-res light buffer (perf win, slight blur) |
| **Use Depth/Stencil** | Required for Shadow Casters |
| **Mask Channel** | Maps Sprite Renderer's `Mask Interaction` to specific channels (R/G/B/A) |

Default blend style 0 ("Default") is enough for most projects. Add more only if you need separate light groups (e.g. "Underwater" with blue cast applied selectively).

## Light 2D properties

```
Color                tint of the light
Intensity            multiplier (0..10 typical)
Falloff              Inner Radius (full intensity) -> Outer Radius (zero)
Falloff Strength     curve sharpness
Use Normal Map       ON to interact with normal-mapped sprites
Shadow Intensity     0..1 (0 = no shadows)
Volume Opacity       fog/volumetric look (small values 0.05..0.2)
```

## Light layers

URP 2D supports **Light Blend Styles** as a form of layering. Configure on the renderer:

```
Renderer2DData
  Light Blend Styles:
    [0] Default       Multiply, Render Scale 1.0   - Default Light Operation
    [1] Highlight     Additive, Render Scale 0.5   - For UI / hero highlights
    [2] Underwater    Multiply, Render Scale 1.0   - With blue tint
```

Each Light 2D references one Blend Style. SpriteRenderers' `Mask Interaction` selects which mask channels apply.

## Volumetric look

Set Volume Opacity > 0 on a Light 2D — it renders a soft volumetric layer outside the lit zone. Combined with Sprite Mask sprites, you can fake light cones in 2D.

## Performance

| Setting | Impact |
|---|---|
| Many overlapping point lights | Light buffer fillrate dominates |
| `Use Normal Map` ON | Doubles per-pixel cost on lit sprites |
| Shadow Intensity > 0 | Adds a pass per shadow-casting light |
| Render Texture Scale = 0.5 | Halves light buffer cost; minor blur |

Profile with Frame Debugger — look for the 2D Light Pass; collapse if it dominates.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Adding Light 2D without 2D Renderer | Configure Renderer2DData asset on URP renderer list |
| Many overlapping point lights with Shadow Intensity 1 | Lower shadow res; combine via blend styles |
| Light layers / blend styles undefined → all default | Set up at start; don't retrofit |
| Normal map use on every sprite | Only on hero / lit-emphasis sprites — fillrate cost |
