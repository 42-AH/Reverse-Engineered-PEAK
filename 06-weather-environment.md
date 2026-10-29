# Weather and environmental hazards

Weather and environmental hazards are not a single unified system with a few universal effects. Fog creates conditions, water affects movement, wind exerts force on bodies, and sunlight uses a rendering value to determine whether the scout overheats. These separate behaviors mean the line between presentation and gameplay becomes quite fluid.

Environmental damage lacks a single pipeline. This separation allows each hazard to develop specialized behaviors, but it also leaves shared questions like exposure detection, effect duration, and character validity to individual components.

The day/night cycle updates lighting and shader parameters. Wind has active and inactive phases, exposure-based status application, and physical effects. Fog moves through the level while applying a status consequence based on its current height. Sun exposure varies by location and shelter. Water affects movement and creates splash effects, while waterfalls apply a distinct knockback. Lava involves proximity heat, contact damage, physical movement, and item cooking. Rising fields control the hazard's movement as progression advances. Tornadoes travel through the environment and impact nearby characters. Unstable rocks and ice transition from fixed geometry to falling or fractured objects. Each represents a separate behavior tied to specific scene content.

## Daylight and exposure

`DayNightManager.Update` advances time and calls `UpdateCycle`. The cycle blends profile values for the sun, sky, fog, and global shader parameters. Storm factors also influence light intensity. The master synchronizes time periodically, while each client updates its presentation based on available state.

Sun exposure connects that visual representation to actual gameplay:

- `DayNightManager.UpdateCycle` writes the sun light's intensity and direction.
- `HotSun.Update` checks its bounds and reads the current sun intensity.
- It performs a line check along the sun direction to see whether the local character has shelter.
- If exposed, the character accumulates heat until reaching `CharacterAfflictions.AddSunHeat`.

The affliction system is where several hazards converge, though their exposure models vary. Fog relies on height boundaries. Sunlight uses directional obstruction tests. Wind samples a light volume and may use raycasts when applying force. Each approach represents a different approximation of shelter.

Interestingly, the sun's render settings also determine whether the scout gets hot.

`HotSun.Update` uses `DayNightManager.instance.sun.intensity` to decide whether exposure applies. That intensity is not a dedicated heat variable. `DayNightManager.UpdateCycle` derives it from blended profile intensity, the sun gradient, and storm-related reductions before assigning it to the light.

The heat system therefore depends on an artistic output along with its own volume and shelter test. Adjusting a profile's light intensity to change how a biome looks can push it across the heat threshold without touching `HotSun` configuration. Transitions between lighting profiles can shift eligibility too, since the hazard reads the blended result.

This coupling means lighting changes affect gameplay balance. A brighter light is not just a brighter image here; the source provides no separate heat contract explaining that dependency to profile authors. The shelter raycast determines obstruction, but cannot tell whether an intensity change was intentional for gameplay or just visual adjustment.

Using rendered sun intensity as the heat authority creates poor separation of concerns. It turns lighting edits into potential hazard edits. Gameplay should read an environmental heat value while lighting derives its appearance from the same time and weather inputs. Even if the coupling is deliberate, it requires validation of which profile changes cross the heat threshold. Currently, an art adjustment can carry gameplay consequences that the light component never mentions.

## Wind

Wind separates scheduling from application. `WindChillZone.HandleTime` controls the active period through the master role. `Update` tracks whether the local and observed characters are within bounds. For exposed local characters, `ApplyStatus` samples `LightVolume` through `GetWindIntensityAtPoint`, adjusts climbing's minimum stamina multiplier, and applies the configured status or slipperiness.

`WindChillZone.FixedUpdate` handles force application. It calls `AddWindForceToCharacter` for the local character and eligible bots, with adjustments for equipment and movement state. Item force runs a separate loop through active ground items using `AddWindForceToItem`. Exposure, character force, and item force are three related but independently implemented paths.

Physical effects also stay separate from status accumulation. Waterfall knockback, wind acceleration, and lava contact can change movement before affliction values grow large. A test checking only afflictions would miss these consequences.

Local ownership checks and master-controlled scheduling split work across participants. A shared storm period does not mean one machine applies force to every character. The character-force consumer checks ownership while individual hazards discover and sample their targets differently.

Three different exposure calculations exist here, and nothing ensures they agree.

`WindChillZone.ApplyStatus` computes `windPlayerFactor` from the local character's position. `AddWindForceToCharacter` accepts a character argument but scales its final force by that same field. The caller also uses this method for bots inside the zone.

The factor comes from one sampled position, yet the force consumer applies it to any passed character. A locally owned bot at another location receives force based on the local player's exposure rather than its own light-volume sample. The method's ownership guard restricts which characters get force, but does not fix the source of the exposure factor. Equipment and raycast adjustments evaluate the target character, making this mixed sampling scope easy to miss.

`AddWindForceToCharacter` has a misleading target boundary. It accepts one character while importing an exposure factor derived from the local player. Changing that player's shelter alters the factor applied to a bot elsewhere. The target argument lacks the information needed to explain the result. This represents a bad dependency between separate characters that belongs in the force calculation's inputs rather than hidden in shared state.

The item path has a different problem. `FixedUpdate` runs its item loop whenever item movement is enabled, outside the active-wind and delayed-force conditions used for the local character. `AddWindForceToItem` does not multiply by `currentForceMult` or check `windActive`. With intensity-curve disabled, it uses constant strength. An inactive wind period therefore does not, through this path, stop item acceleration. With the curve enabled, it reads the retained intensity value rather than applying a shared active-period rule.

