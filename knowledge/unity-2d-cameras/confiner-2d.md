# Cinemachine Confiner 2D

> Source: https://docs.unity3d.com/Packages/com.unity.cinemachine@latest/manual/CinemachineConfiner2D.html

`CinemachineConfiner2D` clamps the camera so it never reveals beyond a level boundary polygon — eliminates per-scene manual clamping logic.

## Setup

```
Level (GameObject)
  PolygonCollider2D    Is Trigger = ON  (purely a bounds reference, no physics)
                        Edit the polygon to outline the playable area

PlayerFollow (CinemachineCamera)
  +Add Extension > CinemachineConfiner2D
       Bounding Shape 2D        drag the PolygonCollider2D
       Damping                  0..1 (smooth-clamp at the boundary)
       Slowing Distance         0..2 (camera decelerates as it approaches the edge)
```

The confiner uses the polygon's outline; the camera viewport (orthographic frustum) is clamped so all four corners stay inside.

## Composite shapes — multi-room levels

For non-contiguous areas (separate rooms with thick walls between), use a `CompositeCollider2D` of multiple PolygonCollider2D children:

```
Level
   Room A (PolygonCollider2D, Used By Composite = ON)
   Room B (PolygonCollider2D, Used By Composite = ON)
   CompositeCollider2D    Is Trigger = ON, Geometry Type = Polygons
```

Camera is confined to whichever room contains the target.

## Switching bounds at runtime

For metroidvania / room-based games where each room has its own bounds:

```csharp
public class RoomBoundsTrigger : MonoBehaviour {
    [SerializeField] private CinemachineConfiner2D confiner;
    [SerializeField] private PolygonCollider2D roomBounds;

    private void OnTriggerEnter2D(Collider2D other) {
        if (other.CompareTag("Player")) {
            confiner.BoundingShape2D = roomBounds;
            confiner.InvalidateBoundingShapeCache();
        }
    }
}
```

`InvalidateBoundingShapeCache` is required — the confiner caches the polygon shape; without invalidation it'll keep using the previous bounds.

## Damping at the edge

`Damping > 0` makes the camera decelerate before hitting the boundary instead of stopping suddenly. Combined with `Slowing Distance`, gives a smooth rest at the edge — feels much better than an abrupt clamp.

## Edge cases

| Issue | Cause | Fix |
|---|---|---|
| Camera shows briefly outside bounds on level load | Cache not yet computed | Force-warp the camera once on Start: `cm.OnTargetObjectWarped(target, Vector3.zero);` |
| Confiner has no effect | Bounding Shape 2D not assigned, or is not a PolygonCollider2D / CompositeCollider2D | Assign the right type |
| Camera "snaps" when entering a new room | Damping = 0 | Set Damping > 0.1 |
| Room polygon smaller than camera viewport | Confiner can't fit camera inside | Increase room polygon, or use Pixel Perfect with smaller orthographic size |

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Manual `Mathf.Clamp` on camera position | Use Confiner 2D |
| Forgot `InvalidateBoundingShapeCache` after swap | Always call after swap |
| BoundingShape too small for the camera viewport | Camera can't fit; either enlarge or shrink ortho size |
| Box Collider 2D as bounds | Confiner needs Polygon (or Composite) — Box doesn't have the polygon API |
