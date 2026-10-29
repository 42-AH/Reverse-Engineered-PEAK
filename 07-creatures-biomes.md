# Creatures and unusual biomes

A bee swarm, a spider, and a jellyfish do not need the same controller. PEAK gives them different controllers accordingly. What they all share is the need for reliable answers: does the target still exist, where is it, and is it protected? Specializing behavior does not eliminate these requirements.

This chapter explores those boundaries and then evaluates them. The main findings involve selecting a protection zone before testing its effectiveness, handling missing creature victims differently on host and client, and tying spider detection to rendered frames. Each has a named consumer; none requires assuming that every class appears in every island.

Bee swarms blend hive-related aggression, movement, and a stinging status field. Spiders scan below their starting position and can grab a character. Antlions pair a nearby target with attack animation and physical consequences. Venus flytraps hold a captured target and apply repeated effects until release. Scorpions extend the attack effect from the mobile-enemy framework.

Mushroom zombies and the Scoutmaster have more complex stateful behaviors. Their targeting and movement work alongside the character system rather than functioning as standard item interactions. Jellyfish operate differently: their contact response applies falling, impulses, and poison through a trigger path.

Gloom influences exposure and how characters perceive separation. Lit safe zones offer protection, while `StatusFieldGloom` provides spatial and visibility queries that other components use. Voice-stealing creatures use those queries to track their victims. The void biome also manages activation, spawn-location, and soul-progress state; it amounts to much more than just a visual profile.

## Swarms and sensing

`BeeSwarm.Update` changes aggression and updates the associated `StatusField`'s contribution. Its fixed update calls `UpdateBeehavior`, where the owner moves the swarm toward its current target or back to its last hive position. The hive relationship, angry-state timing, motion, and stinging field stay separate within that controller.

The mix of shared and specialized controllers reflects genuine behavioral differences. A contact trap must retain and release a victim; a swarm must maintain a moving threat field; a pursuing actor must track targets and manage movement state. Putting all three into a single attack callback would obscure those lifetimes.

Spider detection follows an explicit scheduling chain:

- `Spider.Start` registers with `SpiderManager`; destruction unregisters it.
- `SpiderManager.Update` moves through its list and calls `Spider.Scan`.
- `Scan` updates culling, checks idle and stun conditions, and gets a hit via `GetHit`.
- A qualifying local character reaches the spider's drop path; `SpiderTrigger.OnTriggerEnter` separately calls `GrabCharacter` on contact.

Registration persists even when a spider is culled. The manager can still call `Scan`, which reevaluates proximity through `UpdateCulled` and may reactivate the spider. Treating inactivity as proof that a spider never scans would overlook that manager-driven path.

Spider scanning reveals another explicit tradeoff. The manager spreads expensive sensing across updates rather than letting every spider check every frame. This reduces per-frame work but affects how quickly the behavior responds.

The sensing budget works in frames; the creature, however, lives in time.

`SpiderManager.Update` performs a fixed number of scans and advances a circular index. It does not track accumulated sensing time or assign each spider a next-scan time. The rate at which the list recycles therefore depends on rendered update frequency and on how many spiders are registered.

`Spider.Scan` makes gameplay decisions after checking culling and state, including whether it detects a character below. The scheduler's delay becomes part of how the encounter responds, not just an animation timing. A lower frame rate gives the manager fewer opportunities per second, and more registered spiders means longer waits between individual scans.

The loop also wraps within a single update if fewer spiders exist than the per-frame budget. An eligible remaining spider can be scanned multiple times that frame. This performs repeated proximity and sensing work without adding elapsed-time coverage, while larger populations get scanned less often individually.

`SpiderManager` budgets work but does not define responsiveness in time. The revisit interval depends on frame rate and list length, with repeated scans in one update possible for short lists. A schedule based on due times with a bounded per-frame budget would maintain load control while making latency clear. The current approach makes the sensing policy an accidental result of rendering speed, which is a poor foundation for consistent creature behavior.

## Targets, traps, and contact effects

`Antlion.Update` picks a target and selects active or inactive behavior. `DoActive` leads to an attack sequence, and `Attack` orchestrates animation, movement drag, falling, force, and injury. `VenusFlyTrap.OnCollide` receives a character collision instead and starts capture via `RPC_BitePlayer`. While holding a target, `UpdateAttack` inflicts repeated poison; `RPC_LetGo` removes target references and the capture status.

The common pieces exist at a lower level: character references, afflictions, physics, and ownership checks. Each controller thus manages when those references stay valid. A spatial helper like `GloomSafeZone` becomes a gameplay dependency for consumers with very different effects.

`GloomVoiceConfuser.Update` determines which character belongs to each active voice-thief assignment through `PlayerHandler.TryGetCharacter`. Its missing-character branch handles both lookup failure and the host-role check. The host removes the thief and stops; a non-host does not take that branch just because the lookup failed.

Processing then continues into visibility and state calculations using the unresolved character. Those consumers read the character's center, head, or affliction references. `PlayerHandler.TryGetCharacter` explicitly returns failure with a null character when the player or character cannot be found. The failed lookup thus yields a defined output that the non-host continuation cannot safely use.

