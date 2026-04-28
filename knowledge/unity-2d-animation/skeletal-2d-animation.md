# Skeletal 2D Animation — Bones, Skinning, Weights

> Source: https://docs.unity3d.com/Packages/com.unity.2d.animation@latest/manual/SkinningEditor.html

Skeletal 2D animation = bones + skinned mesh + animation clips. Smaller asset count than frame-by-frame, easier to retarget across characters with the same rig.

## Required packages

- `com.unity.2d.animation`
- `com.unity.2d.psdimporter` (for PSB sources) or `com.unity.2d.aseprite` (for Aseprite sources)

## Workflow

1. Author character with each part on its own layer in Photoshop (PSB) or Aseprite.
2. Import — PSD/Aseprite Importer creates a multi-sprite asset; **Mosaic** + **Character Rig** ON.
3. Open the prefab — Sprite Editor — **Skinning Editor**.
4. **Create Bone** — root → spine → head, root → arm_L, root → leg_L (and mirrors).
5. **Auto Geometry** — generates a triangulated mesh per part.
6. **Auto Weights** — assigns vertex weights to bones; review with Weight Slider / Brush at joints.
7. **Apply** — Unity bakes weights into the asset.
8. Drag the prefab into a scene — each body part has a `SpriteSkin` component.
9. **Animation window** — record bone Transform changes; an Animator state machine drives clips.

## Weight painting tips

| Issue | Fix |
|---|---|
| Stretched limb at shoulder when arm rotates | Increase weight of upper-arm bone at shoulder vertices; decrease torso bone |
| Mesh tearing at hand/wrist | Add intermediate vertices in geometry; smooth weights between forearm and hand |
| Whole sprite moves when only head should | Re-paint head bone weights; ensure body bone weight is 0 on head vertices |

Use the Weight Slider tool with Bone A picked → drag to bias toward A.

## Inverse Kinematics (IK)

Add `IK Manager 2D` on the prefab root. Per limb:

```
GameObject named "ArmL_IK"
  Limb Solver 2D component
     Effector            child Transform you move externally
     Chain Size          2     (forearm + upper-arm)
     Constrain Rotation  ON
     Restore Default     ON
```

Drag a target Transform into Effector. The two-bone chain follows the target while keeping joint constraints.

## Sprite Skin component

Each rendered part has `SpriteSkin`. Visualize bones via its Inspector — gizmos show the bone hierarchy in the Scene view. `SpriteSkin.UpdateMethod = OnDemand` for off-screen culling perf.

## Animation clip layout

Animator state machine driven by parameters (`Speed`, `Grounded`, `Attack` triggers). Clips key bone Transform.position / rotation / scale — same as 3D. Cache parameter hashes:

```csharp
private static readonly int SpeedHash = Animator.StringToHash("Speed");
void Update() => _animator.SetFloat(SpeedHash, _input.magnitude);
```

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Re-importing a PSD that overwrites manual weights | Use Skinning Editor — save weights with the asset; PSD Importer preserves on re-import as long as part identifiers stay stable |
| One Animator per body part | Single Animator on root + Animation clips that key bone transforms |
| String parameter names every frame | Cache `Animator.StringToHash` ints |
| Heavy IK on every character every frame | Disable RigBuilder / IK Manager when off-screen |
