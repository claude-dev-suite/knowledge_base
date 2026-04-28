# Pixel Perfect Camera

> Source: https://docs.unity3d.com/Packages/com.unity.2d.pixel-perfect@latest

For pixel-art games, sub-pixel jitter ruins visual fidelity. The Pixel Perfect Camera component snaps every render to integer pixel multiples.

## Setup

Add `PixelPerfectCamera` to your Main Camera (must be Orthographic).

```
Reference Resolution    320 x 180          base canvas — pick once
Assets PPU              32                 must match all sprites
Crop Frame              Letterbox          preserves aspect, adds black bars
                        or Stretch Fill   stretches viewport
                        or Pillarbox       horizontal letterbox
Pixel Snapping          ON                 snaps every renderer to integer pixels
Upscale Render Texture  ON                 renders to RT then scales up cleanly
Run In Edit Mode        ON for previewing
```

## Common base resolutions

| Reference Res | Style |
|---|---|
| 256 x 144 | Tight retro (NES-era) |
| 320 x 180 | Modern indie (Celeste-like) |
| 384 x 216 | Medium pixel-art |
| 480 x 270 | HD pixel-art |
| 640 x 360 | "Big pixel" or stylized vector |

Use 16:9 aspect ratios — they divide cleanly into common display resolutions.

## Cinemachine integration

If using Cinemachine 3:

```
Main Camera (Orthographic + CinemachineBrain + PixelPerfectCamera)
  CinemachinePixelPerfect extension on each CmCamera
```

Without the extension, Cinemachine smooth-damps the camera in real coordinates — sub-pixel jitter returns.

## Verifying

Slowly walk the player. Background should NOT shimmer at 1px scale. Test on the actual target resolution, not just editor.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Bilinear filter on sprites + Pixel Perfect | Sprites must be Filter Mode = Point |
| Mismatched PPU between sprites and `Assets PPU` | Project-wide PPU consistency |
| Cinemachine without PixelPerfect extension | Add the extension |
| Reference Resolution that doesn't divide target res cleanly | Pick an aspect-ratio-consistent res |