`PruneThiefAssignments` beforehand does not establish the missing invariant. It removes assignments whose thief object is gone but does not require that the assigned victim character exists locally. A live thief and an unavailable victim are separate conditions, yet the code validates only the first before entering the loop.

The resulting failure point is a client-side null dereference during assignment processing. It rests on an assignment outlasting local victim availability rather than on any particular visibility rule. Host authority to retire the creature should not determine whether a client can dereference the victim.

The client path is incorrect to proceed after victim lookup fails. Host authority decides whether to retire or reassign the thief; it does not make a missing client-side character safe to dereference. All participants must halt target-dependent work until they have a valid target. Combining that validity check with the host-only lifecycle decision leaves a predictable null-dereference boundary during assignment processing.

`Scorpion` extends `Mob` and overrides `InflictAttack`. Its local-character effect combines immediate poison, a timed poison affliction, and force. This shows specialization within a shared framework, unlike the antlion's self-contained target and attack logic.

Zombie bites start through `MushroomZombieBiteCollider.OnTriggerEnter`. That path applies immediate effects, adds `Affliction_ZombieBite`, causes falling, and notifies the parent via `OnBitCharacter`. The zombie controller keeps its own states: sleeping, waking, chasing, and lunging. The Scoutmaster's `Update` similarly separates presentation from owner-side choices before picking target acquisition or pursuit. These controllers function as stateful actors, not merely as the final damage call.

`SlipperyJellyfish.OnTriggerEnter` verifies contact and checks a local cooldown, then passes the result through `TriggerRelay`. `Trigger` applies the character's response. The component contains no target-search loop, and its distinctive shape or animation does not make it an AI system.

## Gloom and the Void

Two components directly use Gloom protection. `StatusFieldGloom.PointInField` first checks bounds, then consults `GloomSafeZone` to determine whether a point remains exposed. `GhostBall` calls `GloomSafeZone.CharacterProtectedFromStatus`, gets back the zone, and applies that zone to its repulsion logic. A safe-zone query therefore influences both status classification and physical hazard behavior.

Finding the nearest center is straightforward. It is not what the protection check actually needs to know.

`GloomSafeZone.TryGetClosestSafeZone` locates the nearest lit zone by measuring distance to its center. The static `CharacterProtectedFromStatus` then asks only that one zone whether it protects the character. `StatusFieldGloom.PointInField` follows the same nearest-first approach for a point.

This order fails when zones have different protection radii. A zone can be nearest by center yet fail to contain the point, while another lit zone actually contains the point despite its center being farther away. Rejecting the first zone does not prove that no zone provides protection. The search discards the very information needed to answer what its consumer wants.

`StatusFieldGloom` treats that result as a definitive exposure decision. It can mark a point as inside the hazardous field even though another registered lit zone covers it. `GhostBall` takes the character-level result and selected zone to drive its repulsion behavior. This selection error thus spreads from status classification into physical hazard response.

The issue is not whether radii should be larger or smaller. The problem is that selection happens before validation. Changing a radius shifts which points expose the error but does not resolve the query itself.

`GloomSafeZone` answers the wrong question for its protection users. Picking the nearest center before checking validity can dismiss protection offered by another zone that qualifies. Larger radii will not fix a query that picks first and checks later. Find all eligible protecting zones first, preserving what each zone actually protects; pick one only if needed. Nearest-center lookup can stay a separate operation. Using it as a protection test is the wrong algorithm.

`GloomVoiceConfuser.Update` handles `VoiceThief.Assignments`, identifies the assigned character, and tests separation and visibility with `StatusFieldGloom`. `CharacterCanSeePoint` examines character state, whether the endpoints sit inside the field, distance, and field of view. Voice transport belongs in chapter 10; the relevant responsibility here is determining which creature assignment is active and what that assignment does.

`VoidBiome.Activate` subscribes to soul progress; `Deactivate` clears active state and removes the subscription. Spawn lookup comes separately through `GetSpawnPoint` and `GetSpawnPosition`. These methods and the soul-pillar component set up state and interaction machinery without by themselves determining the path through every authored scene.

## Reading and changing this code

Check the controller's target lifetime and its attack effect. Adding a status effect does not define how the target gets found, held, released, or handled by another participant. Collision-triggered creatures and manager-scanned creatures use different entry points.

For Gloom modifications, separate nearest-zone information from protection and separate field-of-view checks from physical occlusion. `CharacterCanSeePoint` does not perform a terrain raycast within the method being examined. A consumer that uses that result must know exactly which visibility concept it represents before adding further consequences to it.

## Open questions

The safe-zone selection defect is conditional on overlapping eligibility with different zone geometry or specialized rules. The shipped placement and radius combinations were not established in this pass. Likewise, the missing-victim failure was traced through the client branch without reproducing an assignment/character lifetime mismatch in an official session.

Spider detection timing follows the scheduling mechanism described above, but its perceived responsiveness under ordinary frame rates and actual island populations remains unmeasured. No fixed reaction-time claim is made.

The presence of biome and creature classes is not evidence that every variant is active in a particular map. Void/Nadir routing, soul-pillar placement, and creature populations require scene references or official observations. Visibility helpers also encode design choices: the absence of a terrain occlusion query in `CharacterCanSeePoint` is established, but whether that approximation is intended for each consumer remains a design question rather than a demonstrated sight-through-walls bug.
