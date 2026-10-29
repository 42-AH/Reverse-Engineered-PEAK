# Weather and environmental hazards

Weather is not one system with a few different effects attached. Fog adds conditions, water changes movement, wind pushes bodies, and sunlight borrows a rendering value to decide whether the scout gets hot. Follow those decisions and the boundaries between presentation and gameplay get rather negotiable.

There is no single environmental-damage pipeline. That separation gives each hazard room for specialized behavior, but also leaves shared questions—who is exposed, when an effect ends, and which characters remain valid—to individual components.

The day/night cycle updates lighting and shader parameters. Wind has active and inactive periods, exposure-dependent status application, and physical effects. Fog advances through the level while applying a status consequence below its current height. Sun exposure depends on location and shelter. Water changes movement and produces splash effects, while waterfalls apply a separate knockback response.

Lava combines proximity heat, contact injury, physical movement, and item cooking. Rising fields add progression-controlled movement of the hazard itself. Tornadoes move through the environment and affect nearby characters. Unstable rocks and ice have their own transitions from fixed geometry to falling or fractured objects. These are separate behaviors whose presence and configuration belong to particular scene content.

## Daylight and exposure

`DayNightManager.Update` advances time and calls `UpdateCycle`. The cycle blends profile values for the sun, sky, fog, and global shader parameters. Storm-related factors also affect light intensity. The master periodically synchronizes time, while each client updates its presentation from the available state.

Sun exposure connects that presentation to gameplay:

- `DayNightManager.UpdateCycle` writes the sun light's intensity and orientation.
- `HotSun.Update` checks its bounds and the current sun intensity.
- It performs a line check along the sun direction to determine whether the local character is sheltered.
- Eligible exposure accumulates time and reaches `CharacterAfflictions.AddSunHeat`.

The affliction system is the shared destination for several hazards, but their exposure models differ. Fog uses a height boundary. Sunlight uses a directional obstruction test. Wind samples a light volume and can additionally use a raycast when applying force. Those choices encode different approximations of shelter.

Apparently, the sun's render settings also get to decide whether the scout is hot.

`HotSun.Update` uses `DayNightManager.instance.sun.intensity` to decide whether exposure is eligible. That intensity is not a dedicated heat variable. `DayNightManager.UpdateCycle` calculates it from blended profile intensity, the sun gradient, and storm-related reductions before assigning it to the light.

The heat consumer therefore depends on an artistic output as well as its own volume and shelter test. Adjusting a profile's light intensity to change the appearance of a biome can move it across the heat gate without changing the `HotSun` configuration. A transition between lighting profiles can also change eligibility because the blended result is what the hazard reads.

That coupling makes lighting review part of gameplay-balance review. A brighter light is not merely a brighter image in this path, and the source offers no separate heat-eligibility contract explaining that dependency to a profile author. The shelter raycast establishes obstruction, but it cannot distinguish an intentional heat change from a visual intensity adjustment.

Using the rendered sun intensity as the authority for heat eligibility is bad separation of responsibilities. It turns lighting edits into potential hazard edits. Gameplay should read an environmental heat value; lighting can derive its appearance from the same time and weather inputs. If the coupling is intentional, it still needs validation of which profile changes cross the heat threshold. At present, an art adjustment can carry a gameplay consequence that the light component does not advertise.

## Wind

Wind separates scheduling from application. `WindChillZone.HandleTime` changes the active period through the master role. `Update` records whether the local and observed characters are inside its bounds. For an exposed local character, `ApplyStatus` samples `LightVolume` through `GetWindIntensityAtPoint`, adjusts climbing's minimum stamina multiplier, and applies the configured status or slipperiness contribution.

`WindChillZone.FixedUpdate` handles force. It calls `AddWindForceToCharacter` for the local character and for eligible bot characters, with adjustments for equipment and movement state. Item force has a separate loop through active ground items and a separate `AddWindForceToItem` method. Exposure, character force, and item force are therefore three related but independently implemented paths.

Physical effects also remain separate from status accumulation. Waterfall knockback, wind acceleration, and lava contact can change movement before a status total becomes large. A test that checks only affliction values would miss those consequences.

Local ownership checks and master-controlled scheduling divide work across participants. A shared storm period does not mean that one machine applies every character's force. The character-force consumer checks ownership, while individual hazards decide how to discover and sample their targets.

There are three versions of exposure here, and agreement between them is left to the reader.

`WindChillZone.ApplyStatus` computes `windPlayerFactor` from the local character's position. `AddWindForceToCharacter` accepts a character argument but multiplies its final force by that shared field. The caller also uses this method for bot characters inside the zone.

The factor belongs to one sampled position, but the force consumer treats it as applicable to every passed character. A locally owned bot at another location receives force scaled by the local player's exposure rather than its own light-volume sample. The method's ownership guard narrows which characters receive force; it does not correct the source of the exposure factor. Equipment and raycast adjustments are evaluated for the target character, making the mixed sampling scopes especially easy to overlook.

`AddWindForceToCharacter` has a misleading target boundary: it accepts one character while importing an exposure factor derived from the local player. Changing that player's shelter can change the factor applied to a bot elsewhere. The target argument does not actually contain the information needed to explain the result. That is a bad dependency between separate characters, and it belongs in the force calculation's inputs rather than being smuggled in through shared state.

The item path has a different mismatch. `FixedUpdate` reaches its item loop whenever item movement is enabled; that loop is outside the active-wind and delayed-force conditions used for the local character. `AddWindForceToItem` does not multiply by `currentForceMult` or check `windActive`. With the intensity-curve option disabled, it uses a constant strength factor. An inactive wind period therefore does not, by this path, stop item acceleration. With the curve option enabled, the result depends on the retained intensity value rather than a shared active-period rule.

