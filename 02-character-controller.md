# Character controller

PEAK's scout walks, jumps, clings to terrain, and moves along ropes or vines. When needed, other scouts can help pull them up or carry them if they become unconscious. All these actions share a single physical body. Losing control changes how closely that body follows its animation, and the camera blends toward its physical pose. This matters for understanding the controller because movement involves more than just updating position and playing an animation.

## Ground contacts and the physical rig

`CharacterMovement.Update` samples local input through `CharacterInput`, handles looking and jumping, and updates movement state. The `FixedUpdate` method checks for ground contact and applies movement, gravity, drag, and animation forces to the ragdoll's `Bodypart` objects. `CharacterRagdoll.FixedUpdate` then evaluates animation and IK, stores the resulting poses, and restores body transforms. Animation provides targets that drive physical movement.

A standing controller should select its ground deliberately. In this implementation, the order in which contacts arrive influences the decision.

`CharacterMovement.EvaluateGroundChecks` does not reliably pick the highest accepted contact. After it accepts a valid candidate, later entries replace that candidate without any height comparison. The final choice therefore depends on the order of `groundSamples`. By contrast, the nearby fallback `IsLodged` does compare heights when selecting among valid candidates. These represent fundamentally different selection approaches, not just two ways of doing the same thing.

The calling code makes this difference matter. `OnCollision` adds samples from body-part contacts, while `RaycastGroundCheck` can add a terrain sample before evaluation. The selected contact provides `groundPos`, `groundNormal`, and the ground-relative `currentHeadHeight`. This last value goes to `Stand`, which applies corrective forces to the head, torso, and hip. Contact selection directly feeds the standing controller; it is not just a diagnostic detail.

The problem is that `EvaluateGroundChecks` uses a poor selection rule: whichever contact comes last wins, regardless of whether it offers better ground reference. Rearranging the same accepted contacts can therefore alter what `Stand` receives as input. Depending on collection order inside a physics controller is not a good design choice. The nearby `IsLodged` code already shows what deliberate height comparison looks like; this path simply does not maintain the same standard. Use a consistent criterion and apply it everywhere. Letting contact order decide is not acceptable.

## Climbing and attachment state

Wall climbing lives in `CharacterClimbing`; ropes and vines use `CharacterRopeHandling` and `CharacterVineClimbing`. All three write to `CharacterData`, which holds shared movement flags, references, and timers. `Character` provides broader permission checks and coordinates these components through its references.

Separate climbing components make sense because walls, ropes, and sliding vines each demand different movement rules. The shared ragdoll also lets carrying and external forces use the same body. The real organizational cost appears elsewhere: a movement mode consists of several mutable fields scattered across components, and entering or leaving it happens in pieces. An explicit transition owner would still need these specialized solvers, but could consolidate the shared state changes into one place. As it stands, each component must know which other mode states need clearing.

The flags have specific names, but that does not give them a single owner.

`CharacterData` tracks wall, rope, and vine climbing as separate booleans, plus a climb-handle reference and the object being climbed. `GrabRopeRpc` and `GrabVineRpc` each set their mode and clear competing flags. `StartClimbRpc` sets up wall-climbing state after its own eligibility checks. Exit logic sits in each component, while `CharacterClimbing.StopAnyClimbing` picks a cancellation approach by testing those states in sequence.

Code that uses these states duplicates the same knowledge. `Character.CheckStand`, `CheckGravity`, `CheckMovement`, `CheckJump`, and `CheckSprint` each list climbing conditions separately, as does `CharacterGrabbing.CanGrab`. Some checks intentionally differ: gravity and jumping also consider a climb handle, for instance. Consolidating these into a single check would first require understanding those differences.

This represents poor state ownership, and adding another climbing mode will only make things worse. Its entry, exit, cancellation, and permission rules need updates across multiple components with no enforced consistency. `CharacterData` allows state combinations that its writers must avoid through unwritten convention. A central transition owner and an explicit attachment mode could enforce this agreement while keeping the specialized solvers unchanged. Instead, every modification comes with a search through other components for booleans that need clearing.

## Carrying, visibility, and camera

Helping hands and carrying are separate systems. `CharacterGrabbing.Reach` finds eligible scouts and pulls them toward a target; the same component also holds a joint-based grab used by the Scoutmaster. `CharacterCarrying` manages the carrier relationship and modifies collision behavior. `MainCameraMovement.CharacterCam` blends look direction with head rotation and shifts its target toward the torso as ragdoll control decreases. `HideTheBody` controls local-body visibility and handles separate shadow objects; animation, hands, and eyes have their own presentation components.

`CharacterCarrying.FixedUpdate` detects when a carried scout's carrier has disappeared. The recovery method, `CarrierGone`, restores collision and the carried animation flag using `ToggleCarryPhysics`. However, it does not clear `CharacterData.isCarried`. Normal release via `RPCA_Drop` clears that flag, clears the carrier reference, and updates carry weight. The exceptional path leaves a different state behind.

The recovery condition can remain true on following physics ticks, so the same repair can run repeatedly. Elsewhere, `Character.BreakCharacterCarrying` treats `isCarried` as sufficient to access the carrier reference. The state that the recovery path accepts is therefore a state that another piece of code cannot safely handle. If that code runs while the inconsistency exists, it tries to dereference a missing carrier.

`CarrierGone` is a flawed cleanup method. It restores enough display and collision state to look like recovery, but keeps a relationship that no longer exists. `RPCA_Drop` already demonstrates what ending a carry relationship really requires; the exceptional path simply does less. A single shared local cleanup should break the relationship and restore physics, with network notification handled separately. Having two versions has created conflicting state and set up later code to fail.

`MainCameraMovement.GetHeadOffset` has a branch for wall-climbing state, but both branches use the same interpolation toward the same offset. This condition cannot produce different climb offsets. The overhead is minor but telling: the method suggests a distinction that it does not actually implement, so anyone reading the camera code must check both branches to realize they are identical. A unified calculation would eliminate the false distinction. Meanwhile, the ragdoll blend in `CharacterCam` already handles position and orientation separately.

## Reading and changing this code

Treat `CharacterData` as shared working state rather than a collection of independent flags. The climbing writers and `Character` permission checks represent responsibilities that any movement change must honor. Their names alone do not show where it is safe to intervene, and this section does not explain how to alter networked state.

Keep presentation separate from physical behavior when investigating a visual issue. Animation poses drive body-part targets, the camera blends between different references, and `HideTheBody` changes visibility while keeping shadow objects intact. A missing body part on screen and a missing body part in the physical rig are separate problems with separate owners. Check the relevant components and their serialized references before assuming either belongs to movement.

## Open questions

The ground-contact finding needs a controlled observation in the official Steam build to determine whether ordering changes are visible during ordinary movement. Carry disappearance needs its actual cleanup sequence traced through player removal and scene changes before the conditional state defect can be attached to a reported gameplay bug.

These findings do not establish a particular wobble, fall, or missed landing, simultaneous climbing modes during ordinary play, or incorrect camera placement. The ground-selection policy's original intent is also unknown. The critique concerns the selection behavior and state ownership visible in the source.

Serialized rig values and collision setup have not been audited here. Nor has this chapter measured responsiveness under different frame rates or network conditions. Those gaps prevent a judgment about how well the controller feels, even where its state ownership is difficult to maintain.
