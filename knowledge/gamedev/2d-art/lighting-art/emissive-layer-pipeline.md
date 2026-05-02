# Emissive Layer Pipeline - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/lighting-art`.
> Canonical source: Unity URP custom shader docs; Aseprite layer system
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/lighting-art/SKILL.md

## Concept

Emissive pixels glow regardless of scene lighting. Use cases: street
lamps, lava, magic spells, hero eyes, neon signs, screens, fireflies.
Without emissive support, these elements look "dim" in dark scenes.
Pipeline: author emissive layer in Aseprite, export as separate
texture or alpha channel, sample in shader, add to lit color.

## Walkthrough / mechanics

### Aseprite layer authoring

1. Open sprite. Create layer named "emissive" (or any name).
2. Paint emissive pixels: which pixels should glow regardless of light?
3. The emissive layer's RGB = the glow color. Alpha = mask (where
   glow happens).
4. Other layers ("diffuse" or "main") contain the regular color.

### Export options

#### Option A: Separate textures
1. Export the diffuse layer alone as `sprite.png`.
2. Export the emissive layer alone as `sprite_emissive.png`.

In Aseprite: hide other layers, File > Export. Repeat per layer.

Or use Aseprite Lua script to batch-export per layer.

#### Option B: Alpha-channel mask
Emissive layer's alpha is the mask. RGB encodes color. Single texture
holds both diffuse (visible) and emissive (alpha layer).

Useful if you want a single asset slot. Less flexible if emissive needs
varied colors.

#### Option C: Aseprite Importer multi-layer (Unity)
Import the .aseprite as multi-layer. Aseprite Importer creates
separate Sprite assets per layer. Renderer uses both:
- Main Sprite = diffuse
- Secondary Texture = emissive

This is the cleanest workflow if using Aseprite + Unity.

## Shader integration

### Unity URP 2D shader (HLSL)

```hlsl
sampler2D _MainTex;       // diffuse
sampler2D _Emissive;      // emissive
float4 _EmissiveColor;    // tint, default white

half4 frag(v2f i) : SV_Target {
    half4 lit = sampleLighting(i);  // engine 2D Light contribution
    half4 emissive = tex2D(_Emissive, i.uv) * _EmissiveColor;
    
    // Add emissive AFTER lighting (so unlit pixels still glow)
    return lit * tex2D(_MainTex, i.uv) + emissive;
}
```

The key: emissive ADDED, not multiplied. Pure additive blending. So
even in pitch dark, emissive pixels show full color.

### Godot 4 canvas shader

```gdshader
shader_type canvas_item;

uniform sampler2D emissive_tex : source_color, hint_default_black;
uniform vec4 emissive_color : source_color = vec4(1.0);

void fragment() {
    vec4 base = texture(TEXTURE, UV) * COLOR;
    vec4 emiss = texture(emissive_tex, UV) * emissive_color;
    COLOR = base + emiss;
}
```

### Custom GLSL

Same pattern: sample base, sample emissive, add. The emissive sampling
should NOT be modulated by the scene light.

## Worked example

Authoring a lantern sprite:

1. Aseprite, 32x32 sprite of an iron lantern.
2. Layers:
   - "main" — iron body, dim flame, base look.
   - "emissive" — bright orange flame, halo around lantern, hot
     ember spots.
3. Export as `lantern.aseprite` (Aseprite Importer multi-layer).

In Unity:
- Sprite Renderer with main sprite.
- Secondary Texture = `_Emissive` (slot, set in Sprite Editor).
- Material = custom shader that adds _Emissive to lit base.

Result: lantern appears warm and glowing in scene, even when scene is
dark (Global Light 2D intensity 0.1). The flame "leaks" warm light
visually.

## Animation: flicker and pulse

For dynamic emissive (flickering torch, pulsing magic):

```csharp
// Unity C#
public class TorchFlicker : MonoBehaviour {
    public Material mat;
    public float baseIntensity = 1.0f;
    public float flickerAmount = 0.2f;
    public float flickerSpeed = 7.0f;

    void Update() {
        float flicker = baseIntensity + Mathf.Sin(Time.time * flickerSpeed) * flickerAmount;
        mat.SetColor("_EmissiveColor", new Color(1, 0.5f, 0.1f) * flicker);
    }
}
```

Set the shader uniform `_EmissiveColor` per frame. Scene-wide flicker
without per-frame texture changes.

For varied flicker: use Perlin noise instead of sine for more organic
look.

## Bloom integration

Bloom post-process amplifies bright pixels. With emissive layer,
emissive pixels become bloomed → soft glow halo.

Setup (Unity URP):
1. Volume in scene > Add Override > Post-processing > Bloom.
2. Threshold: 1.0 (only pixels brighter than 1.0 contribute).
3. Set _EmissiveColor intensity > 1.0 to push above threshold.

Now emissive pixels have a real bloom halo, not just additive color.

For pixel art: bloom blurs pixels by nature → conflicts with the
hard-pixel aesthetic. Use sparingly. Some games disable bloom on
non-emissive layer (custom render feature).

## Common pitfalls

- **Emissive multiplied with lit**: dark scene → emissive also dark.
  Wrong. Use ADD, not multiply.
- **Emissive texture compressed**: artifacts visible. Use Compression
  = None.
- **Emissive alpha treated as opacity**: shader masks glow pixels but
  applies to base too. Sample emissive separately.
- **Bloom blurs entire scene**: pixel art looks soft. Limit bloom to
  emissive-only via custom render feature.
- **Bloom threshold too low**: ALL bright pixels bloom (sky bloom,
  white bloom). Set threshold > 1.0.
- **Emissive painted in same layer as diffuse**: cannot extract.
  Always separate layer.
- **Aseprite Importer not picking up layer**: layer naming convention.
  Aseprite Importer expects specific names (check current docs).

## References

- Unity Aseprite Importer multi-layer: https://docs.unity3d.com/Packages/com.unity.2d.aseprite@latest
- Unity URP custom shader: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/writing-custom-shaders-urp.html
- URP Bloom: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/Post-Processing-Bloom.html
- Aseprite Lua scripting (batch layer export): https://github.com/aseprite/api
