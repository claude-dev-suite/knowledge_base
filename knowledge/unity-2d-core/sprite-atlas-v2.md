# Sprite Atlas v2 — Packing & Variants

> Source: https://docs.unity3d.com/Manual/sprite-atlas.html

Sprite Atlas v2 batches many sprites into a single texture so they share a draw call.

## Create

`Window > 2D > Sprite Atlas` → drag folders or sprites into the **Objects for Packing** list.

```
Type                Master  (or Variant)
Include in Build    ON
Allow Rotation      OFF for UI / pixel art (rotation breaks alignment)
                    ON for general-purpose to save space
Tight Packing       ON  → packs by sprite mesh (more density, slightly slower pack)
                    OFF → packs by rectangle (faster pack, more wasted space)
Padding             4 px (avoids bleed at lower mips)
Read/Write          OFF for shipping
Generate Mip Maps   OFF for 2D unless using mip-aware shaders
```

## Master vs Variant

A **Variant** atlas references a Master and re-packs at a different scale. Common case: ship a `MasterAtlas` (full-res) and a `MasterAtlas_Mobile` Variant at 0.5x scale. Toggle which is included by build target via Asset Bundles or Addressables labels.

## Late binding

For Addressables-driven content, wire `SpriteAtlasManager.atlasRequested`:

```csharp
SpriteAtlasManager.atlasRequested += (atlasName, callback) => {
    StartCoroutine(LoadAtlasAsync(atlasName, callback));
};
```

The first time a sprite from an unloaded atlas is requested, this fires; load via Addressables, hand back the atlas via `callback(atlas)`.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Atlas with `Include in Build = OFF` | Tick it; otherwise sprites still ship as individual textures |
| Mixing tight + rotation on UI atlases | Off both for UI — sprites must stay axis-aligned |
| Master + Variant scale that breaks PPU | Keep Variant scale to clean fractions (0.25, 0.5) and verify PPU consistency |
| One atlas per scene | Group by frequency-of-use (always-loaded, per-level, rare) |
