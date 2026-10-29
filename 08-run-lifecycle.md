# Airport, ascent, and ending

Starting a run is more than just launching the game. It's a handoff between boot services, the airport, scene loading, a run manager, and character state. Ending one involves the results screen as well. Many places can signal that a run has started or stopped, but making them all mean the same thing is where things get messy.

The airport lets players choose an ascent or custom rules and then depart. Once the island and character are ready, `RunManager` initializes the run, manages its identity and clock, and handles continuation from saved progress. Campfires serve double duty: they provide a local interaction point while also supporting group participation, progression, and saving.

Death modifies the character's state and possessions. Revival brings back a playable character and records the recovery. When the run ends, it classifies individual outcomes before showing either a cutscene or the results screen. Another player succeeding looks different from your character succeeding, and the results interface reflects that distinction.

## Boot and departure

`GameBooter.Initialize` sets up application and scene callbacks. `AutoBoot` creates the game object and initializes `GameHandler` and `UIInputHandler` when the first scene loads. Invitation readiness happens separately through the networking interfaces in chapter 09.

Departure follows this sequence:

- `AirportCheckInKiosk.Interact_CastFinished` opens the boarding pass and passes its kiosk reference along.
- `BoardingPass.StartGame` resolves the selected ascent for custom-run mode and calls the kiosk.
- `AirportCheckInKiosk.StartGame` submits the run settings for departure if loading isn't already underway.
- `LoadIslandMaster` picks the destination. `BeginIslandLoadRPC` applies the ascent and run settings, then starts loading.

Chapter 05 explains the scene catalogue and bundle requirements. The key distinction here is that choosing a destination doesn't start the run clock.

## Starting a run

`RunManager.Start` sets up map state and waits for room membership, a local character, and loading to finish. An airport character stops there without starting a run. An island character moves forward to `StartRun`. The non-host then retrieves the run identity, while the host sets things up with a delay, publishes a new identity for a fresh run, and begins time synchronization. If restoring from a save, that branch rebuilds spawn history and completes the saved setup instead.

Getting a run identifier is straightforward. Making sure the run actually has one is the tricky part.

`RunManager.GetRunId` calls `NetCode.Matchmaking.TryGetLobbyData` but ignores whether it succeeds. It passes the returned string directly to `Guid.Parse`, then marks the identity as retrieved. A missing or invalid value doesn't register as a pending state; parsing fails before the success flag gets set.

The calling code reveals the timing problem. In `RunManager.Start`, a non-host calls `GetRunId` once its loading prerequisites are done. The host publishes its fresh-run identity later in its setup, after an explicit wait. Local scene readiness and published identity availability aren't the same moment, but the client treats the first as sufficient for the second.

There's also a separate `RPC_ReceiveRunId` path that can deliver the identity. That path doesn't verify the earlier assumption, so there are two possible sources for identity readiness whose order needs coordinating. If the message arrived first, `_hasRunId` skips the lookup. If not, the lookup still assumes a valid value is there right away.

The immediate problem is an initialization exception when the prerequisite is missing or malformed. It also leaves identity-dependent consumers without a clear readiness contract. `Quicksave` grabs the identity from `RunManager` when it builds run progress, and the diagnostic display reads the same property. Neither property access actually signals whether identity setup worked.

Run identity readiness isn't modeled correctly. The code should track pending, valid, and failed states, with both message receipt and property lookup going through proper validation. Until that validation passes, consumers shouldn't assume they have an identity. A longer delay would just hide the missing coordination. The dependency is real; the implementation treats it like a timing issue.

Both entry points need validation. Running the message path and lookup path through a single checked operation would stop the same state from making different assumptions about readiness and input quality.

`StartRun` listens for room population and owner changes and queues an end-condition check. `RunManager.Update` moves the clock forward while active, periodically syncs it on the host, and runs scheduled checks. `OnCharacterRegistered` sends time state to a character when it arrives. The registered character and the run's identity have different paths to being ready.

## Campfires, death, and revival

Joining a campfire is a deliberate choice. `EveryoneInRange` compares nearby characters against active participants. `Light_Rpc` updates the fire, then either ends a miniature run at its ending campfire or moves the map through `MapHandler.GoToSegment`. The progression path also triggers a quicksave. Morale shifts and nearby-character recovery are other campfire roles; cooking relies on the item system from chapter 04 rather than defining the checkpoint transition itself.

Checkpoint progression mixes several outcomes without treating them as the same thing. A lit fire has visuals, an advancing fire changes segment state, and a saved checkpoint preserves progress. Restoring from a save requires rebuilding those relationships rather than just placing the player near a campfire.