These stem from the same underlying issue: scheduling, exposure, and force strength recombine differently for each recipient. The fix requires calculating target-specific exposure and a shared storm-strength value, then passing these explicit inputs into both character and item force application. This keeps the active-period rule consistent without requiring identical physical responses across all targets.

## Fog and water

`Fog.Update` handles movement, status application, and visual updates. `MakePlayerCold` uses the character's height relative to fog and handles a skeleton's injury response separately from ordinary cold. `ApplyVisuals` updates the fog transform and global fog height. The owner publishes periodic height updates. Particle playback and the character's fog flag are managed alongside these steps.

Water uses a tracked population. `WaterZone.OnEnable` starts `UpdateTrackedCharactersRoutine` and subscribes to jump and landing events. The routine discovers player and bot characters, then classifies them using the character center and feet against the zone bounds. `UpdateCharacters` sends movement and jump modifiers to `CharacterMovement.SetWaterMovementModifier`. `UpdateCharacterVisuals` writes animation-related water state. `PlaySplash` handles the event-driven presentation. `OnDisable` stops the routine and removes the event subscriptions.

The movement consumer combines water movement and jump modifiers with existing values rather than replacing them outright. `CharacterMovement.ResetFixedVars` resets those working modifiers. This matters for multiple-zone behavior: the downstream accumulation rule must be understood before assuming that the last water zone simply overwrites every movement effect.

`WaterZone.UpdateTrackedCharactersRoutine` adds discovered characters to `trackedCharacters`. When it encounters a character that now compares null, it yields and continues without removing the dictionary entry. The ordinary update consumers then enumerate the same dictionary independently.

`UpdateCharacterVisuals` dereferences the key when the retained state is shallow or deep. `UpdateCharacters` dereferences it when sending movement modifiers, regardless of whether the recorded water state is none. Neither consumer performs the routine's null check. A dead reference skipped by the classifier still gets passed to the effect application path.

This represents a lifetime failure, not a disagreement about water depth. The classifier recognizes that the character no longer exists, but its response leaves the invalid entry available to other methods. The next consumer can fail while accessing the destroyed character instead of applying modifiers to the remaining live population.

The code also keeps the dictionary across disable and re-enable. `OnDisable` stops the coroutine and removes event subscriptions but does not clear tracked membership. That teardown is correct for the subscriptions yet incomplete for the tracked-object lifetime. The dictionary's current contents cannot safely be treated as a live-character set just because discovery runs repeatedly.

`WaterZone` performs discovery without finishing the cleanup job. Skipping a destroyed entry leaves it in the registry for other consumers to find. Remove dead entries outside enumeration and guard against disappearance between scans. Define what happens to membership on disable as well. A periodically refreshed collection is not a valid registry simply because new objects get added to it. This one neglects the other half of maintenance.

`WaterfallPusher.OnTriggerEnter` uses a different contact path. It identifies a local character through the ragdoll collider, applies a cooldown, and combines falling, camera feedback, and force. Ordinary water movement restrictions and waterfall knockback do not share the same trigger or lifetime.

## Lava and moving hazards

`Lava.Update` calls `Heat` and `TryCookItems`. Its fixed update calls `DoEffects`. `Heat` uses horizontal bounds and distance from the surface, while the contact path adds force and affliction effects. Item cooking runs on the master and passes through `ItemCooking`, connecting this hazard to the item lifecycle discussed in chapter 04. `LavaRising` separately tracks whether a rising field has started or ended and how far its progression has advanced.

Other hazards maintain their own physical controllers. `Tornado.Update` handles its moving lifecycle and owner-side target selection. Its character-force paths combine attraction, lift, and drag. `ShakyRock.FixedUpdate` checks whether a falling object has cleared neighboring geometry before restoring its collision layers. `ShakyIcicleIce.FixedUpdate` similarly manages clearance and a transition to fracture objects. Those clearance tests show why a falling hazard is more than an animation.

`MirageManager.sampleCamera` prepares an auxiliary camera from the main camera before rendering it. That is presentation machinery, separate from the contact and exposure mechanisms above. A visual illusion's rendered image alone does not establish which collision or interaction rules its associated objects use.

## Reading and changing this code

Trace exposure and application separately. A wind status value, a wind force, and a wind-driven item are different consumers even when they share one zone. For water, inspect `CharacterMovement`'s accumulation and reset behavior before attributing overlap effects solely to zone update order.

Visual changes also need gameplay review where render state feeds a hazard. Sun intensity is one such boundary. Wind's light-volume sampling is another. Chapter 03 covers the status accumulator after a hazard submits its contribution. Correct exposure does not by itself establish correct accumulation, and the reverse is also true.

## Open questions

The wind and water failure mechanisms were traced in source, without a corresponding official-session reproduction. Which shipped zones enable item wind, which use the curve option, and where bot characters participate need scene-reference evidence. The water failure requires a tracked character to be destroyed while the zone remains a consumer; its frequency in ordinary transitions is unverified.

The relationship between light intensity and heat can be deliberate balancing. The unresolved question is whether profile authors intend every crossing of the rendering threshold to change heat eligibility. The source establishes the dependency, not the artistic intent.

`LavaTides` contains empty lifecycle methods in the inspected source. That does not prove the game lacks a tide-like hazard: other moving or rising lava components exist. The active scene components must be identified before attributing any visible behavior to that class name. Hazard placement, including roots-related wind and particular lava or mirage variants, likewise needs scene evidence beyond the presence of a component type.
