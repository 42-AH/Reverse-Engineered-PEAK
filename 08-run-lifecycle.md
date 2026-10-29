# Airport, ascent, and ending

Starting a run sounds like an event. Here it is a handoff between boot services, the airport, scene loading, a run manager, and character state. Ending one brings in the results screen as well. There are plenty of places to say a run has started or stopped; getting them to mean the same thing is the less tidy part.

The airport lets the player choose an ascent or custom rules and initiate departure. Once the island and character are ready, `RunManager` initializes the run, coordinates its identity and clock, and supports continuation from saved progress. Campfires combine a local interaction with group participation, progression, and saving.

Death changes the character's state and possessions, while revival restores a playable character and updates recovery records. Ending the run classifies individual outcomes before choosing either a cutscene or the results screen. Someone else succeeding is distinct from the local character succeeding, and the results interface has separate presentations for those cases.

## Boot and departure

`GameBooter.Initialize` registers application and scene callbacks. `AutoBoot` creates the game object and initializes `GameHandler` and `UIInputHandler` on the first scene notification. Invitation readiness is handled separately through the networking interfaces described in chapter 09.

Departure follows this chain:

- `AirportCheckInKiosk.Interact_CastFinished` opens the boarding pass and supplies its kiosk reference.
- `BoardingPass.StartGame` resolves its selected ascent in relation to custom-run mode and calls the kiosk.
- `AirportCheckInKiosk.StartGame` submits the selected run settings for departure when loading is not already active.
- `LoadIslandMaster` selects the destination; `BeginIslandLoadRPC` applies the ascent and run settings before starting the loading process.

Chapter 05 covers the scene catalogue and bundle prerequisite. Here the important boundary is that selecting a destination is not equivalent to starting the run clock.

## Starting a run

`RunManager.Start` initializes map state and waits for room membership, a local character, and completion of loading. An airport character ends this initialization path without starting a run. An island character reaches `StartRun`. The non-host then attempts to obtain the run identity; the host performs its own delayed setup, publishes a new identity for a fresh run, and starts time synchronization. The saved-run branch restores spawn history and finalizes saved setup instead.

Looking up a run identifier is easy. Making sure a run has one turns out to be a separate chore.

`RunManager.GetRunId` calls `NetCode.Matchmaking.TryGetLobbyData` but ignores its success result. It immediately passes the returned string to `Guid.Parse`, then marks the identity as obtained. A missing or malformed value is not represented as a pending initialization state; parsing throws before the success flag is set.

The caller makes the timing issue concrete. In `RunManager.Start`, a non-host calls `GetRunId` after its own loading prerequisites complete. The host's fresh-run identity publication occurs later in its setup path, after an explicit wait. Local scene readiness and availability of the published identity are not the same event, but the client path treats its first as sufficient for the second.

There is also a separate `RPC_ReceiveRunId` path that can supply the identity. That path does not validate the earlier assumption; it means there are two possible sources of identity readiness whose ordering must be coordinated. If the message has already arrived, `_hasRunId` avoids the lookup. If it has not, the lookup still assumes that a valid value is immediately available.

The immediate cost is an exception in initialization when the prerequisite is absent or invalid. It also leaves identity-dependent consumers with a readiness contract they must infer. `Quicksave` obtains the identity from `RunManager` when constructing run progress, while the diagnostic display reads the same property. Neither property access itself expresses whether identity establishment succeeded.

Run identity readiness is not modeled properly. The code needs pending, valid, and failed outcomes, with message receipt and property lookup both going through checked validation. Until that transition succeeds, consumers do not have an identity to use. Adding a longer delay would only disguise the missing coordination. The dependency is real; the implementation treats it like a timing assumption.

Validation belongs at both entry points. Parsing the message path and the lookup path through one checked operation would prevent the same state from having two different assumptions about readiness and input validity.

`StartRun` subscribes to room population and owner changes and schedules an end-condition check. `RunManager.Update` advances the clock while active, periodically synchronizes it on the host, and performs scheduled checks. `OnCharacterRegistered` supplies time state to an arriving character. The registered character and the run's shared identity therefore have different readiness paths.

## Campfires, death, and revival

Campfire participation is explicit. `EveryoneInRange` compares nearby characters against living participants. `Light_Rpc` updates the fire, then either ends a miniature run at its ending campfire or advances the map through `MapHandler.GoToSegment`. The progression path also requests a quicksave. Morale effects and nearby-character recovery are additional campfire responsibilities; cooking uses the item machinery described in chapter 04 rather than defining the checkpoint transition itself.

Checkpoint progression similarly combines several outcomes without treating them as interchangeable. A lit fire has visual state; an advancing fire changes segment state; a saved checkpoint persists progress. The saved-run setup path must restore those relationships rather than merely positioning the player near a campfire.

