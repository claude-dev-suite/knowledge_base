# Offset-and-Paint Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/seamless-textures`.
> Canonical source: Aseprite Filter Offset documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/seamless-textures/SKILL.md

## Concept

The offset-and-paint trick is the canonical workflow for making any
hand-painted texture seamless. The texture is offset by half its size
(wrapping), exposing what would have been the seam at the texture's
center. The artist paints over the seam by hand. After offsetting back,
the texture tiles cleanly.

## Walkthrough / mechanics

### Aseprite workflow (32x32 example)

1. **Open** your work-in-progress texture (e.g., 32x32 grass).
2. **Filter > Offset / Wrap** with X=16, Y=16 (half each dimension).
   - Setting: "Wrap pixels around" (NOT clamp).
   - The seam from previous edges now runs across the center,
     forming a visible cross-hair pattern.
3. **Paint over the seam** with brush, clone-stamp tool (Aseprite
   has "Pattern Tool" since v1.3), or hand pixel.
   - Match the surrounding texture style (don't introduce new motifs).
   - Use the existing palette only.
4. **(Optional) Offset back** by (-16, -16) to verify no new seam at
   the original boundaries.
5. **Test at scale**: place a 2x2 or 3x3 grid of copies in a temporary
   layer to inspect.

### Krita

`Filter > Other > Offset...` with Wrap option. Otherwise same.

### Photoshop

`Filter > Other > Offset` with Wrap option. Same workflow.

### Custom code (for tooling)

```python
def offset_wrap(image, dx, dy):
    """Wrap-offset an image."""
    h, w = image.shape[:2]
    return np.roll(image, shift=(dy, dx), axis=(0, 1))
```

## Worked example

Making a 16x16 grass tile seamless:

```
Step 1: Initial paint
+----------------+
| grass tuft     |
| grass blade    |
| ... (full art)|
+----------------+

Step 2: After offset (8,8)
+----------------+
| (lower-right   |
|  quadrant of   |
|  original)|seam|
| seam-cross----  |
| (upper-left    |
|  quadrant)     |
+----------------+

Step 3: Paint over seam
+----------------+
| (no visible    |
|  seam, blended |
|  artwork)      |
+----------------+

Step 4: Offset back (-8,-8)
Original layout, but now seamless when tiled.
```

## Tile testing in Aseprite

Aseprite v1.3+ has a "Tiled Mode" preview:
- View > Tiled Mode > Both Axes
- Canvas now shows the texture repeating in all directions
- Verify no visible seams as you paint

Useful for live-checking your offset-paint work.

## Common pitfalls

- **Mirrored half-symmetry**: if you offset (16,16) you may inadvertently
  paint a symmetric pattern (top-left mirrors bottom-right). Hand-paint
  to break mirror.
- **Detail at the new seam edge**: after offset, painting in the center
  is fine, but be aware that re-offsetting back may create NEW seam
  visible at original edges. Test always.
- **Color outside palette**: introducing a new color when painting over
  the seam → palette inconsistency. Stick to existing palette.
- **Forgetting to test at scale**: looks fine in editor at 100% zoom,
  visible seam when 4x4 copies stacked. Always test.
- **Procedural noise too obvious**: if base is Perlin noise, the offset
  reveals exact pattern. Paint over with hand-pixel detail to break
  procedural look.
- **Assuming 50% offset works for non-square textures**: for a 32x16
  texture, offset (16, 8) — half each dimension. NOT (16, 16).

## Tile + variant strategy

For repetition-reduction, after making a base tile seamless, paint
3-5 variants:

1. Variant A (canonical): the result of your seamless work.
2. Variant B: same color palette, slightly different detail
   placement (e.g., grass blade angled differently).
3. Variant C: rare variant with a small distinguishing feature
   (small flower, stone, etc.).

Each variant must ALSO be seamless against itself AND the others. Test
combinations.

## Animated seamless textures (water, fire)

For loop animation:

1. Author 4-8 frames of base texture.
2. Each frame must be seamless individually (apply offset-paint to each).
3. Each frame must transition to the next without seam (frame N+1's
   left edge == frame N's left edge in the loop continuity).

Trick for water: frame N is a horizontally-shifted version of frame N-1
plus subtle highlight changes. Creates "drift" feel.

## References

- Aseprite Filters documentation: https://www.aseprite.org/docs/filters/
- 2D textures workflow guide: https://blog.aseprite.org/2020/04/05/v1-3-beta/
- "How to make seamless textures" — community pixel art tutorials