These are separate consequences of the same design problem: scheduling, exposure, and force strength are recombined differently for each recipient. The correction is to calculate a target-specific exposure and a shared storm-strength value, then pass those explicit inputs into both character and item force application. That makes the active-period rule consistent without requiring identical physical responses for every target.

## Fog and water

`Fog.Update` runs movement, status application, and visual updates. `MakePlayerCold` uses the character's height relative to the fog and distinguishes a skeleton's injury response from the ordinary cold response. `ApplyVisuals` updates the fog transform and global fog height. The owner also publishes periodic height updates. Particle playback and the character's fog flag are handled alongside those steps.

Water uses a tracked population. `WaterZone.OnEnable` starts `UpdateTrackedCharactersRoutine` and subscribes to jump and landing events. The routine discovers player and bot characters, then classifies them using the character center and feet against the zone bounds. `UpdateCharacters` sends movement and jump modifiers to `CharacterMovement.SetWaterMovementModifier`; `UpdateCharacterVisuals` writes animation-related water state. `PlaySplash` handles the event-driven presentation. `OnDisable` stops the routine and removes the event subscriptions.

The movement consumer combines water movement and jump modifiers with existing values rather than blindly replacing them. `CharacterMovement.ResetFixedVars` resets those working modifiers. This matters when assessing multiple-zone behavior: the downstream accumulation rule must be read before assuming that the last water zone simply overwrites every movement effect.

`WaterZone.UpdateTrackedCharactersRoutine` adds discovered characters to `trackedCharacters`. When it encounters a character that now compares null, it yields and continues. It does not remove the dictionary entry. The ordinary update consumers then enumerate the same dictionary independently.

`UpdateCharacterVisuals` dereferences the key when the retained state is shallow or deep. `UpdateCharacters` dereferences it when sending movement modifiers, regardless of whether the recorded water state is none. Neither consumer performs the routine's null check. A dead reference skipped by the classifier is still handed to the effect application path.

This is a lifetime failure, not a disagreement about water depth. The classifier recognizes that the character no longer exists, but its response leaves the invalid entry available to other methods. The next consumer can fail while accessing the destroyed character instead of applying modifiers to the remaining live population.

The code also keeps the dictionary across disable and re-enable; `OnDisable` stops the coroutine and removes event subscriptions but does not clear tracked membership. That teardown is correct for the subscriptions and incomplete for the tracked-object lifetime. The dictionary's current contents cannot safely be treated as a live-character set merely because discovery runs repeatedly.

`WaterZone` does discovery without finishing the cleanup job. Skipping a destroyed entry leaves it in the registry for other consumers to encounter. Remove dead entries outside enumeration and guard against disappearance between scans; define what happens to membership on disable as well. A periodically refreshed collection is not a valid registry merely because new objects get added to it. This one neglects the other half of maintenance.

`WaterfallPusher.OnTriggerEnter` uses a different contact path. It identifies a local character through the ragdoll collider, applies a cooldown, and combines falling, camera feedback, and force. Ordinary water movement restrictions and waterfall knockback do not share the same trigger or lifetime.

## Lava and moving hazards

`Lava.Update` calls `Heat` and `TryCookItems`; its fixed update calls `DoEffects`. Heat uses horizontal bounds and distance from the surface, while the contact path adds force and affliction effects. Item cooking is handled on the master and passes through `ItemCooking`, connecting this hazard to the item lifecycle discussed in chapter 04. `LavaRising` separately tracks whether a rising field has started or ended and how far its progression has advanced.

Other hazards retain their own physical controllers. `Tornado.Update` handles its moving lifecycle and owner-side target selection; its character-force paths combine attraction, lift, and drag. `ShakyRock.FixedUpdate` checks whether a falling object has cleared neighboring geometry before restoring its collision layers. `ShakyIcicleIce.FixedUpdate` similarly manages clearance and a transition to fracture objects. Those clearance tests explain why a falling hazard is more than an animation.

`MirageManager.sampleCamera` prepares an auxiliary camera from the main camera before rendering it. That is presentation machinery, distinct from the contact and exposure mechanisms above. A visual illusion's rendered image alone does not establish which collision or interaction rules its associated objects use.

## Reading and changing this code

Trace exposure and application separately. A wind status value, a wind force, and a wind-driven item are different consumers even when they share one zone. For water, inspect `CharacterMovement`'s accumulation and reset behavior before attributing overlap effects solely to zone update order.

Visual changes also need gameplay review where render state feeds a hazard. Sun intensity is one such boundary; wind's light-volume sampling is another. Chapter 03 covers the status accumulator after a hazard submits its contribution. Correct exposure does not by itself establish correct accumulation, and the reverse is also true.

## Open questions

The wind and water failure mechanisms were traced in source, without a corresponding official-session reproduction. Which shipped zones enable item wind, which use the curve option, and where bot characters participate need scene-reference evidence. The water failure requires a tracked character to be destroyed while the zone remains a consumer; its frequency in ordinary transitions is unverified.

The relationship between light intensity and heat can be deliberate balancing. The unresolved question is whether profile authors intend every crossing of the rendering threshold to change heat eligibility. The source establishes the dependency, not the artistic intent.

`LavaTides` contains empty lifecycle methods in the inspected source. That does not prove the game lacks a tide-like hazard: other moving or rising lava components exist. The active scene components must be identified before attributing any visible behavior to that class name. Hazard placement, including roots-related wind and particular lava or mirage variants, likewise needs scene evidence beyond the presence of a component type.
