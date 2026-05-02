# 2D Lights Workflow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/lighting-art`.
> Canonical source: Unity URP 2D Lighting documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/lighting-art/SKILL.md

## Concept

Unity URP 2D Lights replaces baked-in shading with engine-computed
lighting. Sprites become "lit-aware": diffuse + normal map + emissive
mask, plus Light 2D components in scene that compute per-pixel shading
at runtime. Critical for dynamic time-of-day, point lights (torches,
explosions), and shadow casting.

## Walkthrough / mechanics

### Project setup

1. **Use URP**: Window > Package Manager > install Universal RP.
2. **Create URP Asset**: Create > Rendering > URP Asset (with 2D
   Renderer). Set as default in Project Settings > Graphics.
3. **2D Renderer Data**: open the asset. Add features:
   - **Light 2D Render Feature** (always on for 2D Lights)
   - **Normal Map Render Feature** (if using normal maps)
4. **Camera**: Camera GameObject > Renderer = URP 2D Renderer Data.

### Light 2D types

| Type | Use case |
|------|----------|
| **Global Light 2D** | Ambient sun-like global lighting |
| **Point Light 2D** | Torches, lamps, explosions, magic |
| **Sprite Light 2D** | Light-shaped like a sprite (use cookie texture for stained-glass projection) |
| **Freeform Light 2D** | Custom polygonal light shape (campfire glow, magic aura) |
| **Parametric Light 2D** | Procedural shape (circle, rectangle) |

Add: GameObject > Light > 2D > [type]. Attach to scene.

### Light parameters

```
Color           — RGB tint
Intensity       — 0..N (1 = normal, 2 = bright)
Falloff         — 0..1 (smoother edges)
Falloff Strength — distance over which falloff applies
Inner Radius    — for Point: full bright center
Outer Radius    — distance to falloff start
Blend Style     — for stacking with other lights
Shadow Intensity — how much occluders cast shadow
```

### Sprite lit-aware setup

For a sprite to react to 2D Lights:

1. **Sprite Renderer** > Material > "Sprite-Lit-Default" material
   (or custom URP-2D-Lit shader).
2. Sprite must have Normal Map secondary texture (Sprite Editor >
   Secondary Textures > Add > _NormalMap > drag normal map).
3. Optional: emissive secondary texture (_MaskTex with mask channel
   for emissive pixels).

Without normal map: lighting only affects ambient brightness uniformly
(flat sprites). With normal map: per-pixel directional lighting.

### Shadow Caster 2D

For sprites that cast shadows when blocking lights:

1. Add Shadow Caster 2D component to the GameObject.
2. Set "Use Sprite Renderer Mesh" or define custom outline.
3. Set "Casts Shadows" = true.

Shadow Caster outlines define what blocks light. Configure:
- Shadow Cast Group: optional grouping for performance.
- Self Shadows: whether the sprite shadows itself.

## Worked example

Authoring a torch-lit dungeon scene:

1. **Scene**: 2D scene with player, environment tiles, and torches.
2. **Global Light 2D** = dim cool gray (#1a1f2e), intensity 0.3.
   Provides ambient blue-cool fill.
3. **Per torch**: Point Light 2D at torch position:
   - Color: warm orange (#ff7a30)
   - Intensity: 1.5
   - Outer radius: 6 units
   - Inner radius: 0.5 units (full bright at flame)
   - Falloff: 0.8 (smooth edges)
4. **Player sprite**: lit-default material + normal map. Player picks
   up warm shading near torches, cool shading away.
5. **Wall tiles** with Shadow Caster 2D: cast shadow when between
   torch and player.
6. **Flicker animation**: script on torch sets Intensity to 1.5 +
   sin(Time * 7) * 0.2 each frame. Subtle flickering.

Result: dungeon feels lit by torches, ambient blue glow elsewhere,
walls cast shadows.

## Performance considerations

- **Light count**: each Light 2D = ~1 draw pass overhead. Keep <10
  active lights at once. Disable lights outside camera view.
- **Shadow Casters**: each adds geometry. Limit to important
  occluders (walls), not every prop.
- **Light blend styles**: each blend style = ~1 extra render pass.
  Use 1-2 blend styles unless effect requires more.
- **Mobile / Switch**: aggressive light count limits. Test on target.

## Common pitfalls

- **Sprites not reacting to lights**: material is "Sprite-Default"
  not "Sprite-Lit-Default". Switch material.
- **Pre-baked shadows on sprite + 2D Light**: double-shaded look.
  Either flat-paint diffuse OR don't use 2D Lights for shading.
- **Normal map secondary texture wrong slot**: Sprite Renderer doesn't
  find normal map. Verify Secondary Textures has `_NormalMap` (with
  underscore prefix).
- **G channel inverted**: lighting comes from "wrong" direction. Toggle
  Y in import or shader.
- **Shadow Caster encloses sprite incorrectly**: shadows look wrong.
  Adjust outline manually in Sprite Editor.
- **Performance crater on mobile**: too many lights. Audit and
  consolidate. Use BlendStyle = Multiplicative for low-cost
  ambient + Additive for highlights.
- **Lighting doesn't match palette swap**: 2D Light shading happens
  on the swap-output color, not the source. If swap target has
  different ramp, lighting may look off-style.

## References

- Unity URP 2D Lighting: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/2d-lighting.html
- 2D Renderer Data: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/2DRendererData_overview.html
- Shadow Caster 2D: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/ShadowCaster2D-overview.html
- Hyper Light Drifter dev blog (2D Lights examples in production)
