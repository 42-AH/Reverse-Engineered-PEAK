# Saves and reconnection

A reconnect record and a campfire save both use player-state machinery, but they have different lifetimes. One remains part of an active session while the other serves as a snapshot. The distinction is straightforward in principle, though the save code makes it harder to maintain cleanly once edits begin.

A returning scout needs more than just a spawn position. Inventory, conditions, death state, and the scout report must all match the ongoing run. Resuming a saved run additionally requires the selected island, ascent, camp progress, world time, and retained item state. `ReconnectHandler` tracks participant records during play, while `Peak.Quicksave` packages run progress and player records for storage. Session appearance data flows through `PersistentPlayerDataService`, though its name does not mean that service writes to disk.

## Reconnect records

`ReconnectHandler` creates dictionaries for player state and achievement progress, then registers for room callbacks. Its `Update` method waits for a room and local character before finding players ready for registration. `RegisterPlayer` creates an initial `ReconnectData` record when needed and subscribes to changes in character state, affliction, warp, and inventory. These events refresh records alongside the periodic refresh that `Update` performs.

`ReconnectData.CreateFromCharacter` captures the character's current state by allocating a new status array, copying eligible conditions, and recording inventory, equipment selection, map progress, and scout-report data. It skips snapshots taken before initialization or during certain unstable movement and position states. `ReconnectHandler.UpdateReconnectData` only replaces the retained record when the new data passes validation. A rejected transient sample leaves the previous valid record in place.

Using reconnect data for checkpoint player state avoids maintaining a separate description of inventory and character conditions. Keeping the last valid sample when a new one is temporarily unsuitable also serves as a deliberate recovery strategy.

Group refreshes stop working once any registration goes missing.

`ReconnectHandler.UpdateForAllCharacters` iterates through current players and attempts to register any actor not in `_observedActors` if it is ready. Otherwise it logs the failure and breaks the loop in either case, leaving later players unrefreshed.

The caller turns this into a persistence problem rather than a mere naming issue. `Quicksave.SaveNow` immediately runs `SaveData.Create` after that refresh, so later players end up contributing older records. An actor still absent from the retained dictionary does not appear in the enumeration at all. `SaveData.Create` logs invalid records and continues with valid ones, never treating an incomplete refresh as a failed checkpoint.

The refresh loop has the wrong scope for failure. One unregistered player should delay that player's snapshot, not skip unrelated players further along in the enumeration. The code should continue past the missing registration and return an explicit completeness result so `SaveNow` can decide whether to commit. Letting a single incomplete participant silently affect the freshness of everyone after it makes for poor checkpoint logic.

In the current implementation, a single missing registration changes how fresh all subsequent records are, making enumeration order part of save quality.

## Restoring a character

Reconnection and checkpoint loading both converge on character restoration:

- `CharacterSpawner` retrieves a returning player's record via `ReconnectHandler.TryGetReconnectData` when spawning on the island.
- The reconnect spawn path determines whether the returning scout is alive, dead, or eligible for relevant revival handling.
- `CharacterSpawner.ApplyAndPushReconnectData` restores conditions and inventory, updates the scout report, and either selects equipment or reapplies death state.
- A continued host gets `Quicksave.SavedHostState` after `PopulateMapAndPlayerStates` and then applies the same restoration machinery.

The affliction chapter discusses how state restoration differs from standard status modification. This chapter focuses on the records that feed restoration rather than covering that material again.

## Writing the checkpoint

`Campfire.Light_Rpc` advances the segment when needed, calls `Quicksave.SaveNow`, then shows quicksave UI feedback for disk saves. `SaveNow` asks the reconnect handler to refresh all characters, builds `SaveData`, serializes it, and writes the save file. `SaveData.Create` goes through retained player IDs and filters out records that cannot provide valid reconnect data and achievement progress.

This reuse demands explicit snapshot ownership. A record used to restore a returning player is still live session state, even when a save operation temporarily borrows it. The current code copies a struct containing references and then edits one of its arrays, so the apparent value boundary fails to isolate the saved state.

A snapshot that changes its source has overlooked something fundamental about assignment.

`Quicksave.PlayerRunData.TryCreateFromReconnectData` gets a `ReconnectData` struct from `ReconnectHandler.TryGetReconnectData` and applies the checkpoint's campfire adjustments before building the serialized player record. The extra-stamina field is a scalar in the copied struct, but status values live in `currentStatuses`, an array. The local adjustment function writes directly into that array.

`ReconnectHandler.GetReconnectData` returns the stored struct without cloning its array. Copying the struct therefore keeps the reference to the stored status array. The checkpoint adjustment modifies the reconnect cache along with the value being serialized, whereas assigning the copied scalar does not change the scalar in the stored record. The cache ends up with mixed pre-adjustment scalar state and post-adjustment status state.

The next reconnect request and any later save that reuses the retained record before a valid refresh replaces it will see these changes. `ReconnectData.CreateFromCharacter` makes a fresh array when accepting a replacement, but `UpdateReconnectData` deliberately keeps the old record when the new sample fails validation. How long the unintended mutation stays available depends on snapshot freshness.

