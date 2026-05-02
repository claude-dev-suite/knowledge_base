# Weight and Attitude in Walks - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/animation-frames`.
> Canonical source: "Walk cycles" tutorial by Saint11; Animator's Survival Kit
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/animation-frames/SKILL.md

## Concept

A walk cycle communicates more than "this character moves" — it
communicates **weight** (how heavy the character is) and **attitude**
(personality, mood, status). Two characters of similar size walking
differently feel like fundamentally different beings. The walk cycle
is one of the most expressive gameplay-art elements.

## Walkthrough / mechanics

### Weight indicators

A heavy character has:
- **Larger vertical bob**: body sinks more on contact, rises more
  on up-position. 2-3 px bob (vs 1 px for light).
- **Slower cadence**: each frame held longer. ~12-15 fps (vs 8-10 fps
  for light).
- **Pronounced contact pose**: knee bent more on contact, weight
  transmitted visibly into the ground.
- **Wider stance**: feet farther apart in contact poses.
- **Lower arm swing**: arms hang lower, swing tightly close to body.
- **Weighted recoil**: F2 (down position after contact) is deep.

A light character has:
- Small bob (0-1 px).
- Fast cadence (6-8 fps, fewer frames per cycle).
- Springy step: knee bent slightly, but body rebounds quickly.
- Narrow stance.
- High arm swing, energetic.
- Minimal recoil.

### Attitude indicators

| Attitude | Shoulder | Spine | Head | Arms |
|----------|----------|-------|------|------|
| Confident strut | Pulled back | Upright | Up + slightly back | Wide swing, hands relaxed |
| Sneaky | Hunched forward | Forward lean | Down + forward | Close to body, low |
| Tired | Slumped down | Slumped | Down | Hanging, minimal swing |
| Determined | Squared | Slight forward lean | Forward + level | Pumping forward+back, fist clenched |
| Wounded / hurt | One side hunched | Asymmetric lean | Tilted | Holding wound, asymmetric |
| Joyful | Slight up bounce | Upright bouncy | Up | Energetic asymmetric swing |

These describe the **idle stance** that informs walk frames. A
character's walk should embody their attitude in every frame.

### Cadence math

A walk cycle's frame rate determines apparent speed:

```
6 frames @ 8 fps = 750 ms cycle = "slow walk"
6 frames @ 12 fps = 500 ms cycle = "normal walk"
6 frames @ 15 fps = 400 ms cycle = "brisk walk"
8 frames @ 10 fps = 800 ms cycle = "heavy walk"
8 frames @ 18 fps = 444 ms cycle = "athletic walk"
```

Match cadence to character feel. Don't auto-pick "10 fps for everything".

## Worked example

Two characters in the same scene walking differently.

### Knight (heavy, confident)

- 8-frame cycle, 12 fps (per frame ~83 ms).
- Vertical bob: 2 px.
- F1 contact pose: feet 6 px apart, knee bent, weight on right foot.
- F2 down: -2 px below F1, deep crouch on impact.
- Arm swing: 2-3 px back-forward, sword in right hand stays mostly
  forward.
- Shoulder posture: pulled back, upright spine.

### Thief (light, sneaky)

- 6-frame cycle, 14 fps (per frame ~71 ms).
- Vertical bob: 1 px (low profile).
- F1 contact: feet 4 px apart, knee bent slightly.
- F2 down: -1 px below F1.
- Arm swing: minimal, arms close to body.
- Shoulder posture: hunched forward.
- Head: turning toward target (not straight ahead).

The two characters' walks look DRAMATICALLY different. The knight is
"big and proud", the thief is "small and watchful".

## Per-frame timing for weight

For heavier feel, use variable per-frame duration:

```
F1 (contact):    100 ms (held longer to emphasize weight)
F2 (down):       80 ms
F3 (passing):    80 ms
F4 (up):         60 ms (light moment, brisk)
F5-F8: mirror.

Total: 80 ms × 8 = 640 ms per cycle.
But heavier feel because contact is held longer.
```

Aseprite supports per-frame duration in the timeline.

## Walk cycle subtle details

### Head bob delay
The head should follow the body, not move in lockstep. If body bobs at
F2, head bobs at F2 (slight) and F3 (more). This "follow-through"
creates organic feel.

### Weight transfer
At F1 (right contact), weight should be ON the right foot. Left foot
should be lifting. At F4, weight transitions through center. At F5,
weight is on left foot.

This transfer is invisible at 8x8 but visible at 32x32+. Worth painting.

### Hair / cape secondary motion
Hair and cape lag behind the body. When body bobs up, hair stays
slightly behind, then catches up. When body falls, hair lifts up
briefly. Adds 30-50% perceived animation quality for ~10% extra work.

### Eye direction
In a 16x16 sprite, you have 1-2 pixels for eyes. Subtle eye direction
(looking forward, looking down at footing, looking around for threats)
shifts attitude dramatically.

## Common pitfalls

- **Same walk for every character**: feels generic. Differentiate.
- **Robotic step**: equal-time frames, no bob, no arm swing. Add bob
  and easing.
- **Cycle walks "in place"**: anchor offset. Fix root motion handling.
- **Attitude lost mid-cycle**: idle pose has personality, walk doesn't.
  Carry attitude into every frame.
- **Forgetting head bob delay**: body and head move together =
  artificial. Stagger head 1 frame behind body.
- **Inconsistent walk speed**: cycle plays at 12 fps but movement speed
  is for 18 fps walk. Sprite "skates" — feet don't sync with motion.
  Match cycle frame rate to movement speed.

## Movement speed sync

```
movement_speed = 100 px/sec
cycle_total_distance = 50 px (one full step)
cycle_duration = 50 / 100 = 0.5 sec = 500 ms

If cycle is 8 frames: each frame = 500/8 = 62.5 ms = ~16 fps
```

Use this to size frame timing. Otherwise sprite "ice-skates" (faster
than walks suggest) or "moonwalks" (cycle plays but no forward motion).

## References

- Saint11 walk cycle tutorials: https://saint11.org/
- "The Animator's Survival Kit" by Richard Williams (canonical book).
- Stardew Valley walk cycles (NPC differentiation): see source pixel
  art studies online.
- Aseprite per-frame duration documentation.
