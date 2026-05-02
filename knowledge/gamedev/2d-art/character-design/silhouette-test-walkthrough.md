# Silhouette Test Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/character-design`.
> Canonical source: Saint11 character design tutorials; "Designing characters: silhouettes" Pixel Joint
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/character-design/SKILL.md

## Concept

The silhouette test is the canonical way to evaluate character design
readability. Fill the character with pure black on a transparent
background, place at intended in-game scale, and ask: can I tell what
this is in 0.5 seconds? If no, the silhouette is wrong. No amount of
detail / color / shading will fix it.

## Walkthrough / mechanics

### The 5-step test

1. **Black-out**: in Aseprite, duplicate the character sprite. Apply
   "Color > Replace Color" to make all non-transparent pixels pure
   black.
2. **Place at game scale**: paste the silhouette at 1x scale (no
   zoom) onto a typical scene background.
3. **Time-test**: glance for 0.5 seconds. Can you tell?
   - Race / species? (human / elf / monster?)
   - Class / role? (warrior / mage / thief?)
   - Action state? (attacking / idle / running?)
   - Specific character vs generic? (named hero or one of many?)
4. **Comparison**: place silhouette next to other character silhouettes
   (allies, enemies). Are they distinguishable?
5. **Distance test**: zoom out to "viewing from across map". Can you
   still tell character apart from environment?

If pass all 5: silhouette works. If fail any: redesign at silhouette
level before adding details.

### Common readability failures

#### Generic blob
Symptom: black silhouette is a vague humanoid lump. No distinctive
features.

Fix: add at least one strong distinguishing element — hat, weapon,
cape, hair, special pose, asymmetric accessory.

#### Symmetric design
Symptom: silhouette looks the same flipped left-right. Feels
"facing nowhere".

Fix: asymmetric details. Weapon on one side, satchel on other.
Different left/right hairline.

#### Indistinct extremities
Symptom: hands and feet blend with body. Silhouette looks
"melted" or "blob".

Fix: paint clear separation between body and limbs (negative space
gap, distinct silhouette break).

#### Same as enemy
Symptom: hero silhouette indistinguishable from enemy silhouettes
(when both blacked out).

Fix: differentiate via body proportion, accessories, weapon types.
Heroes typically have clear "good guy" features (cape, banner,
distinctive hat).

### Diagnostic process

When silhouette fails, work in this order:

1. **Pose**: change idle pose to be more distinctive. Lean weight,
   turn head, raise arm.
2. **Major shape**: add hat / cape / huge weapon that defines the
   silhouette outline.
3. **Asymmetry**: add asymmetric element to give "facing direction".
4. **Negative space**: add gaps in silhouette (between arms and torso,
   between legs).
5. **Proportions**: exaggerate one feature (large head, small body, or
   inverse) for memorable feel.

Each step reduces "blobbiness" and increases readability.

## Worked example

Designing a wizard character. Initial silhouette failed: looked like
generic robed figure.

Iteration 1: added pointed hat. Better but still generic "wizard".

Iteration 2: gave him a long crooked staff (asymmetric, defines a
clear pose). Now silhouette: "wizard with staff held in left hand,
hat tilted right".

Iteration 3: added a flowing cape that shifts wind-blown to the
left. Negative space between body and cape. Silhouette: "wizard
with staff, hat, billowing cape — facing right".

Comparison test: alongside a knight (silhouette has square shoulders +
sword), thief (silhouette is hooded + crouched), the wizard's
silhouette is now clearly distinct. Pass.

## Tools for silhouette testing

### Aseprite
- Duplicate sprite.
- Replace Color: select a color → "Replace with: #000000".
- Or use "Layer > Layer Properties > Tint = black" if you need to
  preserve original.

### Krita / Photoshop
- Select all opaque pixels. Fill with black.

### Custom workflow
Aseprite Lua script that auto-generates silhouette layer:

```lua
local sprite = app.activeSprite
local layer = sprite:newLayer()
layer.name = "silhouette"
for f = 1, #sprite.frames do
    local cel = layer:newCel(layer, f, sprite.bounds, Image(sprite.spec))
    -- copy original pixels, replace with black where alpha > 0
    -- ... (full Lua API code)
end
```

## In-game silhouette readability

The silhouette test should be done with character at intended game
scale. A 32x32 character on a 1080p monitor is tiny. Test in those
conditions.

Additionally:
- **Camera distance varies** (zoomed out = smaller silhouette). Test
  at typical zoom levels.
- **Background varies**: silhouette against forest vs against sky vs
  against snow. Test against several typical backgrounds.

A character designed for snow background may fail against forest
because the dark silhouette blends with dark trees.

## Outline policy and silhouette

If your character has selout / inline outlines, verify the OUTLINED
sprite has same readability as black silhouette. Outline color
choices matter.

In dark scenes: outline color is critical for readability — it's the
edge that separates character from background.

## Common pitfalls

- **Skipping the silhouette test**: drawing details first, then
  realizing readability is poor at scale. Wasted time.
- **Designing for static portrait, not gameplay scale**: portrait at
  64x64 looks great; in-game at 16x16 unreadable.
- **Silhouette test on flat color, not against game scenes**: hero
  silhouette is fine alone, but blends with forest in-game. Test
  in actual contexts.
- **Same silhouette as common enemy**: confuses gameplay. Differentiate.
- **Symmetric silhouette = no facing direction**: in 2D top-down /
  side-scroll games, characters need clear orientation. Asymmetry
  delivers.
- **Tiny details visible at silhouette stage**: tiny details vanish
  at game scale. Focus on overall shape, not pixel-level.

## References

- Saint11 character design: https://saint11.org/
- "Designing characters: silhouettes" Pixel Joint forum
- Stardew Valley character art studies (community)
- Hollow Knight character design retrospectives (Team Cherry)
- Game Maker's Toolkit "Designing characters" video series
