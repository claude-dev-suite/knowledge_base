# Restricted Palette Design Process - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/palettes`.
> Canonical source: Lospec.com palette-design articles; "Color in pixel art" by Game Maker's Toolkit
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/palettes/SKILL.md

## Concept

Designing a 16-32 color palette for a project is one of the highest-leverage
art decisions. A coherent palette ensures every sprite + tile + UI element
sits visually together. The process: pick an existing curated palette as
a starting point, identify what's missing for your specific game, modify
strategically. Rarely worth designing from scratch.

## Walkthrough / mechanics

### Step 1: Define mood per scene
Before picking colors, list each scene's intended mood:
- Forest level → cozy, green-dominant, warm light
- Cave level → cool, dark, blue-purple shadow, warm torch accents
- Boss arena → dramatic, high contrast, restricted to 6-8 colors

### Step 2: Pick a base palette from Lospec
Start with one of:
- DB16 / DB32 (Dan Hascome) — versatile general-purpose
- Endesga 32 — vibrant, modern
- Resurrect 64 — well-ramped, painterly
- AAP-64 — soft, atmospheric
- PICO-8 — modern lo-fi indie

Don't try to design 32 colors from scratch. Curated palettes have
decades of community testing. Inherit and tune.

### Step 3: Identify gaps
For each material in your game (skin, foliage, stone, water, fire,
metal, cloth, fabric), check if the base palette has a usable 5-color
ramp. Gaps:
- Forest game → if base palette has only 2 greens, you need more.
- Underwater → cyan-blue ramp must support 5+ shades for depth.
- Skin tones → for character variety, need 2-3 skin ramps.

### Step 4: Modify gradually
Replace 4-8 colors at most. Beyond that, palette loses coherence.

For each replacement: pick the original color, decide what new color
serves the missing ramp. Check: does it ramp visually with adjacent
palette entries?

### Step 5: Test at scale
Make a quick scene (character + tilemap + UI) using the modified
palette. Look for:
- Colors that never get used → drop them, replace with something
  more useful.
- Colors that look out of place — too saturated or wrong hue.
- Missing mid-tones (causing banding).
- UI text readable against tilemap backgrounds.

Iterate until each color has a clear purpose.

## Worked example

Building a palette for a forest exploration game with combat.

Start: DB16 (16 colors).

Inventory ramps:
- Skin: ✓ 4 brown shades work as warm character ramp
- Foliage: ✗ only 2 greens, need 4-5
- Stone: ✓ 4 grays work
- Water: ✗ only 1 blue
- Fire: ✓ 3 reds + orange + yellow work
- Metal: ✓ grays double for steel

Modifications:
- Replace #757161 (drab khaki) → #4A7A2E (mid forest green)
- Replace #DAD45E (yellow-green) → #88B654 (bright fresh green)
- Replace #4E4A4E (dark gray) → #1A4A6E (deep blue water)

Final 16 colors:
```
#140C1C #442434 #30346D #1A4A6E
#854C30 #346524 #D04648 #4A7A2E
#597DCE #D27D2C #8595A1 #6DAA2C
#D2AA99 #6DC2CA #88B654 #DEEED6
```

Now I have:
- Foliage ramp: #1A4A6E (under-shadow) → #346524 → #4A7A2E → #6DAA2C → #88B654
- Water ramp: #30346D → #1A4A6E → #597DCE → #6DC2CA
- Skin ramp: #442434 → #854C30 → #D27D2C → #D2AA99
- Stone ramp: #140C1C → #4E4A4E → #8595A1 → #DEEED6
- Fire ramp: #442434 → #D04648 → #D27D2C → #DEEED6

Each material has a usable 4-5 step ramp. Total 16 colors. Coherent.

## Common pitfalls

- **Too many color replacements**: changes the palette's identity.
  Stop at 8-10 modifications max.
- **Replacing only by hue, not by ramp position**: new color doesn't
  fit into any ramp. Pick replacement that fills a SPECIFIC ramp gap.
- **No mid-tone in a ramp**: ramp jumps from dark→light with no
  middle → banding visible. Always include mid-shade.
- **Saturation crushed in shadows**: pure gray shadow reads as dead.
  Keep some chroma in the darkest entry of each ramp.
- **No "neutral" color**: useful for UI text/borders. Reserve 1 light
  neutral + 1 dark neutral.
- **Colors that look identical at game scale**: zoom in to verify
  picker shows adjacent shades, but they may render as the same color
  on screen. Test in-game.
- **Designing in vacuum**: palette must serve specific scenes.
  Mock up a scene EARLY in the process, don't wait for final palette.

## References

- Lospec.com palette catalog: https://lospec.com/palette-list
- DawnBringer's design notes (DB16, DB32): https://pixeljoint.com/forum/forum_posts.asp?TID=12795
- "How to design a color palette" by Lazyhound on YouTube
- Game Maker's Toolkit on color
