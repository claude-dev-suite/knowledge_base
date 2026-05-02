# Resolution Budget Decision

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/pixel-art-fundamentals`.
> Canonical source: Pedro Medeiros (Saint11) art tutorials, Celeste/Towerfall postmortems.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/pixel-art-fundamentals/SKILL.md

## Concept

Resolution choice is a labor budget. A 320x180 game costs roughly 1/4 the art
hours of a 640x360 game, frame-for-frame, but readability scales nonlinearly:
characters at 16x16 are emoji-readable; at 32x32 they hold facial expressions;
at 64x64 you need full anatomy. This article gives concrete labor-per-asset
estimates and a framework for picking native resolution based on team
size and art ambition.

## Labor scaling

For one trained pixel artist, time per asset roughly scales with pixel count
of the moving subject (not background). Rough numbers from indie postmortems:

```
Subject size (px)   Time per static sprite   Time per 8-frame anim
8x8 - 16x16         15-30 min                1-3 hours
24x24 - 32x32       1-3 hours                4-12 hours
48x48 - 64x64       4-8 hours                16-40 hours
96x96 - 128x128     1-3 days                 1-2 weeks
160x160+            painterly territory      separate budget
```

A character with 8 animations at 32x32 is roughly a week. At 64x64 it is a
month. At 128x128 it is the rest of the quarter.

## Native resolutions and their feel

```
Native       Aspect    Display @ 1080p   Vibe              Real-world reference
160x144      10:9      7.5x integer*     GameBoy DMG       Link's Awakening DX
240x160      3:2       6.75x*            GBA               Mother 3, Castlevania Aria
256x224      8:7       4.82x*            SNES NTSC         Chrono Trigger
320x180      16:9      6x clean          Modern lo-fi      Celeste (renders 320x180 -> 1920x1080)
320x240      4:3       4.5x              4:3 lo-fi         Stardew Valley
384x216      16:9      5x clean          Mid res           Hyper Light Drifter base
480x270      16:9      4x clean          Indie sweet spot  Many roguelikes
640x360      16:9      3x clean          Detailed pixel    Owlboy
960x540      16:9      2x clean          High-res pixel    Songs of Conquest
```

*Non-integer scales blur. For non-16:9 native, letterbox or pillarbox to
keep integer scaling. 320x180 is the most common modern choice because it
divides 1920x1080 by exactly 6 and 1280x720 by exactly 4.

## Decision framework

Rank these by importance to your project, then pick the resolution that wins
the most:

```
Want / constraint                     Lean toward
Solo dev, 1 year scope                160-320 wide
2-3 artists, 2 year scope             320-480 wide
Faces and dialogue portraits          480 wide minimum, or render portraits separately
Mobile (battery, RAM)                 320-480 wide
Switch handheld feel                  480 wide
Steam Deck primary target             720 wide possible (HD pixel)
Strong silhouette > detail            smaller wins
Painterly + detailed > silhouette     larger wins
Plan to license to Lospec / fans      respect 16/32/64 multiples
```

## Worked example: solo metroidvania

Constraints: 1 artist, 18 months, 60 unique enemy designs, 5 bosses, 80
screens of unique art. Math:

```
Native option A: 320x180
  - Player:   16w x 24h, 8 animations of avg 8 frames = 64 frames
              ~6h per frame in production pipeline (sketch/clean/anim) = 384h
  - Enemies:  60 x avg 24x32, 5 anims x 6 frames = 30 frames each
              ~3h per frame = 5400h - already overbudget
  Reality: cut to 30 enemies, share rigs. Plausible.

Native option B: 480x270
  - Player:   24w x 36h, 8 animations x 8 frames
              ~14h per frame = 896h
  - Enemies:  30 x avg 36x48, 5 anims x 6 frames
              ~10h per frame = 9000h - DOA for solo
  Reality: 10 enemies, shared rigs, 12-month delay, cut bosses.
```

For solo metroidvania: 320x180. Hero size 16x24. Enemies 16x24 to 32x32.
Bosses up to 64x96 because they reuse silhouette without animation count.

## Display scaling rules

A native pixel must always cover an integer number of physical pixels.
Non-integer scaling (e.g., 320x180 -> 1366x768 = ~4.27x) blurs unless you do
one of:

```
Approach A: pillar/letterbox
  Render 320x180 -> 1280x720 at 4x. Letterbox 43px top/bottom on 768 display.
  Clean pixels, some screen unused.

Approach B: integer-scale render target + bilinear final
  Render to 1280x720 at 4x (pixel perfect), then bilinear upscale to 1366x768.
  Slight softness, full screen.

Approach C: per-pixel snap with slight subpixel
  Render at 1366x768 directly with pixel-perfect camera that snaps subpixel
  motion to integers. Smoother camera, occasional 1px artifacts.

Approach D: hybrid (Celeste, Hyper Light Drifter)
  Render world at native to a render target, then sample that target with
  custom shader that handles edge cases.
```

Unity's Pixel Perfect Camera supports A, C, and D modes. Godot supports
canvas_items stretch which maps to A. Most modern indies pick A (clean
pixels, accept letterbox on weird aspects).

## Pitfalls

- **Mid-project resolution change**: catastrophic. Every existing asset must
  be redrawn or upscaled at integer factor (which looks worse than native).
  Lock in week 1.
- **Mixed PPU between scenes**: title screen at 1280x720 native, gameplay at
  320x180 - players notice the resolution shift. Keep one native.
- **UI at native resolution**: 320x180 UI has no room for body text; players
  squint. Render UI at 2x or screen-space and overlay.
- **Non-power-of-2 sprites in atlases**: causes texture bleeding on some
  GPUs. Pad to POT or use atlas extrude.
- **Photo-real backgrounds + pixel art characters**: aesthetic mismatch
  unless deliberately (Disco Elysium does it with painted bg + pixel
  hud).
- **Choosing 256x144 because "smaller = retro"**: only 16:9 by 64-divide,
  rare displays don't divide cleanly. Stick to standard 320x180 or 480x270.

## References

- Saint11 / Pedro Medeiros art tutorials (Twitter/X archive)
- Celeste GDC talk (Maddy Thorson + Noel Berry, 2019) - resolution rationale
- Owlboy 9-year postmortem - resolution and scope creep lessons
- Pixel Perfect Camera (Unity) docs: https://docs.unity3d.com/Packages/com.unity.2d.pixel-perfect@latest
