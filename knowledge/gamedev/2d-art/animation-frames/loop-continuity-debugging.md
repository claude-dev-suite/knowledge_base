# Loop Continuity Debugging - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/animation-frames`.
> Canonical source: Aseprite onion skin documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/animation-frames/SKILL.md

## Concept

A looping animation must transition seamlessly from its last frame to
its first. Common bug: animation plays forward 1-8 cleanly, then
"jumps" or "hitches" between F8 and F1. This article shows how to
detect and fix loop discontinuities.

## Walkthrough / mechanics

### The 3-step debug process

1. **Visual onion skin verification**: in Aseprite, enable Onion Skin
   showing frames before AND after current. Each frame should appear
   as a smooth interpolation of the surrounding frames. Specifically,
   F1 should look like a natural continuation of F8.

2. **Real-time loop preview**: play animation in preview, watch for
   stuttering. A loop hitch shows as a brief "pause" at the F8->F1
   transition.

3. **Frame-by-frame comparison F1 vs F8**:
   - F1 (start) and F8 (end) should be similar but NOT identical.
   - If F1 == F8: the animation has 7 effective frames + a duplicate.
     The loop appears to "freeze" briefly.
   - If F1 differs heavily from F8: jumps visible.

### What "smooth loop" looks like

For an 8-frame walk:
- F1 = right contact
- F2 = down position
- F3 = passing
- F4 = up position
- F5 = left contact
- F6 = down position
- F7 = passing
- F8 = up position

The natural cycle is F1→F2→...→F8→F1→... where F8 (up position) leads
into F1 (right contact). Position vertically: F8 was at peak Y, F1
should be slightly lower (descending toward contact). NOT at peak Y
again.

If your F1 has the same Y position as F8: loop "freezes" at the top
and doesn't have momentum into the downstroke.

### Common cause: F1 designed as "key pose"

Many artists design F1 as a beautifully composed standing pose
(thinking of it as the animation's "title card"). But F1 is actually
just the next frame after F8 in the loop — it should feel like a
continuation.

**Fix**: design F1 in the context of the loop. F1 should be
"frame after up-position" not "neutral pose".

## Worked example

Walk cycle has visible hitch at loop boundary. Diagnosis:

```
F8 vertical position: y=4 (peak)
F1 vertical position: y=4 (peak, same!)
F2 vertical position: y=2 (descending)
```

F8 and F1 are at the same Y → 1-frame "freeze" at peak. Fix: lower F1
to y=2 (already starting to descend). Now:

```
F8: y=4 (peak)
F1: y=2 (descending)
F2: y=0 (lowest, contact)
```

Smooth loop. Same applies to arms, body angle, hair, etc.

## Aseprite-specific debugging

### Onion skin parameters
- View > Onion Skin > Show Past + Future
- Range: 1-2 frames each direction
- Opacity: ~30% past, 30% future

Stand on F1: see F8 (faint behind) and F2 (faint ahead). F1 should be
between F8's pose and F2's pose.

### Frame timing test
- Play animation
- Right-click each frame > Frame Properties > increase duration to 200ms
  temporarily
- Step through slow-motion to spot hitches

### Onion skin LOCKED to specific frames
For complex animations, use locked onion skin:
- View > Onion Skin > Settings > "Show frames behind/ahead even in
  selected layer only"

## Continuity checklist

For each animated element:
- ✓ Vertical position trajectory smooth (no double-peak, no flat
  section).
- ✓ Body rotation smooth.
- ✓ Limb positions interpolate cleanly.
- ✓ Hair/cape position interpolates (no "snapping back").
- ✓ Sprite anchor consistent (no drift).
- ✓ Effects (sparks, dust) not "popping in" mid-loop.

## Cycle drift bug

Symptom: after playing the cycle 5 times, the character has visibly
moved. Cause: root motion baked into the sprite.

**Fix**: anchor cycle around fixed point. The cycle should NOT
include forward movement; movement comes from gameplay code (player
position += speed * deltaTime). Sprite cycles in place.

Verify in Aseprite: place a guide line at character's anchor pixel.
Check each frame has the anchor at the same position.

## Common pitfalls

- **F1 == F8** (duplicate): visible freeze. Make them similar but
  not identical.
- **F1 is "title pose"** (neutral standing): doesn't fit loop context.
- **Cycle drift** (sprite moves over time): root motion in sprite.
- **Off-by-one frame count** (declared 8 frames, actually 9 or 7):
  loop math wrong. Verify Aseprite frame count matches engine import.
- **Animation freeze on game stop**: engine paused mid-cycle, no
  graceful return to F1. Set transition to "stay on last frame" then
  manually animate back.
- **Inconsistent timing per frame**: F1=100ms, F2=80ms, F3=100ms — the
  "rhythm" feels broken. Either consistent timing or deliberate easing.

## Tools

- **Aseprite Onion Skin** (free with Aseprite).
- **Spine** has more advanced loop curves and motion graphs.
- **Anim Studio** desktop tool — visualizes timing curves.

## References

- Aseprite Animation tutorial: https://www.aseprite.org/docs/animation/
- "Walk cycles for pixel art" by Saint11: https://saint11.org/
- "12 principles of animation" applied to pixel art (various
  community articles)
