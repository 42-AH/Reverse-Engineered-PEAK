# Weather and environmental hazards

PEAK's environment reaches the player through several independent mechanisms. Some hazards add afflictions, some alter movement, and others apply forces to the character's ragdoll. Lighting and visual effects sometimes supply inputs to those gameplay decisions. This chapter follows those connections, with particular attention to wind exposure, water-zone tracking, and the use of rendered sunlight as a heat gate.

There is no single environmental-damage pipeline. That separation gives each hazard room for specialized behavior, but also leaves shared questions—who is exposed, when an effect ends, and which characters remain valid—to individual components.

## What it does

The day/night cycle updates lighting and shader parameters. Wind has active and inactive periods, exposure-dependent status application, and physical effects. Fog advances through the level while applying a status consequence below its current height. Sun exposure depends on location and shelter. Water changes movement and produces splash effects, while waterfalls apply a separate knockback response.

Lava combines proximity heat, contact injury, physical movement, and item cooking. Rising fields add progression-controlled movement of the hazard itself. Tornadoes move through the environment and affect nearby characters. Unstable rocks and ice have their own transitions from fixed geometry to falling or fractured objects. These are separate behaviors whose presence and configuration belong to particular scene content.

## How it's built

`DayNightManager.Update` advances time and calls `UpdateCycle`. The cycle blends profile values for the sun, sky, fog, and global shader parameters. Storm-related factors also affect light intensity. The master periodically synchronizes time, while each client updates its presentation from the available state.

Sun exposure connects that presentation to gameplay:

- `DayNightManager.UpdateCycle` writes the sun light's intensity and orientation.
- `HotSun.Update` checks its bounds and the current sun intensity.
- It performs a line check along the sun direction to determine whether the local character is sheltered.
- Eligible exposure accumulates time and reaches `CharacterAfflictions.AddSunHeat`.

Wind separates scheduling from application. `WindChillZone.HandleTime` changes the active period through the master role. `Update` records whether the local and observed characters are inside its bounds. For an exposed local character, `ApplyStatus` samples `LightVolume` through `GetWindIntensityAtPoint`, adjusts climbing's minimum stamina multiplier, and applies the configured status or slipperiness contribution.

`WindChillZone.FixedUpdate` handles force. It calls `AddWindForceToCharacter` for the local character and for eligible bot characters, with adjustments for equipment and movement state. Item force has a separate loop through active ground items and a separate `AddWindForceToItem` method. Exposure, character force, and item force are therefore three related but independently implemented paths.

`Fog.Update` runs movement, status application, and visual updates. `MakePlayerCold` uses the character's height relative to the fog and distinguishes a skeleton's injury response from the ordinary cold response. `ApplyVisuals` updates the fog transform and global fog height. The owner also publishes periodic height updates. Particle playback and the character's fog flag are handled alongside those steps.

Water uses a tracked population. `WaterZone.OnEnable` starts `UpdateTrackedCharactersRoutine` and subscribes to jump and landing events. The routine discovers player and bot characters, then classifies them using the character center and feet against the zone bounds. `UpdateCharacters` sends movement and jump modifiers to `CharacterMovement.SetWaterMovementModifier`; `UpdateCharacterVisuals` writes animation-related water state. `PlaySplash` handles the event-driven presentation. `OnDisable` stops the routine and removes the event subscriptions.

The movement consumer combines water movement and jump modifiers with existing values rather than blindly replacing them. `CharacterMovement.ResetFixedVars` resets those working modifiers. This matters when assessing multiple-zone behavior: the downstream accumulation rule must be read before assuming that the last water zone simply overwrites every movement effect.

`WaterfallPusher.OnTriggerEnter` uses a different contact path. It identifies a local character through the ragdoll collider, applies a cooldown, and combines falling, camera feedback, and force. Ordinary water movement restrictions and waterfall knockback do not share the same trigger or lifetime.

`Lava.Update` calls `Heat` and `TryCookItems`; its fixed update calls `DoEffects`. Heat uses horizontal bounds and distance from the surface, while the contact path adds force and affliction effects. Item cooking is handled on the master and passes through `ItemCooking`, connecting this hazard to the item lifecycle discussed in chapter 04. `LavaRising` separately tracks whether a rising field has started or ended and how far its progression has advanced.

Other hazards retain their own physical controllers. `Tornado.Update` handles its moving lifecycle and owner-side target selection; its character-force paths combine attraction, lift, and drag. `ShakyRock.FixedUpdate` checks whether a falling object has cleared neighboring geometry before restoring its collision layers. `ShakyIcicleIce.FixedUpdate` similarly manages clearance and a transition to fracture objects. Those clearance tests explain why a falling hazard is more than an animation.

`MirageManager.sampleCamera` prepares an auxiliary camera from the main camera before rendering it. That is presentation machinery, distinct from the contact and exposure mechanisms above. A visual illusion's rendered image alone does not establish which collision or interaction rules its associated objects use.

## Design decisions

The affliction system is the shared destination for several hazards, but their exposure models differ. Fog uses a height boundary. Sunlight uses a directional obstruction test. Wind samples a light volume and can additionally use a raycast when applying force. Those choices encode different approximations of shelter.

Physical effects also remain separate from status accumulation. Waterfall knockback, wind acceleration, and lava contact can change movement before a status total becomes large. A test that checks only affliction values would miss those consequences.

Local ownership checks and master-controlled scheduling divide work across participants. A shared storm period does not mean that one machine applies every character's force. The character-force consumer checks ownership, while individual hazards decide how to discover and sample their targets.