The save path breaks snapshot ownership. Copying a struct does not copy the mutable arrays it holds, and a readonly field would not fix that either. Clone arrays before applying save-specific adjustments, or create a separate representation instead. A save operation that changes the reconnect state it reads overstepped its responsibility. Calling the result a snapshot does not make it one.

`RunProgress.Create` records the active scene and run progress while collecting tracked item-spawn history, opened base-camp luggage, and loose items near the campfire. This amounts to selective world persistence rather than serializing every live object. `RunManager` uses matching spawn history during setup and restores saved base-camp items. `Quicksave.FinalizeRunSetup` restores the run clock and day/night values, then clears the temporary loaded-data state.

`Quicksave.SaveNow` serializes the checkpoint and writes directly to the save's final filename using `File.WriteAllText`, with no temporary-file commit or backup of the previous checkpoint. Its exception handler logs and rethrows without restoring bytes already overwritten by a failed write.

This matters because `Exists` checks file presence while `LoadSaveData` tries to deserialize the contents. A file left incomplete by an interrupted write can still exist but then fail to load. The same single destination serves as both the previously usable save and the in-progress replacement.

`Campfire.Light_Rpc` displays quicksave feedback only after `SaveNow` returns, so this is not claiming success appears before a write failure throws. The actual problem is the missing retained backup when the replacement fails after the file has already been altered.

The file commit strategy falls short for a checkpoint. You should validate the new data, write it separately, then perform a platform-appropriate atomic replacement while keeping the last completed file until the new one is committed. The current path leaves the previous checkpoint vulnerable to a failed replacement without that recovery mechanism. Logging the exception afterward documents the loss, but it does not prevent the save from being lost.

## Loading and session state

Loading starts in `MainMenuMainPage.SetUpContinueButton`. It passes a callback to `Quicksave.TryLoadSave` and checks the result against `Quicksave.Exists` before enabling the continue button. The load path reads the save, verifies its schema version, and requests the map bundle for the saved scene. Spawn-history compatibility runs a separate application-version check in `SpawnHistoryIsValid`.

When `Quicksave.TryLoadSave` runs, the callback from `LoadSaveData` checks the schema version. If the versions do not match, it destroys the save and calls the original function with a failure result. There is no return statement there. The method keeps going into map-bundle preparation and then calls the same function again, this time with the earlier load result.

That earlier result comes from reading and deserializing the file, not from the compatibility check. A save that parses successfully but fails the compatibility check can therefore trigger both a failure callback and a success callback. Because `DestroySaveData` also clears `_loadedData`, the bundle preparation that follows tries to work from an empty state and catches the resulting error.

The actual menu code limits one visible outcome. `MainMenuMainPage.SetUpContinueButton` also checks whether the file exists, so a second success result does not automatically make the continue button appear after deletion. The menu compensates for the loader returning inconsistent results. Its UI state and navigation callback still run twice for a single attempt.

The load method also fails to return when the read fails. Instead, it keeps running with the shared loaded-data state and version logic. This combines file reading, compatibility rejection, state clearing, and caller notification without a single clear endpoint.

The load callback breaks the completion contract. A failed check should end the attempt. The caller should receive one result at the end once everything is ready. Instead, callers have to decide what success means for themselves. `MainMenuMainPage` already wraps the load callback with an existence check. That extra check shows the load callback cannot be trusted, not that the loader's behavior is acceptable.

`PopulateMapAndPlayerStates` handles offline and online saves differently. Offline continuation uses the single player record. Online continuation finds the host's saved record and creates reconnect records for the other saved participants. `LoadSavedGameScene` resets ascent and run settings before selecting the appropriate scene-loading path.

`PersistentPlayerDataService` works differently. It stores player data by actor number, broadcasts updates, and clears the dictionaries when `ClearSessionData` runs. It also passes retained data when characters register. Appearance persistence and unlock storage are discussed in the progression chapter. This service's cache should not be confused with the campfire save.

## Reading and changing this code

Treat reconnect records as shared state even though `ReconnectData` is a struct. Some fields are mutable reference objects, and copying the outer value does not create an independent copy. State inspection and state modification follow different ownership rules.

Keep save-schema compatibility separate from world-content compatibility. `TryLoadSave` and `SpawnHistoryIsValid` answer different questions, and the retained item history depends on the world where it gets applied. The actor-indexed appearance service has its own lifetime. Modifying one representation means following its callers instead of assuming everything called persistent belongs to the same save.

## Open questions

The source findings do not establish that a particular player has lost a checkpoint or returned with incorrect conditions. Registration timing, accepted refreshes, and the precise moment a participant returns determine which incomplete or mutated record is consumed. These paths have not been exercised in the official Steam build during this analysis.

Cloud synchronization and operating-system write behavior have not been audited. The direct-write finding concerns the application's local commit strategy, not a claim about guarantees supplied by a particular filesystem or platform backup service.

The compatibility checks do not by themselves establish which historical releases can safely share item-spawn history. That requires content and version-history evidence beyond this source snapshot. The empty `_2_CODE.Network.Reconnector` class is not evidence that reconnection is absent; the functioning paths are in `ReconnectHandler` and `CharacterSpawner`.
