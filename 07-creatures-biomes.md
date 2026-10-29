# Creatures and unusual biomes

A bee swarm, a spider, and a jellyfish do not need the same AI. PEAK sensibly gives them different controllers. What they do all need is a reliable answer to whether their target still exists, where it is, and whether it is protected. Specializing the behavior does not make those obligations disappear.

This chapter traces those boundaries before evaluating them. The clearest findings concern choosing a protection zone before testing whether it protects, handling a missing creature victim differently on host and client, and tying spider detection work to rendered frames. Each has a named consumer; none depends on assuming that every class appears in every island.

Bee swarms combine hive-related aggression, movement, and a stinging status field. Spiders scan below their starting position and can grab a character. Antlions combine a nearby target with attack animation and physical consequences. Venus flytraps retain a captured target and apply repeated effects until release. Scorpions specialize the attack effect supplied by the mobile-enemy framework.

Mushroom zombies and the Scoutmaster have larger stateful behaviors. Their targeting and movement coordinate with the character system rather than being implemented as ordinary item interactions. Jellyfish are different again: their contact response applies falling, impulses, and poison through a trigger path.

Gloom affects exposure and perceptions of separation. Lit safe zones provide protection, while `StatusFieldGloom` supplies spatial and visibility queries used by other components. Voice-stealing creatures use those queries to manage their victims. The void biome also carries activation, spawn-location, and soul-progress state; it is more than a visual profile.

## Swarms and sensing

`BeeSwarm.Update` updates aggression and changes the associated `StatusField`'s contribution. Its fixed update reaches `UpdateBeehavior`, where the owner moves the swarm toward its current target or its last hive position. The hive relationship, angry-state timing, motion, and stinging field remain separate responsibilities inside that controller.

The mixture of shared and specialized controllers follows real behavioral differences. A contact trap needs to retain and release a victim; a swarm needs a moving threat field; a pursuing actor needs target memory and movement state. Forcing all three into a single attack callback would hide those lifetimes.

Spider detection has an explicit scheduling chain:

- `Spider.Start` registers with `SpiderManager`; destruction unregisters it.
- `SpiderManager.Update` advances through its list and calls `Spider.Scan`.
- `Scan` updates culling, checks the idle and stun conditions, and obtains a hit through `GetHit`.
- A qualifying local character reaches the spider's drop path; `SpiderTrigger.OnTriggerEnter` separately reaches `GrabCharacter` on contact.

Registration is not removed merely because a spider is culled. The manager remains able to call `Scan`, which reevaluates proximity through `UpdateCulled` and can reactivate it. Treating inactivity alone as proof that a spider never scans would miss that manager-driven path.

Spider scanning shows another explicit tradeoff. The manager distributes expensive sensing across updates instead of having every spider probe every frame. That limits per-frame work, but its scheduling unit affects the behavior's responsiveness.

The sensing budget is in frames; the creature, inconveniently, lives in time.

`SpiderManager.Update` performs a fixed number of scans and advances a circular index. It does not accumulate elapsed sensing time or assign a next-scan time to each spider. The rate at which the list is revisited therefore depends on rendered update frequency and on the number of registered spiders.

`Spider.Scan` makes gameplay decisions after its culling and state checks, including whether a character is detected below it. The scheduler's delay is consequently part of encounter response, not merely an animation update. A lower frame rate gives the manager fewer opportunities per second, and a larger registered population increases the wait between scans of an individual spider.

The loop also wraps within the same update when fewer spiders exist than the per-frame budget. An eligible remaining spider can be scanned repeatedly in that frame. That spends repeated proximity and sensing work without adding elapsed-time coverage, while larger populations receive less frequent individual attention.

`SpiderManager` budgets work but fails to define responsiveness in time. The revisit interval is left to frame rate and list length, with repeated scans in one update possible for a short list. A due-time schedule with a bounded per-frame budget would preserve load control while making latency explicit. The present arrangement makes the sensing policy an accidental consequence of rendering speed. That is a poor basis for consistent creature behavior.

## Targets, traps, and contact effects

`Antlion.Update` selects a target and chooses active or inactive behavior. `DoActive` reaches an attack sequence, and `Attack` coordinates animation, movement drag, falling, force, and injury. `VenusFlyTrap.OnCollide` instead receives a character collision and begins capture through `RPC_BitePlayer`. While holding a target, `UpdateAttack` applies repeated poison; `RPC_LetGo` clears target references and the capture status.

The shared pieces are lower-level: character references, afflictions, physics, and ownership checks. That leaves each controller responsible for deciding when those references remain valid. It also means that a spatial helper such as `GloomSafeZone` becomes a gameplay dependency for consumers with quite different effects.

`GloomVoiceConfuser.Update` resolves the character associated with each active voice-thief assignment through `PlayerHandler.TryGetCharacter`. Its missing-character branch combines lookup failure with the host-role check. The host dismisses the thief and skips further processing; a non-host does not take that branch solely because the lookup failed.

Execution then continues into visibility and state calculations using the unresolved character. Those consumers read the character's center, head, or affliction references. `PlayerHandler.TryGetCharacter` explicitly returns failure with a null character when the player or character cannot be found. The failed lookup therefore has a defined output that the non-host continuation cannot safely consume.

