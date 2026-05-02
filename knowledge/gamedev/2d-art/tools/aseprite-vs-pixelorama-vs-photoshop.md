# Aseprite vs Pixelorama vs Photoshop - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/tools`.
> Canonical source: Aseprite features list; Pixelorama Godot project README; Photoshop docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/tools/SKILL.md

## Concept

Three tool options for pixel art. Aseprite is the de-facto standard
($20). Pixelorama is the FOSS alternative (free, Godot-based).
Photoshop is the general-purpose paint tool that artists know but is
overkill and clunky for pixel art. This article compares concrete
features and recommends the right tool for your team.

## Walkthrough / mechanics

### Aseprite

**License**: $20 binary (one-time, no subscription) OR free if you
compile from source yourself (FOSS).

**Strengths**:
- Best-in-class animation support: timeline, onion skin, tags, per-
  frame duration, JSON sidecar export.
- Indexed mode for palette workflows.
- Tilemap mode (since v1.3) for tileset authoring.
- Slice tool for per-region metadata.
- Lua scripting (extensive API for batch + custom tools).
- Aseprite Importer for Unity (native PSD-like import).
- Symmetry painting (X/Y mirror).
- Active development; bug fixes regular.

**Weaknesses**:
- UI can feel old to Photoshop users (single-canvas, fewer panels).
- Limited layer effects (no advanced filters like Photoshop).
- No vector tools.

### Pixelorama

**License**: Free, GPL.

**Strengths**:
- Aseprite alternative that's truly free.
- Godot-based (you can extend it via GDScript).
- Active community + decent feature parity with older Aseprite.
- Cross-platform (Linux/Mac/Win).

**Weaknesses**:
- Less polished than Aseprite.
- Lua scripting absent (GDScript instead, harder for non-Godot devs).
- Tilemap mode missing (as of v1.x; check current).
- No Aseprite Importer for Unity equivalent (manual import workflow).
- Smaller community = fewer tutorials.

### Photoshop

**License**: Adobe Creative Cloud subscription ($23/mo).

**Strengths**:
- Industry-standard for digital painting (non-pixel-art).
- Strong layer effects (blend modes, adjustment layers, masks).
- Vector tools (shapes, paths).
- Widely known.

**Weaknesses**:
- Pixel art workflow clunky: requires careful "Pencil" tool, "Nearest
  Neighbor" interpolation, no native indexed mode for game palettes.
- No timeline animation that matches game pipeline (Adobe Animate
  exists but is a separate product).
- No tilemap mode.
- Subscription model = expensive over time.
- Tile preview / tiled mode plugins exist but are clunky.

## Comparison matrix

| Feature | Aseprite | Pixelorama | Photoshop |
|---------|----------|------------|-----------|
| Cost | $20 once | Free (FOSS) | $23/mo |
| Platform | Win/Mac/Linux | Win/Mac/Linux | Win/Mac |
| Pixel art focus | Yes | Yes | No (general) |
| Animation timeline | Excellent | Good | Limited |
| Onion skin | Yes | Yes | Limited |
| Indexed mode | Native | Native | Native (mode > Indexed) |
| Tilemap mode | Yes (v1.3+) | No | No |
| Symmetry painting | Yes | Yes | Yes |
| Tile preview | Yes (View > Tiled Mode) | Yes | Plugin needed |
| Lua scripting | Extensive API | GDScript (harder) | Action scripts |
| Engine import (Unity) | Native importer | None | PSD Importer |
| Sprite sheet export + JSON | Native | Limited | Plugin needed |
| Slice tool | Yes | Limited | Yes (slice tool) |
| Filters / effects | Limited | Limited | Extensive |
| Vector tools | No | No | Yes |
| Maturity | High | Medium | Very high |
| Community / tutorials | Large | Small | Largest |

## Recommendation by use case

### Solo indie pixel art game
**Aseprite** ($20). Best ROI: spend the money once, save hours weekly
in workflow. The Aseprite Importer for Unity alone justifies the price.

### Hobbyist / learning
**Pixelorama** (free). Lower friction to get started. If you commit
serious effort, upgrade to Aseprite later.

### Small team with mixed art skill
**Aseprite** + **Photoshop**. Aseprite for sprites + animation;
Photoshop for backgrounds, large-resolution painting, UI. They
coexist well: PSD Importer in Unity reads PS files, Aseprite
Importer reads .ase.

### Large studio with traditional artists
**Photoshop** for painters + **Aseprite** for pixel-art specialists +
**Spine** for character animation. Pipeline: Photoshop -> Spine for
big rigged characters; Aseprite for tilesets + small sprites.

### Pure FOSS commitment
**Pixelorama** + **GIMP**. Less polished pipeline but no licensing.

## Workflow examples

### Aseprite + Unity workflow

1. Author sprite in Aseprite with tags ("idle", "walk", "attack").
2. Save as `character.aseprite` (NOT exported as PNG).
3. Drop the .aseprite file into Unity project.
4. Aseprite Importer (Unity, com.unity.2d.aseprite) auto-creates
   AnimationClips per tag, generates Sprite Atlas.
5. Drag generated Animator Controller onto GameObject — animations
   ready.

Time: ~5 min from save to running animation.

### Pixelorama + Unity workflow

1. Author sprite in Pixelorama.
2. Export sprite sheet: File > Export Sprite Sheet > PNG.
3. Manually create Aseprite-style JSON OR use Pixelorama's
   limited export.
4. Drop sprite sheet into Unity.
5. Manually slice in Sprite Editor (Auto-Slice or Manual).
6. Manually create AnimationClips per animation.

Time: ~30 min per character (vs 5 min Aseprite).

### Photoshop + Unity workflow

1. Author character in Photoshop with layers (head, body, etc.).
2. Save as .psd.
3. Drop .psd into Unity.
4. PSD Importer generates Sprite (multi-layer or merged).
5. Use Sprite Editor for slicing if multi-frame.
6. Create AnimationClips manually.

Time: depends on layer complexity. ~15-60 min per character.

## Common pitfalls

- **Buying Aseprite without trying first**: download FOSS source,
  compile, try for free. Or watch tutorials.
- **Using Photoshop for pixel art with default settings**: Pencil
  tool (not Brush), Nearest Neighbor (not Bilinear), pixel-aligned
  rulers — required setup.
- **Pixelorama as full Aseprite replacement**: missing tilemap mode +
  Lua scripting limits use cases.
- **Mixing tools mid-project**: 3 sprites in Aseprite, 5 in
  Photoshop — inconsistent workflow, slower.
- **Forgetting Aseprite Importer in Unity**: manual sprite slicing
  for every character = days lost.
- **Free trial Photoshop with pixel art locked behind subscription**:
  not worth subscribing for pixel art alone.

## References

- Aseprite official: https://www.aseprite.org/
- Pixelorama: https://www.orama-interactive.com/pixelorama/
- Photoshop pixel art tutorial (subreddit): r/PixelArt FAQs
- Aseprite Importer Unity package: https://docs.unity3d.com/Packages/com.unity.2d.aseprite@latest/
