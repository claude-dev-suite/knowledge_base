# Coyote Time & Jump Buffer

> Source: https://learn.unity.com/tutorial/2d-platformer-character-controller

Two "feel" features players notice subconsciously by their absence:

- **Coyote time** — small grace window after walking off a ledge during which the game still treats you as grounded. Compensates for human reaction time.
- **Jump buffer** — pressing jump shortly before landing still triggers a jump on touchdown.

Typical values: coyote 80–120 ms, buffer 80–150 ms. Without them, players feel the controls are fighting them.

## Implementation (counters in seconds)

```csharp
[Header("Jump")]
[SerializeField] private float coyoteTime = 0.1f;
[SerializeField] private float jumpBuffer = 0.1f;
[SerializeField] private float jumpVelocity = 14f;

private float _coyoteCounter;
private float _jumpBufferCounter;

void Update() {
    // Update coyote — full when grounded, ticks down otherwise
    if (_grounded) _coyoteCounter = coyoteTime;
    else           _coyoteCounter -= Time.deltaTime;

    // Update buffer — refreshed by RequestJump(), ticks down otherwise
    if (_jumpBufferCounter > 0f) _jumpBufferCounter -= Time.deltaTime;

    // Execute jump if both windows are active
    if (_jumpBufferCounter > 0f && _coyoteCounter > 0f) {
        var v = _rb.linearVelocity;
        v.y = jumpVelocity;
        _rb.linearVelocity = v;
        _jumpBufferCounter = 0f;
        _coyoteCounter = 0f;
    }
}

public void RequestJump() => _jumpBufferCounter = jumpBuffer;
```

`RequestJump()` is called from input — could be from `InputAction.performed`, or polling `WasPressedThisFrame()`.

## Why two counters

```
Player jumps just before landing:                Player jumps just after walking off ledge:

  airtime --> [Buffer countdown]                   grounded --> walk off --> [Coyote countdown]
  land --> [Buffer still > 0] --> JUMP            press jump --> [Coyote still > 0] --> JUMP
```

Both make the input forgiving in opposite directions of the timing gap.

## Tuning by game feel

| Game style | Coyote | Buffer |
|---|---|---|
| Tight skill-based platformer (Celeste, Hollow Knight) | 100..120ms | 100..150ms |
| Casual platformer | 130..180ms | 130..180ms |
| Speedrun-friendly | 60..90ms (less forgiveness encourages skill) | 80..100ms |
| Arcade-y | 80..100ms | 80..100ms |

Visual cue: a player should NOT be able to jump from a ledge they CLEARLY left ages ago — too much coyote feels like double-jump cheating. 100ms is a sweet spot.

## Common pitfalls

| Issue | Cause | Fix |
|---|---|---|
| "I press jump but nothing happens" | Buffer tick happens before press detect | Update counters in same `Update` order: tick → consume |
| "I jumped twice with one press" | Buffer not zeroed after consume | Set `_jumpBufferCounter = 0f` AND `_coyoteCounter = 0f` after jump |
| "Jump feels delayed" | Buffer too long (>200ms) | Lower it; or add jump-on-press for grounded case (priority over buffer) |
| Coyote works during dash / wall slide | State leaks into "grounded" | Gate `_coyoteCounter = coyoteTime` behind `_grounded && !_dashing && !_wallSliding` |

## Variant: separate ground and ledge coyote

For more control, separate "still grounded" from "just left a ledge":

```csharp
bool justLeft = _wasGrounded && !_grounded;
if (justLeft) _coyoteCounter = coyoteTime;
_wasGrounded = _grounded;
```

This way coyote only triggers on falling-off-edge, not on bouncing off enemies (which interrupts grounded but shouldn't grant a free jump).

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Frame-based counters (`int frames = 6`) instead of time | Use `Time.deltaTime` so behaviour is fps-independent |
| Long coyote that lets player "ghost-jump" mid-air | Cap at ~150ms |
| Forgetting to reset counters after jump | Both must zero |
| Reading input in FixedUpdate | Input sample in Update; physics writes in FixedUpdate |