## Where it's weak

### Wind force does not use one exposure contract

`WindChillZone.ApplyStatus` computes `windPlayerFactor` from the local character's position. `AddWindForceToCharacter` accepts a character argument but multiplies its final force by that shared field. The caller also uses this method for bot characters inside the zone.

The factor belongs to one sampled position, but the force consumer treats it as applicable to every passed character. A locally owned bot at another location receives force scaled by the local player's exposure rather than its own light-volume sample. The method's ownership guard narrows which characters receive force; it does not correct the source of the exposure factor. Equipment and raycast adjustments are evaluated for the target character, making the mixed sampling scopes especially easy to overlook.

The consequence is a positional dependency between otherwise separate characters. Changing the local player's shelter changes the shared factor used for the bot's force. A target-parameter method should not silently import another target's exposure.

The item path has a different mismatch. `FixedUpdate` reaches its item loop whenever item movement is enabled; that loop is outside the active-wind and delayed-force conditions used for the local character. `AddWindForceToItem` does not multiply by `currentForceMult` or check `windActive`. With the intensity-curve option disabled, it uses a constant strength factor. An inactive wind period therefore does not, by this path, stop item acceleration. With the curve option enabled, the result depends on the retained intensity value rather than a shared active-period rule.

These are separate consequences of the same design problem: scheduling, exposure, and force strength are recombined differently for each recipient. The correction is to calculate a target-specific exposure and a shared storm-strength value, then pass those explicit inputs into both character and item force application. That makes the active-period rule consistent without requiring identical physical responses for every target.

### Water tracking skips destroyed characters without removing them

`WaterZone.UpdateTrackedCharactersRoutine` adds discovered characters to `trackedCharacters`. When it encounters a character that now compares null, it yields and continues. It does not remove the dictionary entry. The ordinary update consumers then enumerate the same dictionary independently.

`UpdateCharacterVisuals` dereferences the key when the retained state is shallow or deep. `UpdateCharacters` dereferences it when sending movement modifiers, regardless of whether the recorded water state is none. Neither consumer performs the routine's null check. A dead reference skipped by the classifier is still handed to the effect application path.

This is a lifetime failure, not a disagreement about water depth. The classifier recognizes that the character no longer exists, but its response leaves the invalid entry available to other methods. The next consumer can fail while accessing the destroyed character instead of applying modifiers to the remaining live population.

The code also keeps the dictionary across disable and re-enable; `OnDisable` stops the coroutine and removes event subscriptions but does not clear tracked membership. That teardown is correct for the subscriptions and incomplete for the tracked-object lifetime. The dictionary's current contents cannot safely be treated as a live-character set merely because discovery runs repeatedly.

Removal should be part of reconciliation: gather dead entries, remove them outside enumeration, and guard consumers against objects disappearing between scans. The zone should also define whether disabling clears membership or merely suspends it. Either choice needs a validity check when work resumes. Periodic discovery alone does not maintain a valid registry.

### Sun heat is gated by a rendering output

`HotSun.Update` uses `DayNightManager.instance.sun.intensity` to decide whether exposure is eligible. That intensity is not a dedicated heat variable. `DayNightManager.UpdateCycle` calculates it from blended profile intensity, the sun gradient, and storm-related reductions before assigning it to the light.

The heat consumer therefore depends on an artistic output as well as its own volume and shelter test. Adjusting a profile's light intensity to change the appearance of a biome can move it across the heat gate without changing the `HotSun` configuration. A transition between lighting profiles can also change eligibility because the blended result is what the hazard reads.

That coupling makes lighting review part of gameplay-balance review. A brighter light is not merely a brighter image in this path, and the source offers no separate heat-eligibility contract explaining that dependency to a profile author. The shelter raycast establishes obstruction, but it cannot distinguish an intentional heat change from a visual intensity adjustment.

A shared environmental model should expose a sunlight or heat contribution intended for gameplay, with its own profile inputs. Lighting can derive its visual intensity from the same time and weather state without making the light component the authority for damage eligibility. If the coupling is intentional, validation should at least report which profile transitions cross the heat threshold so a visual change cannot silently alter the hazard.

## For modders

Trace exposure and application separately. A wind status value, a wind force, and a wind-driven item are different consumers even when they share one zone. For water, inspect `CharacterMovement`'s accumulation and reset behavior before attributing overlap effects solely to zone update order.

Visual changes also need gameplay review where render state feeds a hazard. Sun intensity is one such boundary; wind's light-volume sampling is another. Chapter 03 covers the status accumulator after a hazard submits its contribution. Correct exposure does not by itself establish correct accumulation, and the reverse is also true.

## Open questions

The wind and water failure mechanisms were traced in source, without a corresponding official-session reproduction. Which shipped zones enable item wind, which use the curve option, and where bot characters participate need scene-reference evidence. The water failure requires a tracked character to be destroyed while the zone remains a consumer; its frequency in ordinary transitions is unverified.

The relationship between light intensity and heat can be deliberate balancing. The unresolved question is whether profile authors intend every crossing of the rendering threshold to change heat eligibility. The source establishes the dependency, not the artistic intent.

`LavaTides` contains empty lifecycle methods in the inspected source. That does not prove the game lacks a tide-like hazard: other moving or rising lava components exist. The active scene components must be identified before attributing any visible behavior to that class name. Hazard placement, including roots-related wind and particular lava or mirage variants, likewise needs scene evidence beyond the presence of a component type.