The preceding `PruneThiefAssignments` does not establish the missing invariant. It removes assignments whose thief object is gone. It does not require that the assigned victim character exists locally. A live thief and an unavailable victim are different conditions, and the code currently validates only the first before entering the loop.

The resulting failure boundary is a client-side null dereference during assignment processing. It depends on the assignment outliving local availability of its victim, rather than on any particular visibility rule. Host authority to dismiss the creature should not determine whether a client is allowed to dereference the victim.

The client path is wrong to proceed after victim lookup fails. Host authority decides whether to retire or reassign the thief; it does not make a missing client-side character safe to dereference. Every participant must stop target-dependent work until it has a target. Combining that validity check with the host-only lifecycle decision leaves a predictable null-dereference boundary during assignment processing.

`Scorpion` derives from `Mob` and overrides `InflictAttack`. Its local-character effect combines an immediate poison contribution, a timed poison affliction, and force. This is an example of specialization inside a shared framework, unlike the antlion's self-contained target and attack flow.

Zombie bites enter through `MushroomZombieBiteCollider.OnTriggerEnter`. That path applies immediate effects, installs `Affliction_ZombieBite`, causes falling, and notifies the parent through `OnBitCharacter`. The zombie controller maintains its own states, including sleeping, waking, chasing, and lunging. The Scoutmaster's `Update` similarly separates presentation from owner-side decisions, then chooses target acquisition or pursuit. These controllers need to be understood as stateful actors, not only as the final damage call.

`SlipperyJellyfish.OnTriggerEnter` checks contact and a local cooldown before dispatching through `TriggerRelay`. `Trigger` applies the character response. There is no target-search loop in that component. Its unusual shape or animation does not make its implementation an AI system.

## Gloom and the Void

Gloom protection has two direct consumers. `StatusFieldGloom.PointInField` first checks its bounds and then consults `GloomSafeZone` before deciding that a point remains exposed. `GhostBall` also asks `GloomSafeZone.CharacterProtectedFromStatus` and uses the returned zone when applying repulsion. A safe-zone query therefore affects both status classification and physical hazard behavior.

Nearest center is a convenient question to answer. It is not the question the protection check asked.

`GloomSafeZone.TryGetClosestSafeZone` chooses the nearest lit zone by distance to its center. The static `CharacterProtectedFromStatus` then asks only that selected zone whether it protects the character. `StatusFieldGloom.PointInField` follows the same nearest-first pattern for a point.

The order is incorrect for zones with different protection radii. A zone can be nearest by center while not containing the point, and another lit zone can contain the point despite having a more distant center. Rejecting the first does not establish that no zone protects. The search has discarded the information needed to answer its consumer's question.

`StatusFieldGloom` consumes that result as an exposure decision. It can classify a point as inside the hazardous field even though another registered lit zone covers it. `GhostBall` consumes the character-level result and the selected zone for its repulsion logic. The same selection mistake therefore crosses from status classification into a physical hazard response.

The problem is not whether the radii should be larger or smaller. It is that selection precedes eligibility. Changing a radius changes which points expose the error but does not fix the query.

`GloomSafeZone` answers the wrong question for its protection consumers. Selecting the nearest center before checking eligibility can reject protection supplied by another valid zone. Larger radii do not fix a query that selects first and validates afterward. Find eligible protecting zones first, preserving each zone's virtual protection rule; choose a representative only if one is actually needed. Nearest-center lookup can remain a separate operation. Using it as a protection test is simply the wrong algorithm.

`GloomVoiceConfuser.Update` manages `VoiceThief.Assignments`, resolves the assigned character, and evaluates separation and visibility through `StatusFieldGloom`. `CharacterCanSeePoint` considers character state, whether the endpoints are inside the field, distance, and field of view. Voice transport belongs in chapter 10; here the relevant responsibility is deciding which creature assignment is active and what that assignment means.

`VoidBiome.Activate` subscribes to soul progress; `Deactivate` clears its active state and removes the subscription. Spawn lookup is provided separately through `GetSpawnPoint` and `GetSpawnPosition`. Those methods and the soul-pillar component establish state and interaction machinery, without by themselves establishing the route through every authored scene.

## Reading and changing this code

Read the controller's target lifetime as well as its attack effect. Adding a status effect does not establish how the target is acquired, retained, released, or resolved on another participant. Collision-triggered creatures and manager-scanned creatures have different entry points.

For Gloom changes, distinguish nearest-zone information from protection and distinguish field-of-view tests from physical occlusion. `CharacterCanSeePoint` does not perform a terrain raycast in the inspected method. A consumer using that result needs to understand exactly which notion of visibility it represents before adding another consequence to it.

## Open questions

The safe-zone selection defect is conditional on overlapping eligibility with different zone geometry or specialized rules. The shipped placement and radius combinations were not established in this pass. Likewise, the missing-victim failure was traced through the client branch without reproducing an assignment/character lifetime mismatch in an official session.

Spider detection timing follows the scheduling mechanism described above, but its perceived responsiveness under ordinary frame rates and actual island populations remains unmeasured. No fixed reaction-time claim is made.

The presence of biome and creature classes is not evidence that every variant is active in a particular map. Void/Nadir routing, soul-pillar placement, and creature populations require scene references or official observations. Visibility helpers also encode design choices: the absence of a terrain occlusion query in `CharacterCanSeePoint` is established, but whether that approximation is intended for each consumer remains a design question rather than a demonstrated sight-through-walls bug.
