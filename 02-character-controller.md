# Character controller

## What it does

PEAK's scout walks, jumps, clings to terrain, and moves along ropes or vines. Other scouts can help pull them up or carry them when unconscious. Those actions share a physical body: losing control changes how strongly that body follows its animation, while the camera blends toward its physical pose. The distinction matters when reading the controller. Movement is not just a position update with an animation attached afterward.

## How it's built

`CharacterMovement.Update` samples local input through `CharacterInput`, processes looking and jumping, and updates movement state. Its `FixedUpdate` handles ground checks and distributes movement, gravity, drag, and animation forces across the ragdoll's `Bodypart` objects. `CharacterRagdoll.FixedUpdate` evaluates the animation and IK, records the resulting poses, and restores the body transforms. Animation supplies targets for physical movement.

Wall climbing belongs to `CharacterClimbing`; ropes and vines have `CharacterRopeHandling` and `CharacterVineClimbing`. All three write into `CharacterData`, the shared collection of movement flags, references, and timers. `Character` provides broader permission checks and coordinates components through its references.

Helping hands and carrying are different mechanisms. `CharacterGrabbing.Reach` finds eligible scouts and pulls them toward a target; the same component also contains a joint-based grab path used by the Scoutmaster. `CharacterCarrying` maintains the carrier relationship and changes collision behavior. `MainCameraMovement.CharacterCam` blends look direction with head rotation and moves its target toward the torso as ragdoll control falls. `HideTheBody` handles local-body visibility and separate shadow objects; animation, hands, and eyes have further presentation components.

## Design decisions

Separate climbing components make sense because a wall contact, a rope position, and a sliding vine attachment need different movement rules. The shared ragdoll also gives carrying and external forces the same body to act upon. The expensive choice is organizational: a movement mode is represented by several mutable fields, and entering or leaving it is spread across components. An explicit transition owner would still need these specialized solvers, but could make their shared state changes one responsibility. Here, each caller must know which parts of another mode need cancelling.

## Where it's weak

### Ground selection depends on contact order

`CharacterMovement.EvaluateGroundChecks` does not consistently select the highest accepted contact. After accepting a valid candidate, it replaces that candidate with later entries without comparing their height. The resulting choice therefore depends on the order of `groundSamples`. Its nearby fallback, `IsLodged`, does compare heights when choosing among already valid candidates. These are materially different selection rules, not two spellings of the same operation.

The callers make that difference relevant. `OnCollision` adds samples from body-part contacts, and `RaycastGroundCheck` can add a terrain sample before evaluation. The selected contact supplies `groundPos`, `groundNormal`, and the ground-relative `currentHeadHeight`. That last value feeds `Stand`, which applies corrective forces to the head, torso, and hip. Contact selection reaches the standing controller; it is not merely diagnostic information.

Changing contact order can change the ground reference even when the set of accepted contacts is unchanged. That is a bad dependency for a standing controller: the collection's ordering becomes part of the physics policy without an explicit selection rule. The better option was to select by a deliberate criterion, consistently, as the adjacent fallback already does with height. `EvaluateGroundChecks` does not do that.

### Carry cleanup repairs only part of the state

`CharacterCarrying.FixedUpdate` explicitly recognizes a carried scout whose carrier is gone. Its recovery method, `CarrierGone`, restores collision and the carried animation flag through `ToggleCarryPhysics`. It does not clear `CharacterData.isCarried`. Ordinary release through `RPCA_Drop` does clear that flag, clears the carrier reference, and updates carry weight. The exceptional release path leaves a different state behind.

This is more than an untidy pair of methods. The recovery condition can remain true on following physics ticks, so the same repair can run repeatedly. Elsewhere, `Character.BreakCharacterCarrying` treats `isCarried` as sufficient reason to call through the carrier reference. The state accepted by the recovery path is consequently a state another consumer does not safely handle. If that consumer runs while the inconsistency remains, it dereferences the missing carrier.

Anyone changing carry behavior must distinguish a normal drop from disappearance recovery, despite both being exits from the same relationship. A shared local cleanup operation could clear the relationship and restore physics together, with network notification handled separately. Instead, `CarrierGone` repairs presentation and collisions while leaving the relationship broken. Later callers inherit the contradiction.

### Climbing transitions are scattered

`CharacterData` stores wall, rope, and vine climbing as separate booleans, alongside a climb-handle reference and the object being climbed. `GrabRopeRpc` and `GrabVineRpc` each establish their mode and clear competing flags. `StartClimbRpc` establishes wall-climbing state after separate eligibility checks. Exit behavior lives in each component, while `CharacterClimbing.StopAnyClimbing` chooses a cancellation path by testing those states in order.

Consumers repeat the same knowledge. `Character.CheckStand`, `CheckGravity`, `CheckMovement`, `CheckJump`, and `CheckSprint` separately enumerate climbing conditions, as does `CharacterGrabbing.CanGrab`. Some checks deliberately differ: gravity and jumping also account for a climb handle, for example. That means replacing the repetitions with a single broad check would require understanding those distinctions first.

The cost is concrete duplication of transition and permission rules. A change to an attachment mode needs an audit of its entry method, its exit method, general cancellation, and every permission check that recognizes it. The declarations do not enforce mutual exclusion; the writers do. An explicit mode with centralized entry and exit rules would put that invariant in one place while retaining separate movement solvers. The current design makes adding another attachment mode a cross-component consistency exercise.

### The camera retains a distinction with no effect

`MainCameraMovement.GetHeadOffset` branches on wall-climbing state, but both branches apply the same interpolation toward the same offset. That condition cannot produce a different climbing offset. The cost is small but specific: the method advertises a distinction that its behavior does not contain, and someone tracing camera placement must verify both branches before learning that. One shared calculation would remove the false distinction. The ragdoll blend in `CharacterCam` changes position and orientation independently.

## For modders

Treat `CharacterData` as shared working state, not a collection of independent switches. The climbing writers and `Character` permission checks describe responsibilities that a movement change must preserve. Their names alone do not identify a safe place to intervene, and this chapter does not supply instructions for altering networked state.

Keep presentation separate from physical behavior when investigating a visual change. The animation pose feeds body-part targets, the camera blends between different references, and `HideTheBody` changes visibility while maintaining shadow objects. A missing body part in a view and a missing body part in the physical rig are different problems with different owners. Check the component and serialized references involved before attributing either to movement.

## Open questions

The ground-contact finding needs a controlled observation in the official Steam build to determine whether ordering changes are visible during ordinary movement. Carry disappearance needs its actual cleanup sequence traced through player removal and scene changes before the conditional state defect can be attached to a reported gameplay bug.

These findings do not establish a particular wobble, fall, or missed landing, simultaneous climbing modes during ordinary play, or incorrect camera placement. The ground-selection policy's original intent is also unknown. The critique concerns the selection behavior and state ownership visible in the source.

Serialized rig values and collision setup have not been audited here. Nor has this chapter measured responsiveness under different frame rates or network conditions. Those gaps prevent a judgment about how well the controller feels, even where its state ownership is difficult to maintain.
