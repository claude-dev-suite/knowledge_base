# Pixel Perfect Camera Config

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/pixel-art-fundamentals`.
> Canonical source: Unity Pixel Perfect Camera docs (com.unity.2d.pixel-perfect), Godot stretch mode docs.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/pixel-art-fundamentals/SKILL.md

## Concept

Pixel art breaks the moment a sprite renders at a fractional pixel position
or a non-integer scale. Engine "pixel perfect cameras" exist to absorb the
math: they pick an integer scale based on display size, snap world positions
to pixel grid, and handle camera smooth-follow without subpixel shimmer.
Their config has 4-6 dials and getting them wrong produces classic bugs:
rolling-shutter texture seams, jittering 1px, or blurry letterboxes.

## Unity 2D Pixel Perfect Camera

Component: `Pixel Perfect Camera` from package `com.unity.2d.pixel-perfect`.

```
Asset PPU:                 16   (must match Sprite Import Pixels Per Unit)
Reference Resolution X:    320
Reference Resolution Y:    180
Crop Frame:                None / Pillarbox / Letterbox / Window / Stretch Fill
Grid Snapping:             Upscale Render Texture / Pixel Snapping / None
Filter Mode (sprite):      Point (no filter)
Compression (sprite):      None
```

### Mode breakdown

```
Crop Frame = Pillarbox/Letterbox:
  Renders at largest integer scale, black bars on extra space.
  Cleanest pixels. Use for handheld-feel or fixed aspect.

Crop Frame = Window:
  Camera shows extra world tiles when display is wider than reference.
  Pixels stay clean; map design must accommodate visible margin.

Crop Frame = Stretch Fill:
  Final image stretched to fit display. NON-INTEGER blur. Avoid.

Grid Snapping = Pixel Snapping:
  Sprites snap to camera-pixel grid. Camera can move at subpixel.
  Result: smooth camera + crisp sprites. Recommended default.

Grid Snapping = Upscale Render Texture:
  Render at native res to texture, blit to screen at integer scale.
  Camera and sprites both subpixel-clamp to the texture grid.
  Smoothest motion at cost of camera "snapping" feel.
```

For Celeste-style smooth camera with crisp pixels: `Pillarbox/Letterbox` +
`Pixel Snapping` + sprite filter Point.

## Cinemachine integration

Cinemachine 2D virtual cameras output a "desired" position with float
precision. Pixel Perfect Camera then snaps. Pitfall: if you have damping on
the virtual camera AND pixel snapping, the camera appears to step. Fix by
keeping damping high enough that step boundaries occur during deceleration:

```
Recommended VCam damping:
  X damping = 0.3 - 0.5
  Y damping = 0.3 - 0.5
  Lookahead time = 0
  Dead Zone = 0.1 - 0.2 (in screen-space units, not pixels)
```

If you see "rolling shutter" tile seams at the edges of the viewport during
camera motion, the issue is usually:

```
1. Tile atlas missing extrude padding -> add 2px extrude in TexturePacker
   or Unity's Sprite Atlas Padding setting (set to 4).
2. Sprite Filter Mode = Bilinear -> set to Point.
3. Tilemap Renderer Mode = Chunk vs Individual; with subpixel camera,
   Individual mode often renders cleaner edges (slight perf cost).
```

## Godot 4 stretch config

Project Settings -> Display -> Window -> Stretch:

```
Mode = canvas_items:
  Whole viewport scales together; canvas items render at native res.
  GUI scales with viewport.

Mode = viewport:
  Render to a viewport at base resolution, scale that texture to display.
  Equivalent to Unity's Upscale Render Texture mode.

Aspect = keep:
  Maintain the project aspect; pillarbox/letterbox extra space.

Aspect = expand:
  Show more world if display is wider than project.

Aspect = ignore:
  Stretch to fit. Blurry. Avoid.
```

For pixel-perfect Godot: `Mode = viewport`, `Aspect = keep`,
`Project Settings -> Rendering -> Textures -> Default Texture Filter = Nearest`.

## Custom renderer (raylib, SDL, BevyECS)

Without an engine helper, do this manually:

```c
// Pseudocode for any low-level renderer
const int NATIVE_W = 320, NATIVE_H = 180;
RenderTexture target = LoadRenderTexture(NATIVE_W, NATIVE_H);
SetTextureFilter(target.texture, FILTER_POINT);

// Each frame:
BeginTextureMode(target);
  ClearBackground(BLACK);
  // Camera position must be integer in world space (or pixel-snapped)
  int cam_x = (int)floorf(camera.x);
  int cam_y = (int)floorf(camera.y);
  DrawWorld(cam_x, cam_y);
EndTextureMode();

// Compute integer scale that fits display
int scale_x = display_w / NATIVE_W;
int scale_y = display_h / NATIVE_H;
int scale = min(scale_x, scale_y);  // integer scale

// Letterbox the scaled native target onto display
int dst_w = NATIVE_W * scale;
int dst_h = NATIVE_H * scale;
int off_x = (display_w - dst_w) / 2;
int off_y = (display_h - dst_h) / 2;
DrawTexturePro(target.texture,
               (Rect){0, 0, NATIVE_W, -NATIVE_H},  // flip Y for raylib
               (Rect){off_x, off_y, dst_w, dst_h},
               (Vec2){0, 0}, 0, WHITE);
```

The two key invariants: camera position is integer-pixel, final blit is
integer-scale.

## Subpixel sprite motion (the smooth-camera trick)

Pure integer camera "snaps" - feels jerky. Subpixel camera with snapping
sprites feels modern. Algorithm:

```
1. Camera world position is float, accumulated from input/physics.
2. World offset for rendering = camera_position (float).
3. Sprite position on screen = round(world_pos.x - camera.x) (int pixels).
4. Camera offset for sub-pixel "shimmer" = camera_pos - floor(camera_pos).
5. Apply offset by either:
   a) Render world to texture, blit with subpixel UV offset (Upscale Render
      Texture in Unity).
   b) Sample tilemap texture with shifted UVs (custom shader).
```

This produces 4x effective camera precision without blurring sprites.

## Pitfalls

- **Sprite import PPU mismatch with Asset PPU**: 16px tiles imported at
  PPU=32 render at half size; world feels small. Fix: every sprite at the
  same PPU.
- **UI inside the pixel-perfect camera**: HUD pixels snap with camera,
  jittering as camera moves. Solution: separate UI camera with
  `Pixel Perfect = off`, render UI at higher res.
- **Non-integer Reference Resolution / Display ratio**: 320x180 on a 1366x768
  monitor scales 4x with 86px letterbox. Acceptable. Avoid 320x200 (no clean
  scale to common displays).
- **Tilemap chunks tearing**: Unity Tilemap Renderer in Chunk mode caches
  geometry; subpixel camera reveals chunk boundaries. Switch to Individual
  mode or use Tilemap Renderer "Animate" plugin.
- **Filter Mode left as Bilinear**: every sprite is blurry. Always Point.
- **Compression on pixel-art sprites**: introduces dithering artifacts. Set
  Compression = None on pixel sprites.
- **Bloom + Pixel Perfect**: bloom samples post-resolution; if camera is
  Upscale Render Texture, bloom blurs the upscaled pixels. Apply bloom on
  the native render target before scaling.

## References

- Unity Pixel Perfect Camera docs: https://docs.unity3d.com/Packages/com.unity.2d.pixel-perfect@latest
- Godot stretch mode: https://docs.godotengine.org/en/stable/tutorials/rendering/multiple_resolutions.html
- "Slime World" article on Pico-8 style pixel-perfect rendering by Pekka Vaananen