Death reaches `Character.RPCA_Die`, which unequips, sets dead state, records the event, drops items, handles the death representation, and moves the character before checking the run-ending condition. `CheckEndGame` performs the master-side all-dead check. Ghost presentation is separate: `PlayerGhost.RPCA_InitGhost` associates an owner and target and restores the owner's visual customization, with special handling for the owner's own view.

Revival follows `RPCA_ReviveAtPosition` into `ReviveCharacter`. The latter resets death and unconsciousness state, clears relevant effects, and optionally applies post-revival status. The caller then moves the character, updates progress and statistics, and tells `ReconnectHandler.UpdateFromRevive` about the new position. That recovery update matters because revival is both a current gameplay change and a new state from which later reconnection must resume.

## Ending the run

`Character.RPCEndGame` performs outcome processing. It marks game over, clears resumable save data, classifies characters through `CheckWinCondition`, and updates their statistics. Depending on the result and run mode, it opens `EndScreen` directly or starts the appropriate `PeakHandler` ending sequence. It then raises the run-ended event and calls `RunManager.EndGame`.

The ordinary summit outcome considers a living character's summit or helicopter-rope state; miniature-run and Nadir paths use their own branch. `PeakHandler.EndCutscene` and `EndCutsceneFinal` prepare different presentations before reaching the results interface. `EndScreen` displays the elapsed time, plays the timeline, and distinguishes personal success, another player's success, and failure. The ending is a coordination path across managers, not one scene-load call.

The character layer owns the final classification, while `RunManager` owns shared timing and identity. This lets character-specific outcomes feed one results sequence, but it requires the clock to become final before result consumers use it. The results interface also triggers time-achievement evaluation, so elapsed time is gameplay data rather than decorative text.

A stopped clock should be the uncomplicated part of this arrangement.

`RunManager.TimeSinceRunStarted` adds the current frame's delta when the clock has not been updated during that frame. This compensates for access before `Update` while the clock is running. The getter does not check `timerActive`, however. Once the clock stops, subsequent frames still meet the not-updated-this-frame condition, so the getter returns the stored final time plus the caller's current frame delta.

This is not an indefinitely advancing stopped timer: the extra delta is not written back to the stored time. It is a read-time offset that changes with the frame on which a consumer asks. That distinction limits the defect but does not remove it. A final elapsed time should not depend on the duration of the results screen's current frame.

Synchronization adds another offset. `RPC_SyncTime` adjusts incoming time by elapsed network time regardless of the accompanying active flag. Applying that adjustment to a running clock estimates its current value. Applying it to a stopped clock changes the final result based on when the stop message arrives. The method has enough information to distinguish those cases but uses the running-clock treatment for both.

`RunManager.EndGame` sends a stopped-clock update, and `EndScreen` later reads `TimeSinceRunStarted` for its time label. `AchievementManager.TestTimeAchievements` also reads that property, converts it to a whole-second value, and uses it for time-based results. `CharacterStats` uses the same clock when recording timeline entries. The final-time contract therefore reaches display, progression evaluation, and run history.

The problem is the absence of an immutable final elapsed value. A stopped clock is represented by a boolean beside a value whose getter and synchronization path continue performing live-time adjustments. Different consumers can read at different frames, and participants can receive the stop update at different times.

The clock's final-state contract is broken. Finalization should capture one elapsed value and publish it without live-clock or transmission-age adjustments. An inactive accessor should return that value. Results and time-achievement checks should not each receive an estimate of a run that has already stopped. Keeping running-clock compensation alive in the finished state adds uncertainty precisely where the code should be removing it.

## Reading and changing this code

Separate scene readiness from run readiness. A character can exist before identity setup has completed, and a checkpoint's visual state does not by itself describe saved progression. Changes to revival also need to account for statistics and reconnect records, not just health or position.

Treat final results as shared gameplay data. A presentation change should not introduce a new time estimate or classify victory independently of `CheckWinCondition`. Lower-level movement and status changes belong in their own systems, but their completion signals must remain consistent with the lifecycle consumers.

## Open questions

The identity failure is conditional on lookup returning an unavailable or invalid value before another identity path completes. That timing was traced in source but not reproduced in an official multiplayer session. The actual ordering of lobby data delivery and identity messages needs observation before estimating frequency or recovery behavior.

The clock offsets are established by the getter and synchronization logic. Their visible magnitude depends on frame duration, delivery timing, and result formatting. No incorrect badge award or personal-best result was observed in this pass.

Interrupted endings, simultaneous deaths, host changes during initialization, and restoration around campfire transitions still need end-to-end tests. The methods describe their local responsibilities, but those responsibilities alone do not establish that every combined transition succeeds. Summit rescue timing and Nadir routing also depend on authored scene events beyond the outcome checks inspected here.
