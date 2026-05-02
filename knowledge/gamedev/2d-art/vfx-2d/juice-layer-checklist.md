# Juice Layer Checklist - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/vfx-2d`.
> Canonical source: "The Art of Screenshake" by Jan Willem Nijman (Vlambeer, GDC 2013)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/vfx-2d/SKILL.md

## Concept

"Juice" = the small visual + audio + game-feel layers that make every
action feel impactful. From Vlambeer's 2013 talk: every meaningful
event should layer 7-9 small effects, NOT just one. A single hit
without juice feels mechanical; a single hit with full juice layered
feels visceral. This article is a checklist for systematic juicing of
any meaningful event.

## Walkthrough / mechanics

### The 9 layers of juice

For any meaningful event (hit, jump, pickup, kill, level transition):

1. **Animation squash-stretch**: subtle 1-3 px deform of affected sprite.
2. **Color flash**: 1-3 frame pure white tint on hit sprite.
3. **Hitstop / time freeze**: 1-10 frame game pause on impact.
4. **Particle burst**: spawn 5-20 particles at event location.
5. **Screen shake**: directional or random shake, 100-300ms duration.
6. **Sound effect**: layered SFX (impact + foley + ambient reaction).
7. **Trail effect**: motion trail on moving objects (sword swing, dash).
8. **Decal / aftermath**: persistent visual mark (scorch, blood, dust).
9. **Camera response**: subtle zoom punch, position lerp toward event.

### Layer breakdown: hit on enemy

Event: player sword hits enemy.

1. **Animation**: enemy sprite squashes 1 px on impact.
2. **Color flash**: enemy sprite tints white for 2 frames.
3. **Hitstop**: game pauses 50-80ms (~4 frames at 60fps).
4. **Particles**: 8-12 small "spark" particles spawn at hit point,
   moving in arc.
5. **Screen shake**: 4-px omni shake, decay over 150ms.
6. **Sound**: sword "thwack" + enemy "grunt".
7. **Trail**: sword swing leaves brief curved trail.
8. **Decal**: small dust cloud at impact point, fading 500ms.
9. **Camera**: zoom in 5% for 100ms then ease back.

Total ~9 effects, each ~50-300ms duration, layered to perceive single
"impactful hit". Without any one layer, the hit feels less.

### Scaling intensity

Different event types want different juice intensity:

| Event | Layers | Intensity scale |
|-------|--------|----------------|
| Footstep | 1-2 (anim+sound) | low |
| Pickup coin | 3-4 (anim+particle+sound+brief flash) | low |
| Light hit | 5-7 (most layers, small) | medium |
| Heavy hit | 8-9 (all layers, medium) | high |
| Boss death | 9 (all layers, max intensity, screen-clearing) | extreme |
| Player death | 8-9 (red flash, slow-mo, big shake, screen wipe) | extreme |

## Worked example

Implementing juice for a "player jumps and lands" event in Unity:

```csharp
public class PlayerJuice : MonoBehaviour {
    [SerializeField] Animator anim;
    [SerializeField] ParticleSystem dustParticles;
    [SerializeField] CinemachineImpulseSource impulseSource;

    public void OnLand() {
        // 1. Animation squash
        anim.SetTrigger("Land");

        // 4. Particle burst (dust at feet)
        dustParticles.Emit(8);

        // 5. Camera shake via Cinemachine Impulse
        impulseSource.GenerateImpulseWithVelocity(Vector3.down * 0.5f);

        // 6. Sound
        AudioManager.Play("land", volume: 0.7f);

        // 9. Camera punch (separate Cinemachine extension)
        CameraPunch.Punch(zoomDelta: 0.05f, durationMs: 80);
    }
}
```

Layers 1, 4, 5, 6, 9 = 5 effects on a "jump land" event. Sufficient for
a non-critical event; full 9-layer treatment reserved for combat hits.

## Detecting "juice deficit"

A common bug: action looks "weak" or "floaty" or "unsatisfying".
Diagnostic: turn off each juice layer one at a time and see when the
problem becomes worst. The missing layer is your juice deficit.

Common deficits:
- **No hitstop**: hits feel "passes through" enemy without weight.
- **No screen shake**: action feels disconnected from world.
- **No color flash**: hits don't register visually as "happened".
- **No particles**: events feel sterile / clinical.
- **No sound**: events feel un-acknowledged.

## Anti-patterns

### Over-juicing
TOO much juice = annoying or nausea-inducing. Symptoms:
- Constant screen shake = motion sickness.
- Hitstop on every action = laggy feel.
- Particles everywhere = visual noise.

Calibrate: every layer subtle. Stack subtle layers for compound effect.

### Single-event juice
Putting all 9 layers on ONE event (e.g., boss death) and skipping
juice on smaller events (e.g., regular hit). Players don't get
constant feedback.

Solution: minor events get 3-4 layers; major events get 8-9.

### Static juice
Same juice on every event of the same type. Repetitive.

Solution: random variation. 3-5 hit-particle templates randomized.
Slight variance in hitstop duration (40-60ms range). Sound has
several variants pitched randomly.

## Common pitfalls

- **Forgetting to FORWARD time after hitstop**: game stays frozen.
  Always end hitstop with an explicit timer.
- **Hitstop affects EVERYTHING (particles freeze too)**: screenshake
  freezes mid-shake, particles look dead. Solution: hitstop pauses
  gameplay logic but NOT particles or shake.
- **Camera shake too aggressive**: nausea. Magnitude < 8 px typical.
- **Particles spawn but don't display (depth sorting)**: particle
  layer behind sprite. Set particle z = sprite z - 1.
- **Color flash uses wrong shader uniform**: tint applied as base
  multiply, not additive. Should ADD white to base color.
- **Bloom on color flash**: flash is "extra bright" so bloom amplifies
  → blinding flash. Either lower flash intensity or disable bloom on
  flash layer.

## References

- "The Art of Screenshake" by Jan Willem Nijman: https://www.youtube.com/watch?v=AJdEqssNZ-U (GDC 2013)
- Vlambeer game-feel articles
- "Game Feel" book by Steve Swink (canonical reference)
- Game Maker's Toolkit "Secrets of Game Feel and Juice" video