Death flows into `Character.RPCA_Die`, which unequips items, marks the character as dead, logs the event, drops possessions, handles the death display, and repositions the character before checking if the run should end. `CheckEndGame` does the host-side check for all-dead. Ghost display is separate: `PlayerGhost.RPCA_InitGhost` links an owner to a target and restores the owner's cosmetics, with special logic for how the owner sees themselves.

Revival starts at `RPCA_ReviveAtPosition` and continues into `ReviveCharacter`. That function resets death and unconsciousness, removes related effects, and optionally adds post-revival status. The caller then moves the character, updates progress and stats, and alerts `ReconnectHandler.UpdateFromRevive` to the new position. That recovery update matters because revival is both an immediate gameplay change and a new checkpoint from which later reconnection continues.

## Ending the run

`Character.RPCEndGame` processes the outcome. It marks game over, wipes resumable save data, evaluates characters through `CheckWinCondition`, and refreshes their stats. Depending on the result and run mode, it opens `EndScreen` immediately or launches the appropriate `PeakHandler` ending sequence. It then fires the run-ended event and calls `RunManager.EndGame`.

The standard summit result checks whether a living character has reached the summit or is on the helicopter rope; miniature-run and Nadir use different logic. `PeakHandler.EndCutscene` and `EndCutsceneFinal` build different presentations before heading to the results screen. `EndScreen` shows elapsed time, plays the timeline, and tells apart personal success, another player's success, and failure. Ending involves coordinating multiple managers, not just one scene load.

The character layer determines the final result, while `RunManager` handles shared timing and identity. That way character-specific outcomes feed into one results sequence, but it requires the clock to stop before result consumers read it. The results screen also triggers time-achievement checks, so elapsed time is gameplay data, not just window dressing.

A stopped clock should be the straightforward part of this design.

`RunManager.TimeSinceRunStarted` includes the current frame's delta when the clock hasn't been updated that frame. This compensates for access before `Update` runs while the clock is active. The getter doesn't check `timerActive` though. After the clock stops, frames still satisfy the not-updated-this-frame condition, so the getter returns the stored final time plus whatever delta the caller's frame contributes.

This isn't a stopped timer that keeps advancing indefinitely. The extra delta never gets written back to the stored time; it's just a read-time offset that shifts each frame a caller asks for it. This limitation reduces the issue but doesn't eliminate it. A final elapsed time shouldn't fluctuate based on how long the results screen's current frame takes.

Network synchronization introduces another offset. `RPC_SyncTime` adjusts incoming time by elapsed network time regardless of whether the accompanying active flag is set. For a running clock, that adjustment estimates its current value. For a stopped clock, it changes the final result depending on when the stop message arrives. The method has the information to tell these cases apart but applies running-clock logic to both.

`RunManager.EndGame` sends a stopped-clock update, and `EndScreen` later reads `TimeSinceRunStarted` to populate its time label. `AchievementManager.TestTimeAchievements` reads that same property, converts it to a whole-second value, and uses it for time-based progression. `CharacterStats` uses the clock when adding timeline entries. The final-time contract therefore affects display, achievement evaluation, and run history.

The core issue is that there's no immutable final elapsed value. A stopped clock is just a boolean next to a value whose getter and synchronization continue applying live-time adjustments. Different consumers read at different frames, and stop updates reach participants at different times.

The clock's final-state contract has broken down. Finalization should lock in a single elapsed value and share it without applying live-clock or transmission-age adjustments. An inactive accessor should return that value straightforwardly. Results and time-achievement checks shouldn't each get a fresh estimate for a run that's already finished. Keeping running-clock compensation active after the run ends only adds uncertainty where the code should be cutting it off.

## Reading and changing this code

Decouple scene readiness from run readiness. A character can exist before identity setup finishes, and a checkpoint's visual state alone doesn't convey saved progression. Changes to revival need to account for statistics and reconnect records alongside health and position.

Handle final results as shared gameplay data. A presentation change shouldn't create a new time estimate or determine victory separately from `CheckWinCondition`. Lower-level movement and status updates can live in their own systems, but their completion signals must stay aligned with what lifecycle consumers expect.

## Open questions

The identity failure is conditional on lookup returning an unavailable or invalid value before another identity path completes. That timing was traced in source but not reproduced in an official multiplayer session. The actual ordering of lobby data delivery and identity messages needs observation before estimating frequency or recovery behavior.

The clock offsets are established by the getter and synchronization logic. Their visible magnitude depends on frame duration, delivery timing, and result formatting. No incorrect badge award or personal-best result was observed in this pass.

Interrupted endings, simultaneous deaths, host changes during initialization, and restoration around campfire transitions still need end-to-end tests. The methods describe their local responsibilities, but those responsibilities alone do not establish that every combined transition succeeds. Summit rescue timing and Nadir routing also depend on authored scene events beyond the outcome checks inspected here.
