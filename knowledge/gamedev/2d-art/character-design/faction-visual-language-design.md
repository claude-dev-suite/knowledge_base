# Faction Visual Language Design - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/character-design`.
> Canonical source: "Designing the World of Hollow Knight" Team Cherry; Riot Games faction design talks
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/character-design/SKILL.md

## Concept

Players parse character roles and factions in milliseconds when the
art applies a **consistent visual language**. Each faction (or class,
or role) shares silhouette traits, color palette, accessories, and
shape language. Once players learn the language, every new character
instantly communicates "ally / enemy / boss / merchant / friendly NPC"
without text.

## Walkthrough / mechanics

### Visual language dimensions

Each faction is defined along these dimensions:

| Dimension | Examples |
|-----------|---------|
| **Color palette** | Faction A = blue+silver, Faction B = red+gold |
| **Silhouette family** | Knights = helmets + capes; Mages = hoods + staves; Rogues = masks + daggers |
| **Shape language** | Round shapes = friendly; Angular = aggressive; Sharp = villainous |
| **Detail density** | Heroes = detailed; Mooks = simpler; Bosses = ornate |
| **Pose** | Confident posture vs defensive vs threatening |
| **Accessories** | Banners, sigils, talismans, capes — all coded |

### Faction rules to set early

Before designing characters, decide:

1. **Color codes**: each faction = 1-2 distinctive colors that no
   other faction uses prominently.
2. **Silhouette signature**: one shape element that all members of
   faction share (a particular helm shape, hood style, weapon type).
3. **Material**: each faction prefers certain materials (Royal
   Guard = polished steel, Cult = bone + hide, Magi = silk + gold).
4. **Sigil / emblem**: a small icon shown on banners, shields,
   capes — repeats across faction.

Define these in a 1-page design doc. Apply to every character.

### Encoding threat

Within faction, individual character roles encode threat level:

- **Generic mook**: simplest silhouette, smallest size, basic palette.
- **Veteran**: same silhouette + 1 distinctive feature (scar, bigger
  weapon, more armor).
- **Elite / officer**: ornate accessories, slightly larger size,
  cape or feature.
- **Boss**: distinctive silhouette breaking the family rules,
  oversized, signature color saturated.

Each tier reads instantly: smaller + plain = "easy", bigger + ornate =
"dangerous".

## Worked example

Designing 4 factions for an action RPG.

### 1. Royal Guards (lawful, hero allies)
- **Color palette**: blue (#3060c8) + silver (#c0c0d0) + gold accents (#d8b830).
- **Silhouette signature**: square shoulders, tall full-helm with
  crest plume.
- **Shape language**: angular but symmetric.
- **Material**: polished plate armor.
- **Sigil**: golden crown emblem on tabard.
- **Threat tiers**: rookie = scout helm; veteran = full helm + spear;
  captain = crest plume + cape.

### 2. Bandits (chaotic, common enemies)
- **Color palette**: muddy reds (#8a3c1c) + brown (#604030) + dirty
  white.
- **Silhouette signature**: hood pulled low, cloth wrapped face, light
  armor.
- **Shape language**: asymmetric, layered.
- **Material**: leather + cloth, mismatched.
- **Sigil**: red rag tied around arm.
- **Threat tiers**: thief = simple hood + dagger; raider = patched
  armor + club; chief = horned helm + axe.

### 3. Cult of Bone (corrupted, religious enemies)
- **Color palette**: bone white + black + sickly green.
- **Silhouette signature**: hooded robes with high pointed cowl,
  bone ornaments.
- **Shape language**: asymmetric, draped.
- **Material**: cloth + bone fragments.
- **Sigil**: skull eye-socket emblem.
- **Threat tiers**: acolyte = simple robe; priest = bone collar +
  staff; high-priest = bone crown + glow.

### 4. Forest Spirits (neutral, environmental)
- **Color palette**: greens + earth + glowing accent.
- **Silhouette signature**: organic, leafy, tree-like.
- **Shape language**: round, flowing.
- **Material**: bark + leaves + moss.
- **Sigil**: leaf or flower in center of body.
- **Threat tiers**: sapling = small + cute; warden = larger + horn;
  ancient = tree-sized + glowing eyes.

Player learns these languages in 2-3 encounters. From then on, sees
hooded figure → "bandit, attack". Sees blue + silver → "guard,
friendly". Sees bone + green → "cult, dangerous".

## Implementation tips

### Use palette swap for variants
A "guard" faction might have 4 sub-variants (rookie / regular /
veteran / captain). Author one base sprite, swap palettes for tiers.

```
Tier         Primary    Secondary  Accent
Rookie       light blue silver     -
Regular      blue       silver     -
Veteran      dark blue  silver     gold trim
Captain      blue       silver     gold full + cape
```

Same silhouette base, palette swap encodes hierarchy.

### Decoration overlays
Authoring base + accessory decorations (crest plume, cape, sash) as
separate sprites that overlay. Allows mix-and-match for variant
generation.

### Animation language
Each faction can have signature animations:
- Guards: rigid march, formal salute idle.
- Bandits: slouchy walk, swagger.
- Cult: floating glide (no walk), prayer pose idle.
- Spirits: organic sway, dancing motion.

Same character moving in faction-style instantly tells you "this is a
guard, this is a bandit". Doubles the visual language.

## Common pitfalls

- **Inconsistent application**: 80% of guards use the visual language
  but 20% don't. Players notice; faction breaks.
- **Color overlap between factions**: faction A and B both use blue
  prominently. Players confuse them.
- **No threat tier differentiation within faction**: all bandits look
  identical → players can't tell weak from strong.
- **Faction language too subtle**: details are too small to read at
  game scale. Apply to silhouette + palette, not just texture details.
- **Random one-off variants**: a single guard with red armor breaks
  language. Either commit to palette or design specifically for this
  variant (e.g., "captain in red").
- **Boss breaking faction language**: bosses should evolve faction
  language, not abandon it. A bone-cult boss is still bone-themed,
  just super-sized.
- **Faction language too rigid**: every member identical. Add slight
  variation to avoid mannequin feel (different hair, scars, posture).

## Design document template

For each faction:
```
Faction: [name]
Role: [hero/villain/neutral]
Threat level: [low/medium/high/boss-tier]

Color palette (1-3 dominant colors):
Silhouette signature: [one shape element]
Shape language: [round/angular/asymmetric/symmetric]
Material: [steel/cloth/bone/wood/etc.]
Sigil: [emblem description]
Animation style: [posture, movement quality]

Threat tiers:
- Tier 1 (mook): [base + minimal]
- Tier 2 (regular): [+1 feature]
- Tier 3 (elite): [+ornate]
- Tier 4 (boss): [signature size + saturated color + breaks rules slightly]
```

Fill out before designing first character. Apply rigorously.

## References

- Riot Games "Reading Champion Silhouettes" GDC talk
- "Designing the World of Hollow Knight" Team Cherry interview
- Stardew Valley NPC visual differentiation (community studies)
- Game Maker's Toolkit "Designing characters" video series
- Saint11 character design tutorials: https://saint11.org/
