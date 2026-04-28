# Contact Filters & Allocation-Free Queries

> Source: https://docs.unity3d.com/ScriptReference/ContactFilter2D.html

Per-frame physics queries (`Physics2D.OverlapCircle`, `Raycast`, etc.) without care produce GC garbage. Use ContactFilter2D + `NonAlloc` overloads for hot paths.

## ContactFilter2D

```csharp
private static readonly Collider2D[] HitsBuffer = new Collider2D[16];
private ContactFilter2D _enemyFilter;

void Awake() {
    _enemyFilter = new ContactFilter2D {
        useLayerMask = true,
        layerMask = LayerMask.GetMask("Enemy"),
        useTriggers = false,    // skip trigger colliders
    };
}

void Hit() {
    int n = Physics2D.OverlapCircle(transform.position, 0.5f, _enemyFilter, HitsBuffer);
    for (int i = 0; i < n; i++) {
        HitsBuffer[i].GetComponent<IDamageable>()?.Damage(10);
    }
}
```

`HitsBuffer` is a static reusable array — zero allocations per call.

## Allocation-free APIs

All take a pre-allocated array + return `int` for hit count:

```csharp
Physics2D.OverlapCircle(point, radius, filter, results);
Physics2D.OverlapBox(point, size, angle, filter, results);
Physics2D.Raycast(origin, direction, filter, results, distance);
Physics2D.LinecastNonAlloc(start, end, results, layerMask);
Physics2D.GetContacts(collider, filter, results);
rb.Cast(direction, filter, results, distance);
```

Old single-result APIs (`Physics2D.OverlapCircle(...)` returning a single `Collider2D`) DO allocate the result list internally — avoid in tight loops.

## Useful filter fields

| Field | Effect |
|---|---|
| `useLayerMask` + `layerMask` | Restrict to a layer set |
| `useTriggers` | Include/exclude trigger colliders |
| `useDepth` + `minDepth` / `maxDepth` | Z-position filtering (rare in pure 2D) |
| `useNormalAngle` + `minNormalAngle` / `maxNormalAngle` | Filter by surface normal angle (e.g. only ground-facing surfaces) |
| `useOutsideNormalAngle` | Inverts normal-angle filter |

## Cast variants on Rigidbody2D

`Rigidbody2D.Cast` sweeps the entire rigidbody collider set forward — useful for "would I bump into something?" before applying movement:

```csharp
private static readonly RaycastHit2D[] CastHits = new RaycastHit2D[4];

void TryMove(Vector2 displacement) {
    int n = _rb.Cast(displacement.normalized, _wallFilter, CastHits, displacement.magnitude);
    if (n == 0) {
        _rb.MovePosition(_rb.position + displacement);
    } else {
        // hit something — clamp displacement to the hit distance
        _rb.MovePosition(_rb.position + displacement.normalized * CastHits[0].distance);
    }
}
```

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `Physics2D.OverlapCircleAll` per frame | NonAlloc + reused buffer |
| New `ContactFilter2D` per call | Cache in Awake |
| Buffer too small (silent truncation) | Make it generously sized; assert `n < buffer.Length` |
| `LayerMask.GetMask` per frame | Cache once in Awake |
