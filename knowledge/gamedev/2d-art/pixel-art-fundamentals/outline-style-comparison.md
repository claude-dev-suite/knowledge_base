# Outline Style Comparison

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/pixel-art-fundamentals`.
> Canonical source: Pedro Medeiros (Saint11) outline tutorials, Slynyrd pixel art blog.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/pixel-art-fundamentals/SKILL.md

## Concept

Outline style is a pixel-art identity choice that affects every sprite for
the rest of production. The four common modes (full, selout, dark-shade
inline, no-outline) each trade silhouette readability for visual softness,
and each has implementation gotchas around shader-based dynamic outlines and
sprite-vs-environment contrast. This article compares the four with concrete
hex codes, sample sprites in ASCII, and a decision matrix.

## The four styles compared

```
Full outline:        every silhouette pixel + interior shape lines are darkest color
Selective outline:   outline only where sprite would otherwise blend with bg
Dark-shade inline:   outline replaced by darkest variant of fill color
No outline:          interior shading carries silhouette; relies on bg contrast
```

## Visual comparison (16x16 hero head, schematic)

```
  Full outline           Selective outline      Dark-shade inline      No outline
  (4 colors used)        (4 colors used)        (3 colors used)        (3 colors used)

  . . . X X X X . .      . . . X X X X . .      . . . d d d d . .      . . . d D D d . .
  . . X X X x x X .      . . . D D D x X .      . . . D D D x d .      . . D D D x x d .
  . X X o x x x X .      . . . o x x x X .      . . . o x x x d .      . . o x x x x d .
  . X x x x x x X .      . . . x x x x X .      . . . x x x x d .      . . x x x x x d .
  . X x O x x x X .      . . . x O x x X .      . . . x O x x d .      . . x O x x x d .
  . X x x x x x X .      . . . x x x x X .      . . . x x x x d .      . . x x x x x d .
  . . X x x x X . .      . . . x x x X . .      . . . x x x d . .      . . . x x x d . .
  . . . X X X . . .      . . . . X X . . .      . . . . d d . . .      . . . . D d . . .

  Legend:
  X = outline color (#1a1318 - near-black)
  D = dark shade color (#5a3a3a - dark skin)
  d = darkest fill color (#7a4a4a - inline outline = dark fill)
  o = mid skin (#c08070)
  x = light skin (#e0a890)
  O = eye pixel (#1a1318)
```

Notice how dark-shade-inline uses `d` (which is also the darkest skin color)
as outline. The "outline" merges into the form. Saves one palette slot.

## When each wins

```
Full outline:
  + Maximum silhouette readability on busy backgrounds.
  + Classic 16-bit feel (Chrono Trigger, Mega Man, Final Fantasy VI).
  + Easiest to extract for damage-flash shaders (just lerp outline color).
  - Looks "stickered on" if outline stays pure black.
  - Eats palette slot per character family.

Selective outline (selout):
  + Modern indie standard (Celeste, Hyper Light Drifter, Owlboy).
  + Sprite blends into bg where appropriate (grass meets character grass-side).
  + Looks painterly without losing silhouette.
  - Hardest to author (decide every edge per frame).
  - Animation requires re-deciding selout per frame.

Dark-shade inline:
  + Single palette per character (no separate outline color).
  + Soft, painterly.
  + Used in Sword and Sworcery, Owlboy interior detail.
  - Silhouette weak against dark bg - sprite disappears.
  - Hard to read at small sizes.

No outline:
  + Most painterly, "concept art" feel (Hollow Knight's smaller sprites).
  + Frees palette slots.
  - REQUIRES strong bg-character contrast at the engine level.
  - Color flash / hit shader effects harder (no outline to flash).
```

## Outline color choice

Pure black outline (`#000000`) creates the "sticker" effect. Use a hue-tinted
near-black tied to the dominant scene color:

```
Scene mood        Outline hex     Notes
Forest day        #1a1f12         Slight green-warm
Forest night      #08111a         Slight blue-cool
Desert day        #2a1810         Warm brown-black
Cave              #0a0510         Slight purple
Snow              #181825         Slight blue
Underwater        #0a1a26         Slight cyan
```

Tinted outline integrates with palette ramps. Palette ramp first color often
serves as outline.

## Selout decision: where to outline

```
Apply outline pixel where:
  a) Adjacent bg color is within 30% luminance of the sprite edge color.
  b) Edge is along character silhouette (helmet top, foot bottom, weapon tip).
  c) Edge is on the side facing dominant light (rim lighting effect).

Skip outline pixel where:
  a) Edge is against high-contrast bg (sprite on bright sky, inside a cave).
  b) Edge is between two interior shades (ramp transition).
  c) Edge is "casting shadow" on itself (under chin, inside fold).
```

This is per-pixel discipline. For a 32x32 sprite, expect 30-60 outline-pixel
decisions.

## Shader-based dynamic outline

Drawing outline at runtime instead of in the sprite:

```glsl
// Fragment shader, sprites with transparent edges
// Sample 4 cardinal neighbors; if center alpha=0 and any neighbor alpha>0,
// draw outline color.

uniform sampler2D u_tex;
uniform vec2 u_texelSize;
uniform vec4 u_outlineColor;

void main() {
    vec4 c = texture2D(u_tex, v_uv);
    if (c.a > 0.0) {
        gl_FragColor = c;
        return;
    }
    float n = texture2D(u_tex, v_uv + vec2(0, u_texelSize.y)).a;
    float s = texture2D(u_tex, v_uv - vec2(0, u_texelSize.y)).a;
    float e = texture2D(u_tex, v_uv + vec2(u_texelSize.x, 0)).a;
    float w = texture2D(u_tex, v_uv - vec2(u_texelSize.x, 0)).a;
    float edge = step(0.01, n + s + e + w);
    gl_FragColor = vec4(u_outlineColor.rgb, edge * u_outlineColor.a);
}
```

Pros: outline can change color dynamically (highlight when interactable). Cons:
adds 1 pixel of "bleed" outside sprite bounds; conflicts with selout (you get
full outline back); requires extra sprite atlas padding to avoid sampling
neighbor sprites.

Use shader-outline for *highlights* (selected unit, interactable item), not
for the base sprite outline.

## Pitfalls

- **Mixing styles in one game**: hero is full-outline, enemies are no-outline.
  Visual incoherence. Pick one base style; deviate only for foreground props.
- **Outline color across scenes**: same character with one tinted outline
  per biome looks like 4 different characters. Use one outline tint per
  character regardless of scene.
- **Outline thickness > 1 px** at small sizes: 1px outline is sufficient up
  to 64x64. Above 96x96, 2px outline becomes legible.
- **Selout dropped on animation frames**: sprite "flickers" outline frame to
  frame. Verify selout consistency in onion-skin view.
- **Pure black outline + dark bg**: silhouette merges. Use tinted outline.
- **Selout + dynamic shader outline**: doubles up on edge pixels. If using
  shader outline, the source sprite must be no-outline.
- **Damage-flash shader replacing outline color**: looks like outline
  disappeared. Replace fill colors only; keep outline stable.

## References

- Saint11 outline tutorials (archive on https://saint11.org/blog/)
- Slynyrd "Pixelblog" outline guide: https://saint11.org/ and https://www.slynyrd.com/
- Hyper Light Drifter art breakdown (Heart Machine devlogs)
- Pixel Logic by Michael Azzi (book, 2017) - chapter on outlines
