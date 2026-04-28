# Dash & Wall Jump

> Source: https://learn.unity.com/tutorial/2d-platformer-character-controller

Two staple platformer abilities. Both share a pattern: temporarily override normal physics, restore on completion.

## Dash

```csharp
[Header("Dash")]
[SerializeField] private float dashSpeed = 24f;
[SerializeField] private float dashDuration = 0.15f;
[SerializeField] private float dashCooldown = 0.5f;

private bool _dashing;
private float _dashCooldownCounter;
private bool _hasDash = true;     // refilled on ground touch

public void RequestDash(Vector2 dir) {
    if (_dashing || _dashCooldownCounter > 0f || !_hasDash) return;
    StartCoroutine(DoDash(dir.sqrMagnitude > 0f ? dir.normalized : Vector2.right * Mathf.Sign(transform.localScale.x)));
}

private IEnumerator DoDash(Vector2 dir) {
    _dashing = true;
    _hasDash = false;
    _dashCooldownCounter = dashCooldown;

    // disable input movement, freeze gravity
    var savedGravity = _rb.gravityScale;
    _rb.gravityScale = 0f;
    _rb.linearVelocity = dir * dashSpeed;

    // optional: i-frames against enemies during dash
    _hitbox.gameObject.layer = LayerMask.NameToLayer("PlayerInvuln");

    yield return new WaitForSeconds(dashDuration);

    _rb.gravityScale = savedGravity;
    _rb.linearVelocity = new Vector2(_rb.linearVelocity.x * 0.5f, _rb.linearVelocity.y);  // bleed dash velocity
    _hitbox.gameObject.layer = LayerMask.NameToLayer("Player");
    _dashing = false;
}

void Update() {
    _dashCooldownCounter -= Time.deltaTime;
    if (_grounded) _hasDash = true;     // refill on ground
}
```

Key knobs:

| Field | Effect |
|---|---|
| `dashSpeed` | Higher = more dramatic; tune so dash distance ~3 player widths |
| `dashDuration` | Short (100..200ms) = sharp; longer feels floaty |
| `dashCooldown` | How often it refreshes after a dash |
| `_hasDash` | Refill on ground (Celeste-style) or refill on cooldown only |

## Wall slide + wall jump

```csharp
[Header("Wall")]
[SerializeField] private Transform wallCheck;
[SerializeField] private Vector2 wallCheckSize = new(0.1f, 0.6f);
[SerializeField] private LayerMask wallMask;
[SerializeField] private float wallSlideSpeed = 3f;
[SerializeField] private Vector2 wallJumpVelocity = new(8f, 12f);
[SerializeField] private float wallJumpInputLockTime = 0.15f;

private bool _wallContact;
private float _inputLockCounter;

void Update() {
    _wallContact = Physics2D.OverlapBox(wallCheck.position, wallCheckSize, 0f, wallMask) && !_grounded;

    // Wall slide — cap fall speed when in contact
    if (_wallContact && _rb.linearVelocity.y < 0f) {
        var v = _rb.linearVelocity;
        v.y = Mathf.Max(v.y, -wallSlideSpeed);
        _rb.linearVelocity = v;
    }

    if (_inputLockCounter > 0f) _inputLockCounter -= Time.deltaTime;
}

public void RequestJump() {
    if (_grounded || _coyoteCounter > 0f) {
        DoNormalJump();
    } else if (_wallContact) {
        DoWallJump();
    } else {
        _jumpBufferCounter = jumpBuffer;     // standard buffer
    }
}

private void DoWallJump() {
    int dir = transform.localScale.x > 0 ? -1 : 1;     // jump AWAY from facing
    _rb.linearVelocity = new Vector2(wallJumpVelocity.x * dir, wallJumpVelocity.y);
    _inputLockCounter = wallJumpInputLockTime;     // ignore directional input briefly so the player can't immediately re-attach
}
```

`_inputLockCounter` is critical — without it, a player holding "into the wall" snaps back instantly, which feels broken.

## Tuning

| Knob | Effect |
|---|---|
| `wallSlideSpeed` | Lower = sticky, higher = barely-affecting |
| `wallJumpVelocity.x` | Horizontal kick — too low = no detachment, too high = launches off |
| `wallJumpVelocity.y` | Vertical lift — usually similar to normal jump |
| `wallJumpInputLockTime` | 100..200ms — long enough to detach |

## Combine with dash

Dash + wall jump in the same controller produce metroidvania traversal — players love expressive movement. Order of priority on `RequestJump`:

```
1. Wall jump if wall-contacting (overrides ground)
2. Normal jump if grounded or coyote
3. Buffered jump otherwise
```

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Dash without input lock allowing re-cancel mid-dash | Set `_dashing` flag and ignore input writes during it |
| Wall jump that doesn't lock input | Player gets stuck; lock for 100..200ms |
| Refilling dash on wall touch (overpowered) | Refill on ground only, OR explicit power-up |
| Coroutine left running on disable | `StopCoroutine` in OnDisable; re-set gravity if dashing |
| Dash that doesn't cancel gravity | Gravity pulls dash trajectory downward — disable scale during dash |
