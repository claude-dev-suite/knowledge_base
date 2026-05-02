# Unity Aseprite Importer Pipeline - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/tools`.
> Canonical source: Unity Aseprite Importer documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/tools/SKILL.md

## Concept

The Aseprite Importer (com.unity.2d.aseprite, native Unity package
since 2023) reads .aseprite/.ase files directly into the Unity project.
Auto-generates Sprite atlases, splits frames, creates AnimationClips
based on Aseprite tags, and supports Indexed mode for palette swaps.
Replaces the old "export PNG sprite sheet, manually slice in Sprite
Editor, manually create AnimationClips" workflow.

## Walkthrough / mechanics

### Installation

Window > Package Manager > add by name:
```
com.unity.2d.aseprite
```

Or via Project Settings > Package Manager > Scoped Registries > Unity
Registry, then install "2D Aseprite Importer".

Now Unity recognizes .aseprite and .ase files in the project.

### Import settings

Drop `character.aseprite` into Assets/. Unity imports it. Click the
asset to inspect:

```
Texture Type: Sprite (auto-set)
Sprite Mode: Multiple
Pixels Per Unit: 16/32/etc. (set per project)
Filter Mode: Point (no filter) — IMPORTANT for pixel art
Compression: None
Mip Maps: Off (2D doesn't need mipmaps)

Aseprite Importer:
  Enable Tags: Yes (default)
  Generate Animation Clips: Yes
  Generate Animator Controller: Yes
  Layer Import: Merge Frame (default) or Group Layer
  Indexed Mode: respect / discard
```

### Tag-based animation generation

In Aseprite, tag your frames:
- F1-F8: tag "idle"
- F9-F16: tag "walk"
- F17-F22: tag "attack"

Save the .aseprite file. Unity's Aseprite Importer reads tags and
generates one AnimationClip per tag, all sharing the same Sprite atlas.

### Animator Controller generation

If "Generate Animator Controller" is enabled, the importer creates a
.controller asset with one state per AnimationClip. State transitions
default to AnyState; you adjust manually for your gameplay.

### Indexed mode integration

For palette-swap workflows: author the .aseprite file in Indexed mode.
Aseprite Importer preserves the index in the R channel. You write a
custom shader that reads the R channel as palette index and looks up
in a palette LUT texture (see `palette-swap-shader-walkthrough.md`).

## Worked example

Authoring a player character:

1. **Aseprite**: create new file 32x32, RGB mode (or Indexed for
   palette swap).
2. Author idle (4 frames), walk (8 frames), attack (5 frames),
   jump (3 frames) — total 20 frames.
3. Tags:
   - "idle": F1-F4
   - "walk": F5-F12
   - "attack": F13-F17
   - "jump": F18-F20
4. Save as `Assets/Sprites/Player.aseprite`.
5. **Unity**: Wait for import to complete.
6. In Project, expand `Player.aseprite`:
   - Sprite atlas (multiple Sprite assets per frame).
   - 4 AnimationClips: Player_idle, Player_walk, Player_attack,
     Player_jump.
   - 1 Animator Controller: Player.
7. Drag `Player` (top-level asset) into a scene → GameObject created
   with SpriteRenderer + Animator.
8. Animator already has clips wired. Set up state transitions
   (Idle <> Walk on speed > 0.1, Attack on input, etc.).
9. Press Play. Animations work.

Time: ~10 min from saving Aseprite to running game.

## Layer Import options

### Merge Frame (default)
Each Aseprite frame is merged into a single Unity Sprite. Layers
flattened. Best for most use cases.

### Group Layer
Each Aseprite layer becomes a separate Unity Sprite per frame. Useful
when you want to:
- Animate body and accessories separately.
- Apply different shaders to different layers (e.g., emissive cape).
- Swap accessories without re-rendering the whole sprite.

Adds complexity; use only when needed.

## Frame range filtering

You can import only specific tags:

Aseprite Importer > Frame Range:
- All Frames (default)
- Tagged Range: import only frames in these tags
- Custom: manually specify frame indices

Useful when the .aseprite file has many test frames you don't want in
Unity.

## Comparison: old workflow vs Aseprite Importer

### Old workflow (pre-importer)

1. Aseprite: File > Export Sprite Sheet > sprite_sheet.png + json.
2. Drop sprite_sheet.png into Unity.
3. Texture Type: Sprite, Multiple. Filter: Point.
4. Open Sprite Editor. Manual slice (auto by cell or manual rect per
   frame).
5. Create AnimationClips manually: for each animation:
   - Right-click Sprite > Create > Animation Clip.
   - Drag frames in.
6. Wire to Animator manually.

Per character: ~30-45 minutes. Painful for projects with 50+
characters.

### New workflow (Aseprite Importer)

1. Aseprite: save as .aseprite.
2. Drop into Unity.
3. Drag the auto-generated GameObject prefab into scene.

Per character: ~10 minutes. Repeatable.

## Common pitfalls

- **Forgot to install com.unity.2d.aseprite**: Unity treats .aseprite
  as unknown file. Install package first.
- **Filter Mode set to Bilinear / Trilinear**: pixel art looks
  blurry. Always Point.
- **Mip Maps enabled**: Unity generates blurry lower-res mip levels,
  used at zoom-out. Pixel art looks bad. Disable.
- **Compression > None**: pixel art compression artifacts. Set to None.
- **Tags not set in Aseprite**: importer falls back to "all frames in
  one clip". Not useful. Tag your frames.
- **Multi-layer sprites with Merge Frame**: layer effects flattened.
  Use Group Layer for layer-specific shaders.
- **Animator Controller overwrite on re-import**: changes to your
  manual state transitions get LOST when the .aseprite is re-imported.
  Solution: disable "Generate Animator Controller" after first import,
  manage manually.
- **Aseprite file path with non-ASCII characters**: importer can fail
  on some platforms.

## References

- Unity Aseprite Importer: https://docs.unity3d.com/Packages/com.unity.2d.aseprite@latest
- Aseprite documentation: https://www.aseprite.org/docs/
- Unity 2D Sprite system: https://docs.unity3d.com/Manual/Sprites.html
