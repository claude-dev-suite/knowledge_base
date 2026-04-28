# Cinemachine 2D — Position Composer

> Source: https://docs.unity3d.com/Packages/com.unity.cinemachine@latest/manual/CinemachinePositionComposer.html

`Position Composer` (formerly `Framing Transposer`) is the standard 2D follow body in Cinemachine 3.

## Setup

```
Main Camera (Orthographic + CinemachineBrain)
PlayerFollow (CinemachineCamera)
   Lens         Orthographic + Size 5
   Follow       Player Transform
   Body         Position Composer
       Tracked Object Offset    (0, 1, 0)   slight upward look
       Damping                  X 0.3, Y 0.3
       Dead Zone Width / Height 0.1, 0.2
       Soft Zone Width / Height 0.5, 0.5
       Lookahead Time           0..0.3
       Lookahead Smoothing      ~5
   Aim          Do nothing      (2D doesn't aim — orthographic camera looks down -Z)
```

## Screen position breakdown

```
+---------------------------------------+
|                                       |
|       +-------------------+           |
|       |    Soft Zone      |           |
|       |  +-----------+    |           |
|       |  | Dead Zone |    |           |
|       |  |   target  |    |           |
|       |  +-----------+    |           |
|       +-------------------+           |
|                                       |
+---------------------------------------+
```

| Region | Behaviour |
|---|---|
| **Dead Zone** | Target inside this rectangle → camera doesn't move |
| **Soft Zone** | Target moving here → camera eases to recenter |
| **Outside Soft Zone** | Camera tracks immediately to keep target on screen |

## Tracked Object Offset

A tiny upward offset (Y = 0.5..1) helps platformers — player sees more sky than ground, which is what they're navigating into.

## Damping

Lower X damping for snappy left/right tracking; slightly higher Y damping to smooth out jumps. Typical platformer values: X = 0.2, Y = 0.5.

## Lookahead

`Lookahead Time` extrapolates target velocity → camera leads the player. Useful for fast horizontal movement (Sonic-like). Set Lookahead Time = 0.1..0.2 for a noticeable but not nauseating lead.

## Camera priorities

Multiple CmCameras coexist; the highest-priority active one drives the actual camera. Cinemachine blends between them when priorities change.

```csharp
public class CameraStateRouter : MonoBehaviour {
    [SerializeField] private CinemachineCamera gameplayCam;     // Priority 10
    [SerializeField] private CinemachineCamera dialogueCam;     // Priority 5
    public void StartDialogue() => dialogueCam.Priority.Value = 20;
    public void EndDialogue()   => dialogueCam.Priority.Value = 5;
}
```

CinemachineBrain blends with default time (~2s); per-camera blend overrides via Custom Blends asset.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Hard-coded `Camera.main.transform.position = ...` follow on Update | Cinemachine + Position Composer |
| Aim configured for 2D | Set Aim = Do nothing |
| Damping = 0 (camera glued to player) | Some damping (>= 0.1 X, 0.2 Y) gives natural feel |
| Dead Zone too tiny | Camera oscillates on small player movements; widen the dead zone |
