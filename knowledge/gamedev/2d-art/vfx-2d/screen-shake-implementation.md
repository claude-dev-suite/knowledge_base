# Screen Shake Implementation - Deep Dive

> Phase B article. Companion to dev-suite skill `gamedev/2d-art/vfx-2d`.
> Canonical source: Unity Cinemachine Impulse documentation; "The Art of Screenshake" Vlambeer talk
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/gamedev/2d-art/vfx-2d/SKILL.md

## Concept

Screen shake = brief random offset to camera position over a short
duration. Critical for game feel: hits, explosions, landings, and
boss damage all benefit. Implementation has 3 dimensions: magnitude
(pixels of offset), frequency (shakes per second), and decay (how
intensity falls over time). Tuning each correctly = readable
non-nauseating shake.

## Walkthrough / mechanics

### Manual implementation (custom)

```csharp
public class ScreenShake : MonoBehaviour {
    Vector3 originalPos;
    float duration = 0;
    float magnitude = 0;
    float frequency = 30; // shakes per sec

    void LateUpdate() {
        if (duration > 0) {
            duration -= Time.deltaTime;
            float t = 1.0f - duration / 0.3f; // assuming 300ms total
            float intensity = (1.0f - t * t) * magnitude;  // quadratic decay
            
            float angle = Random.Range(0, Mathf.PI * 2);
            float r = Random.Range(0, intensity);
            transform.position = originalPos + new Vector3(
                Mathf.Cos(angle) * r,
                Mathf.Sin(angle) * r,
                0
            );
        } else {
            transform.position = originalPos;
        }
    }

    public void Shake(float mag, float dur) {
        magnitude = mag;
        duration = dur;
    }
}
```

LateUpdate ensures shake happens AFTER other camera movement. originalPos
captured before shake starts.

### Cinemachine Impulse (Unity, recommended)

Cinemachine has built-in Impulse system. Setup:

1. **Cinemachine Impulse Source** on event-causing GameObject (sword,
   explosion).
2. **Cinemachine Impulse Listener** on Cinemachine Virtual Camera.
3. Configure:
   - Source: Impulse Type, Default Velocity (vector), Time Envelope
     (curve over time).
   - Listener: Channel Mask, Gain, Use 2D Distance.

Trigger with code:
```csharp
[SerializeField] CinemachineImpulseSource impulseSource;

void OnHit() {
    impulseSource.GenerateImpulseWithVelocity(new Vector3(0, -1, 0));
    // velocity = direction + magnitude
}
```

Cinemachine handles decay envelope automatically.

### Decay curves

```
Linear:     intensity * (1 - t / duration)
Quadratic:  intensity * (1 - t / duration)^2  (snappier)
Cubic:      intensity * (1 - t / duration)^3  (very fast falloff)
Bounce:     oscillating with decaying envelope
```

Quadratic is the indie standard — feels "snappy" without being abrupt.

### Frequency

```
Low frequency (15 Hz):   "rumbling" feel, like distant earthquake
Medium (30 Hz):           standard hit/explosion feel
High (60+ Hz):            sharp jitter, like high-energy impact
Random per-frame:         chaotic but per-frame depends on framerate
```

For 60fps games: 30 Hz frequency = shake position changes every 2
frames. Higher = changes every frame = chaotic.

## Worked example

Calibrating screen shake for a 2D platformer's actions.

### Footstep (light)
- Magnitude: 0 (no shake) — too small to matter.
- Don't shake on footsteps.

### Light attack hit
- Magnitude: 2 px.
- Duration: 100 ms.
- Frequency: 30 Hz.
- Decay: quadratic.
- Direction: omni-directional.

### Heavy attack hit
- Magnitude: 5 px.
- Duration: 200 ms.
- Frequency: 30 Hz.
- Decay: quadratic.
- Direction: along hit normal (toward enemy).

### Player land from high jump
- Magnitude: 3 px.
- Duration: 150 ms.
- Direction: vertical only.

### Explosion
- Magnitude: 8 px.
- Duration: 500 ms.
- Frequency: 25 Hz (lower for "rumbling" feel).
- Decay: quadratic.

### Boss death
- Magnitude: 12 px.
- Duration: 1500 ms.
- Frequency: 20 Hz (heavy rumbling).
- Decay: very slow quadratic.
- Plus camera punch + chromatic aberration + slow-mo.

### Earthquake (gameplay event)
- Magnitude: 4 px.
- Duration: continuous (start/stop on event).
- Frequency: 15 Hz.
- Decay: none while active.

## Direction strategies

### Omni-directional (default)
Random angle each shake. Works for "general impact" feel.

### Directional (hit normal)
Shake along the direction of attack. More grounded — communicates
"you got hit FROM there".

```csharp
Vector3 hitNormal = (target.position - source.position).normalized;
impulseSource.GenerateImpulseWithVelocity(hitNormal * magnitude);
```

### Vertical only (landing, jump)
Constrain to Y axis. Feels like "ground impact".

### Horizontal only (rare, dash)
Constrain to X axis. Feels like "lateral force".

## Common pitfalls

- **Magnitude too high**: nausea-inducing. Cap at 8-10 px for most
  games.
- **Duration too long**: prolonged shake feels like a bug, not feedback.
  100-300ms typical.
- **Constant shake** (always shaking): player adapts and stops noticing.
  Reserve shake for IMPORTANT events.
- **Shake stacking** (multiple shakes overlap): sum > single max,
  becomes nauseating. Cinemachine Impulse handles this; manual
  implementations need to clamp.
- **Forgetting LateUpdate**: shake conflicts with player camera follow.
  LateUpdate ensures shake applies after follow.
- **Shake breaks pixel-perfect camera**: if camera position must be
  integer, shake offset must round. Or shake the rendered image
  in post-process, not camera position.
- **Mobile motion sickness**: keep mobile shake aggressive than desktop.
  Cap magnitude < 5 px on mobile.

## Pixel-perfect camera + shake

If using Pixel Perfect Camera:

- Shake the OFFSET applied AFTER pixel snap.
- Round shake offset to integer pixel before applying.
- Or shake at sub-pixel and let pixel-perfect snap absorb (but breaks
  intent — sub-pixel shake invisible).

Best practice: apply shake offset, round to integer pixel.

```csharp
Vector3 shakeOffset = ComputeShake();
shakeOffset.x = Mathf.Round(shakeOffset.x * pixelsPerUnit) / pixelsPerUnit;
shakeOffset.y = Mathf.Round(shakeOffset.y * pixelsPerUnit) / pixelsPerUnit;
camera.position = baseCameraPos + shakeOffset;
```

## References

- Unity Cinemachine Impulse: https://docs.unity3d.com/Packages/com.unity.cinemachine@latest/manual/CinemachineImpulse.html
- "The Art of Screenshake" Vlambeer: https://www.youtube.com/watch?v=AJdEqssNZ-U
- "Juicing your Cameras" Steve Swink chapter
- Game Maker's Toolkit "Game Feel" video on shake
