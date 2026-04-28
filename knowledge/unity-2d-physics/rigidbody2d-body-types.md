# Rigidbody2D Body Types

> Source: https://docs.unity3d.com/Manual/class-Rigidbody2D.html

Rigidbody2D has three body types — choose deliberately.

| Body Type | Behaviour | Cost |
|---|---|---|
| **Dynamic** | Moves under force / gravity / velocity. Receives full physics simulation. | Highest |
| **Kinematic** | Moves only via `MovePosition` / `MoveRotation`. No automatic forces, no gravity. | Medium — still triggers collisions but no resolution |
| **Static** | Doesn't move. Not allowed to translate at runtime. Best perf. | Lowest |

## Dynamic — most gameplay objects

```csharp
[RequireComponent(typeof(Rigidbody2D))]
public class Push : MonoBehaviour {
    private Rigidbody2D _rb;
    void Awake() {
        _rb = GetComponent<Rigidbody2D>();
        _rb.interpolation = RigidbodyInterpolation2D.Interpolate;
        _rb.collisionDetectionMode = CollisionDetectionMode2D.Continuous;
    }
    public void Push2D(Vector2 dir) => _rb.AddForce(dir * 10f, ForceMode2D.Impulse);
}
```

Use Continuous CCD on fast movers (bullets, dash) to avoid tunneling.

## Kinematic — character controllers

For tight platformer feel, control motion manually:

```csharp
private void FixedUpdate() {
    Vector2 desired = _rb.position + _input * speed * Time.fixedDeltaTime;
    _rb.MovePosition(desired);
}
```

Kinematic still triggers collisions — `OnCollisionEnter2D` / `OnTriggerEnter2D` fire. Use `Rigidbody2D.Cast` to probe before moving for solid-vs-solid handling.

## Static — terrain & tilemaps

Set when geometry never moves at runtime. Critical for Composite Collider on tilemaps.

```
Static Rigidbody2D + TilemapCollider2D (Used By Composite) + CompositeCollider2D
```

If you need a moving Static (e.g. doors), it must be Kinematic, not Static. Static literally cannot translate.

## Switching at runtime

Setting `bodyType` works but invalidates physics state — use sparingly. Common case: enable physics on dropped item (Static -> Dynamic) when picked up.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Dynamic body for player when you want manual control | Kinematic + MovePosition |
| Static for moving doors / platforms | Kinematic |
| `transform.position` writes on a Rigidbody2D | Use MovePosition / set linearVelocity |
| Discrete CCD on fast bullets | Continuous |
| `Rigidbody2D.velocity` (deprecated in Unity 6) | `Rigidbody2D.linearVelocity` |
