# Sorting Layers, Order in Layer & Sorting Group

> Source: https://docs.unity3d.com/Manual/SortingGroup.html

Three orthogonal sorting mechanisms applied in this order:

1. **Sorting Layer** (broad strokes)
2. **Order in Layer** (fine integer within layer)
3. **Sorting Group** (wraps a parent + children to sort as one unit)

## Sorting Layers — define upfront

`Edit > Project Settings > Tags and Layers > Sorting Layers`. Order top-to-bottom = back-to-front:

```
Default
Background
World
Characters
FX
UI-World        (in-world UI like floating damage numbers)
```

Set every Sprite Renderer's `Sorting Layer` to one of these — never leave on `Default` for gameplay sprites.

## Order in Layer — fine control

Integer; higher = drawn later (front). Common: `int sortingOrder = -(int)(transform.position.y * 100)` in `LateUpdate` for top-down games (south characters draw above north characters).

```csharp
[RequireComponent(typeof(SpriteRenderer))]
public class YSort : MonoBehaviour {
    private SpriteRenderer _sr;
    void Awake() => _sr = GetComponent<SpriteRenderer>();
    void LateUpdate() => _sr.sortingOrder = -Mathf.RoundToInt(transform.position.y * 100f);
}
```

## Sorting Group — multi-part characters

A Knight made of head + body + sword + shadow. Without a Sorting Group, parts z-fight against world objects. Add `SortingGroup` on the parent:

```
Knight (GameObject + SortingGroup)  Sorting Layer: Characters, Order: 0
  Body (SpriteRenderer)
  Head (SpriteRenderer)
  Sword (SpriteRenderer + Order: 1 to draw above body)
  Shadow (SpriteRenderer + Order: -1)
```

Children sort against each other inside the group; the group itself sorts against the world via its own Sorting Layer + Order.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| All gameplay on `Default` sorting layer | Define and use named layers |
| Multi-part character without Sorting Group | Add SortingGroup on root |
| Mixing Y-sort and fixed Order on same layer | Pick one strategy per layer |
| Sorting Group nested inside another with conflicts | Flatten or document the priority chain |
