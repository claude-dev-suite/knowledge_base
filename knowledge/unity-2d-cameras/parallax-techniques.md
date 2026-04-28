# Parallax Techniques

> Source: https://learn.unity.com/tutorial/2d-game-kit-walkthrough

Two practical approaches: **multi-camera** (heavier, pixel-perfect-friendly) and **sprite-based** (lighter, recommended for most games).

## Sprite-based parallax (recommended)

Each background layer moves a fraction of the camera's movement.

```csharp
[ExecuteAlways]
public class ParallaxLayer : MonoBehaviour {
    [SerializeField] private Transform cam;
    [SerializeField, Range(0f, 1f)] private float horizontalMul = 0.5f;
    [SerializeField, Range(0f, 1f)] private float verticalMul   = 0.5f;
    private Vector3 _start;
    private Vector3 _camStart;

    private void Start() {
        if (cam == null) cam = Camera.main.transform;
        _start = transform.position;
        _camStart = cam.position;
    }

    private void LateUpdate() {
        var delta = cam.position - _camStart;
        transform.position = _start + new Vector3(
            delta.x * horizontalMul,
            delta.y * verticalMul,
            0
        );
    }
}
```

Multiplier guidelines:

| Layer | Multiplier |
|---|---|
| Sky / far clouds | 0.05..0.15 |
| Distant mountains | 0.2..0.35 |
| Mid background | 0.4..0.55 |
| Foreground silhouettes | 0.7..0.85 |
| Player plane (no parallax) | 1.0 (or no script — it follows the world) |
| In-front-of-player accent | 1.1..1.3 (faster than world for super-foreground sense) |

## Infinite scrolling (looping backgrounds)

For sky/clouds that should loop infinitely as the camera moves:

```csharp
private void LateUpdate() {
    var delta = cam.position - _camStart;
    transform.position = _start + delta * horizontalMul;

    float spriteWidth = _renderer.bounds.size.x;
    float camOffset = (cam.position.x - transform.position.x);
    if (Mathf.Abs(camOffset) >= spriteWidth) {
        transform.position += new Vector3(Mathf.Sign(camOffset) * spriteWidth, 0, 0);
    }
}
```

For seamless tiling, prepare sprites with edges that match (or use `tilingMode` on a Sprite Renderer with `Draw Mode = Tiled`).

## Multi-camera parallax (pixel-perfect-friendly)

Each layer rendered by its own camera with its own orthographic size. Heavier but yields perfect alignment for pixel art.

```
Camera_Far    Orthographic Size 10  Culling Mask: ParallaxFar
Camera_Mid    Orthographic Size 7   Culling Mask: ParallaxMid
Camera_Game   Orthographic Size 5   Culling Mask: Game (default)
```

Stack with `Stacked Camera` in URP. All three move with the player (or a parent rig); the size difference creates the parallax effect mathematically.

## Vertical parallax for platformers with verticality

Don't only multiply X — multiply Y too (with a smaller value, often 0.5x of horizontal). Pure horizontal parallax looks flat when the player jumps.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Parallax math in `Update` | `LateUpdate` — runs after camera follow |
| Multiplier > 1 on all layers | Should be < 1 for "behind" feel; use > 1 only for super-foreground accents |
| Sprites with mismatched edges (visible seams) | Author seamless tileable backgrounds, or use Draw Mode = Tiled |
| Camera follow + parallax recalculating same delta | Centralize: parent layers under a rig that follows camera, only Z stays |
| No vertical parallax in tall platformer | Add `verticalMul` (smaller than horizontal) |
