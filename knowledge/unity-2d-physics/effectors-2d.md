# Effectors 2D

> Source: https://docs.unity3d.com/Manual/class-PlatformEffector2D.html

Effectors are auto-applied behaviours on a collider. Tag the collider as `Used By Effector` and add the effector component on the same GameObject.

## PlatformEffector2D — one-way platforms

```
BoxCollider2D (Used By Effector = ON)
PlatformEffector2D
   Surface Arc        140  (degrees from up vector that count as the platform top)
   One Way            ON
   Use Side Friction  ON / OFF as desired
   Use Side Bounce    OFF
```

Player jumps from below — passes through. Lands on top — collides. Drop-through:

```csharp
// Disable the player's collider for ~0.2s when pressing Down + Jump
StartCoroutine(DropThrough());
IEnumerator DropThrough() {
    Physics2D.IgnoreCollision(_playerCol, _platformCol, true);
    yield return new WaitForSeconds(0.25f);
    Physics2D.IgnoreCollision(_playerCol, _platformCol, false);
}
```

## SurfaceEffector2D — conveyor belts

```
EdgeCollider2D (Used By Effector = ON)
SurfaceEffector2D
   Speed              5    (m/s along the surface)
   Force Scale        0.5  (how strongly the surface pushes the contacting body)
```

## AreaEffector2D — wind, current

Inside the trigger collider, applies a constant force.

```
BoxCollider2D (Is Trigger = ON, Used By Effector = ON)
AreaEffector2D
   Force Angle        90    (degrees, 0 = right)
   Force Magnitude    8
   Drag               1     (also reduces velocity inside the area)
```

## PointEffector2D — attract / repel

```
CircleCollider2D (Is Trigger = ON, Used By Effector = ON)
PointEffector2D
   Force Magnitude    -10    (negative = attract, positive = repel)
   Distance Scale     1      (1/r vs 1/r² falloff via Force Mode)
```

## BuoyancyEffector2D — water

```
EdgeCollider2D at the water surface (Is Trigger = ON, Used By Effector = ON)
BuoyancyEffector2D
   Density            2
   Linear Drag        2
   Angular Drag       1
   Surface Level      <Y position of the water surface>
```

Objects falling into the water decelerate and float according to density vs Rigidbody2D mass.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Effector without `Used By Effector` flag on collider | Tick it; otherwise effector does nothing |
| One-way platform that gets stuck when player jumps from inside | Tune Surface Arc; or add drop-through coroutine |
| Conveyor belt using AreaEffector | Use SurfaceEffector — applies force only along the edge contact |
| Buoyancy water as solid collider | Set Is Trigger = ON; otherwise rigidbodies bounce off the surface |
