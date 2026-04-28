# PSD Importer Workflow

> Source: https://docs.unity3d.com/Packages/com.unity.2d.psdimporter@latest

The PSD Importer reads layered PSB / PSD files and creates a multi-sprite asset that preserves layer hierarchy — ideal for skeletal 2D animation source files.

## Author in Photoshop

- Save as **PSB** (large document format) — supports >30000 px and very deep layer trees.
- Each body part on its own layer (`Head`, `Torso`, `Arm_L`, `Forearm_L`, `Hand_L`, etc.).
- Group layers logically (`Body/Arm_L/Upper`, `Body/Arm_L/Forearm`); the hierarchy maps to the Unity bone hierarchy.
- Set the document size to fit the unposed character with margin (no clipping at extremes).

## Import settings

```
Texture Type           Sprite (2D and UI)
Sprite Mode            Multiple
Mosaic                 ON     (packs each layer into the sprite atlas image)
Character Rig          ON     (creates a bone-rigged prefab from the layer tree)
Use Layer Group        ON     (preserves PSD group structure as bone hierarchy)
Reslice                OFF after the first import (lets you tweak slices without losing them)
Pixel Per Unit         match project-wide PPU
Filter Mode            Point  (pixel art) or Bilinear (HD art)
Compression            None for pixel art; ASTC for mobile HD
```

## Generated assets

Importing produces:

```
Hero.psb
  Hero (multi-sprite asset — open with Sprite Editor)
  Hero (Texture2D — packed atlas)
  Hero_Skeleton (Prefab — bone-rigged with all parts as children)
  Hero_Library (SpriteLibraryAsset, if Use Sprite Library = ON)
```

Drop the prefab into a scene → ready to animate.

## Round-trip workflow

1. Edit PSB in Photoshop — add a layer for a new piece of armor.
2. Save — Unity re-imports.
3. Reload Skinning Editor — Auto Weights for the new piece (or paint manually).
4. Apply — animation clips that already exist still work; the new part shows up at default pose.

Critical: keep layer **names stable**. Renaming a layer in Photoshop breaks the binding to its sprite identity in Unity.

## Common pitfalls

| Issue | Cause | Fix |
|---|---|---|
| "Couldn't find sprite" after PSB save | Layer renamed | Restore original name; PSD Importer matches by name + path |
| Bones lost after re-import | Character Rig was OFF | Re-enable; re-paint weights |
| Sprite atlas grew unexpectedly | Too many disconnected layers | Group them; use Reslice = OFF after initial slicing |
| Animations broken across re-import | Hierarchy changed (parts moved between groups) | Avoid restructuring; if needed, re-bind clips manually |

## Aseprite alternative

For pixel art, `com.unity.2d.aseprite` reads `.aseprite` files natively, with these benefits:
- Animation tags become AnimationClips automatically
- Pivots set per layer in Aseprite carry over
- Smaller files than PSB for low-res pixel art

Use Aseprite for pixel-art characters; PSB for hi-res / hand-drawn.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Editing layers in Photoshop without saving as PSB | Save as PSB; PSD has layer count limits PSB does not |
| Disabling Mosaic | Without it, each layer becomes a separate texture — many draw calls |
| Renaming layers between imports | Never; this breaks identity matching |
| Mismatched PPU between PSB and project | Always match — sub-pixel rendering follows |
