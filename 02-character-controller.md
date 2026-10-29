# Character controller

PEAK's scout walks, jumps, clings to terrain, and moves along ropes or vines. Other scouts can help pull them up or carry them when unconscious. Those actions share a physical body: losing control changes how strongly that body follows its animation, while the camera blends toward its physical pose. The distinction matters when reading the controller. Movement is not just a position update with an animation attached afterward.

## Ground contacts and the physical rig

`CharacterMovement.Update` samples local input through `CharacterInput`, processes looking and jumping, and updates movement state. Its `FixedUpdate` handles ground checks and distributes movement, gravity, drag, and animation forces across the ragdoll's `Bodypart` objects. `CharacterRagdoll.FixedUpdate` evaluates the animation and IK, records the resulting poses, and restores the body transforms. Animation supplies targets for physical movement.

A standing controller ought to choose its ground deliberately. Here, the order in which the contacts arrive gets a vote.

`CharacterMovement.EvaluateGroundChecks` does not consistently select the highest accepted contact. After accepting a valid candidate, it replaces that candidate with later entries without comparing their height. The resulting choice therefore depends on the order of `groundSamples`. Its nearby fallback, `IsLodged`, does compare heights when choosing among already valid candidates. These are materially different selection rules, not two spellings of the same operation.

The callers make that difference relevant. `OnCollision` adds samples from body-part contacts, and `RaycastGroundCheck` can add a terrain sample before evaluation. The selected contact supplies `groundPos`, `groundNormal`, and the ground-relative `currentHeadHeight`. That last value feeds `Stand`, which applies corrective forces to the head, torso, and hip. Contact selection reaches the standing controller; it is not merely diagnostic information.

`EvaluateGroundChecks` has a bad selection rule: later contact wins, whether or not it is the better ground reference. Reordering the same accepted contacts can therefore change the input to `Stand`. That is an unnecessary dependency on collection order inside a physics controller. The adjacent `IsLodged` code already demonstrates a deliberate height comparison; this path does not bother to maintain the same standard. Pick a criterion and apply it. Leaving contact order to decide is not a defensible substitute.

## Climbing and attachment state

Wall climbing belongs to `CharacterClimbing`; ropes and vines have `CharacterRopeHandling` and `CharacterVineClimbing`. All three write into `CharacterData`, the shared collection of movement flags, references, and timers. `Character` provides broader permission checks and coordinates components through its references.

Separate climbing components make sense because a wall contact, a rope position, and a sliding vine attachment need different movement rules. The shared ragdoll also gives carrying and external forces the same body to act upon. The expensive choice is organizational: a movement mode is represented by several mutable fields, and entering or leaving it is spread across components. An explicit transition owner would still need these specialized solvers, but could make their shared state changes one responsibility. Here, each caller must know which parts of another mode need cancelling.

The flags have specialized names. Unfortunately, that does not give them a single owner.

`CharacterData` stores wall, rope, and vine climbing as separate booleans, alongside a climb-handle reference and the object being climbed. `GrabRopeRpc` and `GrabVineRpc` each establish their mode and clear competing flags. `StartClimbRpc` establishes wall-climbing state after separate eligibility checks. Exit behavior lives in each component, while `CharacterClimbing.StopAnyClimbing` chooses a cancellation path by testing those states in order.

Consumers repeat the same knowledge. `Character.CheckStand`, `CheckGravity`, `CheckMovement`, `CheckJump`, and `CheckSprint` separately enumerate climbing conditions, as does `CharacterGrabbing.CanGrab`. Some checks deliberately differ: gravity and jumping also account for a climb handle, for example. That means replacing the repetitions with a single broad check would require understanding those distinctions first.

This is poor state ownership, and adding another climbing mode will make it worse. Its entry, exit, cancellation, and permission rules must all be updated across components that have no enforced agreement. `CharacterData` permits combinations that its writers are expected to avoid by convention. A central transition owner and an explicit attachment mode would enforce that agreement while leaving the specialized solvers alone. Instead, every change comes with a scavenger hunt for the other booleans that need clearing.

## Carrying, visibility, and camera

Helping hands and carrying are different mechanisms. `CharacterGrabbing.Reach` finds eligible scouts and pulls them toward a target; the same component also contains a joint-based grab path used by the Scoutmaster. `CharacterCarrying` maintains the carrier relationship and changes collision behavior. `MainCameraMovement.CharacterCam` blends look direction with head rotation and moves its target toward the torso as ragdoll control falls. `HideTheBody` handles local-body visibility and separate shadow objects; animation, hands, and eyes have further presentation components.

`CharacterCarrying.FixedUpdate` explicitly recognizes a carried scout whose carrier is gone. Its recovery method, `CarrierGone`, restores collision and the carried animation flag through `ToggleCarryPhysics`. It does not clear `CharacterData.isCarried`. Ordinary release through `RPCA_Drop` does clear that flag, clears the carrier reference, and updates carry weight. The exceptional release path leaves a different state behind.

The recovery condition can remain true on following physics ticks, so the same repair can run repeatedly. Elsewhere, `Character.BreakCharacterCarrying` treats `isCarried` as sufficient reason to call through the carrier reference. The state accepted by the recovery path is consequently a state another consumer does not safely handle. If that consumer runs while the inconsistency remains, it dereferences the missing carrier.

`CarrierGone` is a broken cleanup path. It restores enough presentation and collision state to look like recovery while retaining a relationship that no longer exists. `RPCA_Drop` already shows that leaving a carry relationship requires more work; the exceptional path simply does less of it. One shared local cleanup operation should clear the relationship and restore physics, with network notification handled separately. Duplicating that job has produced contradictory state and left later callers to trip over it.

`MainCameraMovement.GetHeadOffset` branches on wall-climbing state, but both branches apply the same interpolation toward the same offset. That condition cannot produce a different climbing offset. The cost is small but specific: the method advertises a distinction that its behavior does not contain, and someone tracing camera placement must verify both branches before learning that. One shared calculation would remove the false distinction. The ragdoll blend in `CharacterCam` changes position and orientation independently.

## Reading and changing this code

Treat `CharacterData` as shared working state, not a collection of independent switches. The climbing writers and `Character` permission checks describe responsibilities that a movement change must preserve. Their names alone do not identify a safe place to intervene, and this chapter does not supply instructions for altering networked state.

Keep presentation separate from physical behavior when investigating a visual change. The animation pose feeds body-part targets, the camera blends between different references, and `HideTheBody` changes visibility while maintaining shadow objects. A missing body part in a view and a missing body part in the physical rig are different problems with different owners. Check the component and serialized references involved before attributing either to movement.

## Open questions

The ground-contact finding needs a controlled observation in the official Steam build to determine whether ordering changes are visible during ordinary movement. Carry disappearance needs its actual cleanup sequence traced through player removal and scene changes before the conditional state defect can be attached to a reported gameplay bug.

These findings do not establish a particular wobble, fall, or missed landing, simultaneous climbing modes during ordinary play, or incorrect camera placement. The ground-selection policy's original intent is also unknown. The critique concerns the selection behavior and state ownership visible in the source.

Serialized rig values and collision setup have not been audited here. Nor has this chapter measured responsiveness under different frame rates or network conditions. Those gaps prevent a judgment about how well the controller feels, even where its state ownership is difficult to maintain.
