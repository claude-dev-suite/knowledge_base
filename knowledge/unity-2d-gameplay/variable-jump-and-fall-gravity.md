# Variable Jump Height & Asymmetric Gravity

> Source: https://docs.unity3d.com/ScriptReference/Rigidbody2D-gravityScale.html

Two tricks that shape jump arcs:

1. **Variable jump height** — release Jump button early to cut the rise short.
2. **Asymmetric gravity** — fall faster than you rise, for snappy descent.

Together they make jumps feel responsive and precise.

## Variable jump (early release)

```csharp
[SerializeField] private float lowJumpGravityMul = 2.5f;
private bool _earlyRelease;

void Update() {
    if (_earlyRelease && _rb.linearVelocity.y > 0f) {
        _rb.gravityScale = _baseGravity * lowJumpGravityMul;   // crank gravity to abort rise
    }
    _earlyRelease = false;     // single-frame flag
}

public void OnJumpReleased(InputAction.CallbackContext _) => _earlyRelease = true;
```

Wire `OnJumpReleased` to the Input System's `canceled` event. Hold = full jump; tap = small hop.

## Asymmetric (fall) gravity

```csharp
[SerializeField] private float fallGravityMul = 2f;

void Update() {
    if (_rb.linearVelocity.y < 0f) {
        _rb.gravityScale = _baseGravity * fallGravityMul;     // snappier fall
    }
    else if (_earlyRelease && _rb.linearVelocity.y > 0f) {
        _rb.gravityScale = _baseGravity * lowJumpGravityMul;
    }
    else {
        _rb.gravityScale = _baseGravity;
    }
    _earlyRelease = false;
}
```

Order matters: fall gravity check first (most common state), then early release, then default.

## Tuning sample (good defaults)

```
gravityScale (base)      1
jumpVelocity            14
fallGravityMul          2.0    -> falls feel punchy without being unfair
lowJumpGravityMul       2.5    -> tap = small hop ~ 60% of full
coyoteTime              0.1
jumpBuffer              0.1
```

Increase `fallGravityMul` for arcade feel (Mario-like); decrease for floaty (Sonic Adventure).

## Apex hangtime (advanced)

For extra "stick to the apex" feel, lower gravity briefly when vertical velocity is near zero:

```csharp
void Update() {
    float vy = _rb.linearVelocity.y;
    if (Mathf.Abs(vy) < apexThreshold) {
        _rb.gravityScale = _baseGravity * apexGravityMul;     // ~ 0.5
    } else if (vy < 0f) {
        _rb.gravityScale = _baseGravity * fallGravityMul;
    } else if (_earlyRelease && vy > 0f) {
        _rb.gravityScale = _baseGravity * lowJumpGravityMul;
    } else {
        _rb.gravityScale = _baseGravity;
    }
}
```

`apexThreshold = 1.0f` is a good start. Players land aerial maneuvers more easily because the apex feels less sudden.

## Why gravityScale (not custom integration)

Unity's built-in physics already integrates gravity each FixedUpdate. Hijacking via `gravityScale` lets the engine handle the math; you just modulate the constant. Custom integration via `_rb.AddForce(Vector2.down * customG)` is also fine but loses interaction with effectors / joints.

## Capping fall speed

Long falls exceed reasonable speeds. Clamp:

```csharp
[SerializeField] private float maxFallSpeed = 18f;
void Update() {
    if (_rb.linearVelocity.y < -maxFallSpeed) {
        var v = _rb.linearVelocity;
        v.y = -maxFallSpeed;
        _rb.linearVelocity = v;
    }
}
```

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Same gravity on rise and fall | Multiply fall gravity for snappier descent |
| Variable jump that ALSO works on jump-button-not-yet-pressed | Gate by `_rb.linearVelocity.y > 0f` |
| Setting gravityScale every frame even when no change | Branch first; assign only on transition |
| No fall speed cap on tall levels | Cap at ~18..22 m/s |
| Tweaking jumpVelocity to fix "too floaty" | Usually fall gravity needs tweaking, not jump velocity |
