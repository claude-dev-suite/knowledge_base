# Sprite Library & Sprite Resolver — Runtime Outfit Swap

> Source: https://docs.unity3d.com/Packages/com.unity.2d.animation@latest/manual/SpriteLibrary.html

Swap sprites at runtime (skins, outfits, weapons) without re-rigging or duplicating animations. The Sprite Library defines named buckets of sprites; the Sprite Resolver picks one at runtime.

## Sprite Library Asset

`Assets > Create > 2D > Sprite Library Asset`. Inside, define **Categories** with **Labels**:

```
Category: Hair
  Label: Brown   -> hair_brown.png
  Label: Blue    -> hair_blue.png
  Label: Red     -> hair_red.png

Category: Torso
  Label: Default -> torso_default.png
  Label: Armor   -> torso_armor.png
  Label: Robe    -> torso_robe.png
```

## Sprite Library component

On the character root, add `SpriteLibrary` and assign the asset.

## Sprite Resolver

On each body part GameObject (instead of using the SpriteRenderer's `Sprite` field directly):

```
SpriteRenderer
SpriteResolver  →  Category: Hair, Label: Brown
```

Sprite Resolver writes the selected sprite into the SpriteRenderer at runtime — and on every category/label change.

## Runtime swap

```csharp
public class CharacterCustomization : MonoBehaviour {
    [SerializeField] private SpriteResolver hairResolver;
    [SerializeField] private SpriteResolver torsoResolver;

    public void SetHair(string label) {
        hairResolver.SetCategoryAndLabel("Hair", label);
    }
    public void SetTorso(string label) {
        torsoResolver.SetCategoryAndLabel("Torso", label);
    }
}
```

The skinned mesh (bone weights) is shared — you're just swapping the texture rendered through the existing skin.

## Sprite Library Asset Override

Sometimes you want characters that mostly share a base library but override a few categories (e.g. Boss has unique armor):

```
Sprite Library Asset (BaseLibrary)
   Hair, Torso, Legs, Arms

Sprite Library Asset (BossOverride) — uses BaseLibrary as Source Library + adds:
   Torso (overrides — "Default" label points to boss_torso.png)
```

Assign `BossOverride` on the boss prefab's `SpriteLibrary` component — falls back to `BaseLibrary` for any unspecified category.

## Animation interaction

Animation clips don't have to know about library entries — they animate bone transforms only. Sprite changes happen via the resolver based on game state, decoupled from the animator. If you DO want animation to drive sprite swaps (e.g. a 4-frame attack with unique sprite per frame), key the Sprite Resolver's Label field in the clip.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Duplicating prefab + animator for each skin | One prefab, swap library or resolver labels |
| Hardcoding sprite paths via `Resources.Load` | Sprite Library + Resolver |
| Not using override libraries for boss-specific sprites | Use Source Library + override pattern |
| Forgetting `Apply` after editing library asset | Always Apply — otherwise library is dirty in editor only |
