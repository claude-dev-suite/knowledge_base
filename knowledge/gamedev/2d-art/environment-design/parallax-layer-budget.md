# Parallax Layer Budget - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/environment-design`.
> Canonical source: "Level Design 101: Leveraging Parallax Scrolling in 2D"; Hollow Knight environmental art studies
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/environment-design/SKILL.md

## Concept

Parallax = background layers scrolling at different speeds, creating
depth illusion. The art is in choosing how many layers (the budget) and
what each contributes. Too few = flat scene; too many = expensive
(each layer = additional draw call) and visually noisy. This article
gives a concrete framework for layer budget per game style.

## Walkthrough / mechanics

### Layer budget by game style

| Style | Layers | Notes |
|-------|--------|-------|
| Minimal indie / mobile | 3 | Far bg + mid bg + gameplay |
| Modern indie 2D action | 5 | Sky + far + mid + near + gameplay |
| Lush atmospheric | 7+ | Multiple foreground/background subdivisions |
| Hollow Knight-style | 8-10 | Sky + cave-far + cave-mid + cave-near + gameplay + foreground-rocks + foreground-fog + post-process |

Each additional layer = 1 extra draw call. On modern desktop: trivial.
On mobile / Switch: budget matters. Test FPS as you add layers.

### Scroll speed assignment

Assign each layer a scroll-speed multiplier (camera movement * factor):

```
Layer 1 (most distant - sky/clouds):  factor 0.05
Layer 2 (far mountains):                factor 0.15
Layer 3 (mid hills/forest):             factor 0.40
Layer 4 (near props):                   factor 0.75
Layer 5 (gameplay):                     factor 1.00
Layer 6 (foreground details):           factor 1.25
```

Rule: each layer should be ~2-3x faster than the layer before it (in
"farther" direction). Linear ratios feel mechanical; exponential
spacing feels natural.

### Color management per layer

Atmospheric perspective: distant objects desaturate + lose contrast +
shift toward sky color (warm bias for sunlit, cool bias for shadow).

Per-layer palette adjustments:

```
Saturation     Layer
20-30%         Sky / far bg
40-60%         Far mountains
60-80%         Mid hills
80-100%        Foreground / gameplay
80-100%        Decorative foreground (matches gameplay)
```

Apply via tint shader or pre-baked into the art. Most modern engines
have URP Volume Color Grading per camera priority — layer-specific
tints possible.

### Layer dimensions

Each parallax layer must be wider than camera viewport (so it fills
the screen as it scrolls).

For a 320x180 native viewport:
- Layer 1 (slowest): canvas 480x270 minimum (1.5x viewport). Scrolls
  slowly so doesn't need to be enormous.
- Layer 5 (gameplay): unbounded — covers the actual world.
- Layer 6 (fastest): canvas 640x360 (2x viewport). Scrolls fast,
  needs more total width.

Or use **tileable** parallax layers: paint a horizontally-seamless
strip and repeat as camera moves.

## Worked example

Designing parallax for a forest exploration game:

Camera target: 320x180 viewport (1080p × 6 integer scale).

### Layer 1: Sky
- Content: gradient from #6090d0 (top) to #b8d8e8 (horizon).
- Scroll factor: 0.0 (sky doesn't move with camera at all).
- Saturation: 30%.
- Tile: just one big canvas, position fixed.

### Layer 2: Distant mountains
- Content: silhouettes of mountains, monochromatic blue-gray.
- Scroll factor: 0.10.
- Saturation: 25%.
- Tile: 480x270 sprite, repeated horizontally.

### Layer 3: Far forest
- Content: forest silhouettes, dark green.
- Scroll factor: 0.30.
- Saturation: 40%.
- Tile: 640x360, repeated.

### Layer 4: Mid hills
- Content: detailed forest, multi-color.
- Scroll factor: 0.60.
- Saturation: 70%.
- Tile: 800x450, repeated.

### Layer 5: Gameplay
- Content: tilemap (tiles, platforms, enemies).
- Scroll factor: 1.0.
- Saturation: 100%.
- Not tiled; world coordinates.

### Layer 6: Foreground vines
- Content: leafy vines hanging from above.
- Scroll factor: 1.20 (faster than camera).
- Saturation: 100%.
- Tile: 320x180 strips.

Total: 6 layers. Renders 6 sprites + tilemap per frame. ~10-20% GPU
overhead vs. flat (1 layer) on desktop.

### Sample Unity Cinemachine setup

Each layer = its own GameObject with Sprite Renderer + Parallax
Component (custom or Cinemachine 2D Parallax extension).

```csharp
public class ParallaxLayer : MonoBehaviour {
    [SerializeField] float scrollFactor = 0.5f;
    Vector3 startPos;
    Camera mainCam;

    void Start() {
        startPos = transform.position;
        mainCam = Camera.main;
    }

    void Update() {
        Vector3 camDelta = mainCam.transform.position - Vector3.zero;
        transform.position = startPos + camDelta * scrollFactor;
    }
}
```

For Cinemachine: use Cinemachine Body Track Subject + custom
Parallax Extension.

## Common pitfalls

- **All layers same scroll speed**: no parallax, looks flat.
- **Linear scroll speed ratios** (0.2, 0.4, 0.6, 0.8, 1.0): feels
  mechanical. Use exponential (0.05, 0.15, 0.40, 0.75, 1.0).
- **Foreground SLOWER than camera**: looks weird, like player isn't
  moving. Foreground SHOULD scroll FASTER (factor > 1.0).
- **Same color/saturation per layer**: depth illusion broken. Apply
  desaturation gradient.
- **Tile width less than viewport width**: visible seam as layer
  scrolls. Make tile width >= viewport width.
- **Forgot to round positions to pixel**: layers shimmer at sub-pixel.
  Snap each layer to integer pixel after scroll.
- **Performance crater on mobile from many layers**: profile. Combine
  layers if possible.
- **Layer at fixed Y but scrolling X**: looks fine on horizontal
  movement, weird on vertical. Allow Y parallax too if needed.

## Vertical vs horizontal parallax

Most games are horizontal-scroll (left-right movement). For vertical
movement (climbing tower, falling), apply parallax to Y axis too.

Tip: Y-axis parallax often wants different ratios than X. Player
moving up looks at sky → sky needs slow scroll factor on Y too.

## References

- "Level Design 101: Leveraging Parallax Scrolling in 2D" article
- Hollow Knight environmental art breakdowns (Team Cherry interviews)
- Stardew Valley parallax study (visible in modding community
  documentation)
- Cinemachine 2D documentation: https://docs.unity3d.com/Packages/com.unity.cinemachine@latest
