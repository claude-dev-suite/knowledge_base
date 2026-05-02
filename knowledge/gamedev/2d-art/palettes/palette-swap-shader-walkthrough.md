# Palette Swap Shader - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/palettes`.
> Canonical source: Game Maker's Toolkit "Color in pixel art" video; Lospec articles
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/palettes/SKILL.md

## Concept

Palette swap = render the same sprite with a different color palette
without redrawing. Used for character variants, factions, status
effects, day/night shifts. Two main implementations: indexed mode
(sprite stores palette indices, palette texture replaces colors at
render) or color-replacement shader (sprite stays RGB, shader matches
known colors and remaps).

## Walkthrough / mechanics

### Indexed-mode approach

Aseprite Indexed mode stores each pixel as a 1-byte palette index
(0-255). At render, shader samples the sprite (giving an index) and
uses the index to look up the actual color in a separate palette
texture (1×N, where N is your color count).

Unity URP shader (HLSL fragment):

```hlsl
sampler2D _MainTex;        // sprite stored with R channel = palette index
sampler2D _PaletteTex;     // 1xN palette LUT
float _PaletteSize;        // number of colors in palette

half4 frag(v2f i) : SV_Target {
    half index = tex2D(_MainTex, i.uv).r;
    half2 lutUV = half2(index * 255.0 / _PaletteSize, 0.5);
    return tex2D(_PaletteTex, lutUV);
}
```

Sprite must be exported with R channel = index. Aseprite "Indexed
mode" export to PNG preserves indices in the R channel of an indexed
PNG.

To swap: change `_PaletteTex` material property → instant variant.

### Color-replacement approach

Sprite is normal RGBA. Shader has uniform array of N source colors
and N target colors. Match-and-replace per pixel.

Unity URP shader:

```hlsl
#define MAX_COLORS 16
float4 _SrcColors[MAX_COLORS];
float4 _DstColors[MAX_COLORS];
int _ColorCount;

half4 frag(v2f i) : SV_Target {
    half4 c = tex2D(_MainTex, i.uv);
    for (int n = 0; n < _ColorCount; n++) {
        if (distance(c.rgb, _SrcColors[n].rgb) < 0.01) {
            return half4(_DstColors[n].rgb, c.a);
        }
    }
    return c;
}
```

Set `_SrcColors` once (the original palette) + `_DstColors` per
variant. Limit ~16 colors due to shader uniform array size.

## Worked example

Player character authored in DB16 palette (16 colors). Variants:
red-team / blue-team / poisoned.

```cpp
// Palette: DB16 - https://lospec.com/palette-list/dawnbringer-16
const Color[] DB16 = [
  #140C1C, #442434, #30346D, #4E4A4E, #854C30, #346524, #D04648, #757161,
  #597DCE, #D27D2C, #8595A1, #6DAA2C, #D2AA99, #6DC2CA, #DAD45E, #DEEED6
];

// Red team: swap colors 6, 9 to red shades
const Color[] RED_TEAM = [...same except 6→#FF4040, 9→#CC2020];

// Blue team: same plot, blue
const Color[] BLUE_TEAM = [...6→#4060FF, 9→#2030CC];

// Poisoned: every color shifted toward purple
const Color[] POISONED = [
  // each color rotated H by +180° toward purple
  // and saturation reduced 30%
  ...
];
```

Set `_DstColors` material property to the variant array. Done.

## Common pitfalls

- **Sprite has anti-aliasing colors not in palette**: AA pixels =
  custom blend colors → don't match any source. Authoring rule: use
  ONLY palette colors when authoring (Aseprite Indexed mode enforces
  this).
- **Floating-point precision in color compare**: `distance(c, src) <
  0.01` works for typical 8-bit colors. Tighter epsilon = misses;
  looser = false matches.
- **Indexed PNG support**: not all engines load indexed PNGs as
  index-textures. Unity may auto-convert to RGB. Use Texture Importer
  → Format → R8 to force.
- **Variant mid-game** (poison damage tick): blend two palette swaps
  over time. Lerp the `_DstColors` array between two variants.
- **Mixing palette swap with normal map**: swap only diffuse pass; lit
  shading happens normally on swapped colors.
- **Palette size > shader array size**: split sprite by region (limbs
  use one palette, body another), or use indexed-mode workflow.

## References

- Aseprite Indexed mode: https://www.aseprite.org/docs/color-mode/
- Lospec palette database: https://lospec.com/palette-list
- Unity URP custom shader: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest/manual/writing-custom-shaders-urp.html
- Game Maker's Toolkit "Secrets of Game Feel and Juice"
