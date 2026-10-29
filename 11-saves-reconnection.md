# Saves and reconnection

A reconnect record and a campfire save share player-state machinery. They do not share a lifetime. One is still part of a running session; the other is supposed to be a snapshot. The distinction is easy enough to describe, and rather less carefully maintained when the save code starts editing the data.

A returning scout needs more than a spawn position. Inventory, conditions, death state, and the scout report must agree with the ongoing run. Continuing a saved run also needs the selected island, ascent, camp progress, world time, and retained item state. `ReconnectHandler` maintains participant records during play, while `Peak.Quicksave` packages run progress and player records for later use. Session appearance data travels through `PersistentPlayerDataService`; its name does not mean that service writes a disk save.

## Reconnect records

`ReconnectHandler` creates dictionaries for player state and achievement progress and registers for room callbacks. Its `Update` waits for a room and local character, then finds players ready for registration. `RegisterPlayer` creates an initial `ReconnectData` record when necessary and subscribes to character-state, affliction, warp, and inventory changes. Those events refresh records in addition to the periodic refresh in `Update`.

`ReconnectData.CreateFromCharacter` gathers the character's current state. It allocates a new status array, copies the eligible conditions, and captures inventory, equipment selection, map progress, and scout-report data. It rejects snapshots taken before initialization or during particular unstable movement and position states. `ReconnectHandler.UpdateReconnectData` replaces the retained record only when the new data is valid. A rejected transient sample therefore leaves the earlier valid record available.

Reusing reconnect data for checkpoint player state avoids a second, independently maintained description of inventory and character conditions. Retaining the last valid sample when a new one is transiently unsuitable is also a deliberate recovery policy.

Refreshing everyone stops being a group operation at the first missing registration.

`ReconnectHandler.UpdateForAllCharacters` iterates the current players. When it encounters an actor not present in `_observedActors`, it tries to register that actor if ready, otherwise logs the failure. It then breaks out of the loop in either case. Players later in the enumeration are not refreshed.

The caller makes this a persistence defect rather than merely an inaccurate method name. `Quicksave.SaveNow` immediately follows that refresh with `SaveData.Create`. Later players can consequently contribute older records, while an actor that is still absent from the retained dictionary is not included in the dictionary enumeration at all. `SaveData.Create` logs invalid records and proceeds with the remaining ones; it does not turn an incomplete refresh into a failed checkpoint.

The refresh loop has the wrong failure scope. One unregistered player is a reason to defer that player's snapshot, not abandon unrelated players that appear later in the enumeration. Continue past the missing registration and return an explicit completeness result so `SaveNow` can decide whether to commit. Letting one incomplete participant silently determine the freshness of everyone after it is poor checkpoint logic.

Instead, a single missing registration changes the freshness of records that follow it. Enumeration order becomes part of save quality.

## Restoring a character

Reconnection and checkpoint loading converge on character restoration:

- `CharacterSpawner` looks for a returning player's record through `ReconnectHandler.TryGetReconnectData` when spawning on the island.
- The reconnect spawn path determines whether the returning scout is alive, dead, or eligible for the relevant revival handling.
- `CharacterSpawner.ApplyAndPushReconnectData` restores conditions and inventory, catches up the scout report, and selects equipment or reapplies death state.
- A continued host obtains `Quicksave.SavedHostState` after `PopulateMapAndPlayerStates`, then uses the same application machinery.

The affliction chapter covers the difference between state restoration and ordinary status modification. This chapter follows the records feeding that restoration rather than repeating that issue.

## Writing the checkpoint

For disk saves, `Campfire.Light_Rpc` advances the segment when appropriate, calls `Quicksave.SaveNow`, and then invokes the quicksave UI feedback. `SaveNow` asks the reconnect handler to refresh all characters, constructs `SaveData`, serializes it, and writes the save file. `SaveData.Create` enumerates retained player IDs and excludes records that cannot supply valid reconnect data and achievement progress.

That reuse requires snapshot ownership to be explicit. A record used for a returning player is still live session state, even when a save operation borrows it. The current code copies a struct containing references and then edits one of its arrays. The apparent value boundary does not isolate the saved state.

A snapshot that edits its source has missed a fairly basic part of the assignment.

`Quicksave.PlayerRunData.TryCreateFromReconnectData` obtains a `ReconnectData` struct from `ReconnectHandler.TryGetReconnectData`. It applies the checkpoint's campfire adjustments before constructing the serialized player record. The extra-stamina field is a scalar in the copied struct, but the status values live in `currentStatuses`, an array. The local adjustment function writes into that array directly.

`ReconnectHandler.GetReconnectData` returns the stored struct without cloning its array. Copying the struct therefore preserves the reference to the stored status array. The checkpoint adjustment changes the reconnect cache as well as the value being serialized. By contrast, assigning the copied scalar does not change the scalar in the stored record. The cache is left with a mixture of pre-adjustment scalar state and post-adjustment status state.

The affected consumers are the next reconnect request and any subsequent save that reuses the retained record before a valid refresh replaces it. `ReconnectData.CreateFromCharacter` creates a fresh array when a replacement is accepted, but `UpdateReconnectData` deliberately retains the old record when the new sample is invalid. Snapshot freshness determines how long the unintended mutation remains available.

The save path violates snapshot ownership. Copying a struct does not copy the mutable arrays it references, and a readonly field would not repair that either. Clone the arrays before making save-specific adjustments, or construct a separate representation. A save operation that changes the reconnect state it reads is doing damage outside its responsibility. Calling the result a snapshot does not make it one.

`RunProgress.Create` records the active scene and run progress. Its construction also collects tracked item-spawn history, opened base-camp luggage, and loose items near the campfire. This is selected world persistence rather than a serialization of every live object. `RunManager` uses compatible spawn history during setup and restores saved base-camp items. `Quicksave.FinalizeRunSetup` restores the run clock and day/night values, then clears the temporary loaded-data state.

`Quicksave.SaveNow` serializes the assembled checkpoint and writes directly to the save's final filename with `File.WriteAllText`. There is no temporary-file commit or preservation of the previous completed checkpoint in this method. Its exception handler logs and rethrows; it cannot restore bytes already replaced by a failed write.

The distinction matters because `Exists` is a file-existence check, while `LoadSaveData` attempts to deserialize the contents. A file left incomplete by an interrupted write can still exist and then fail to load. The same single destination is both the previously usable save and the in-progress replacement.

`Campfire.Light_Rpc` shows quicksave feedback only after `SaveNow` returns, so this is not a claim that its success notification precedes a thrown write failure. The problem is the absence of a retained last-good checkpoint when the replacement fails after altering the file.

The file commit strategy is inadequate for a checkpoint. Validate the new data, write it separately, then use a platform-appropriate atomic replacement while retaining the last completed file until the new one is committed. The present path exposes the previous checkpoint to a failed replacement without that recovery arrangement. Logging the exception afterward explains the loss; it does not protect the save.

## Loading and session state

Loading begins in `MainMenuMainPage.SetUpContinueButton`. It passes a callback to `Quicksave.TryLoadSave` and combines the result with `Quicksave.Exists` before enabling the continue button. The load path reads the save, checks its schema version, and requests the map bundle associated with the saved scene. Spawn-history compatibility has a separate application-version check in `SpawnHistoryIsValid`.

Inside `Quicksave.TryLoadSave`, the callback from `LoadSaveData` checks the schema version. On incompatibility it destroys the save and invokes the caller with failure. There is no return after that branch. Execution continues into map-bundle preparation and then invokes the same caller again with the earlier load result.

That earlier result describes reading and deserializing the file, not passing the compatibility check. A successfully parsed but incompatible save can therefore cause a failure callback followed by a success callback. Because `DestroySaveData` also resets `_loadedData`, the intervening bundle preparation attempts to work from cleared state and handles the resulting failure through its catch block.

The actual menu consumer limits one visible consequence. `MainMenuMainPage.SetUpContinueButton` additionally checks whether the file exists, so a second success result does not by itself establish that the continue button becomes available after deletion. The menu compensates for the loader's inconsistent result. Its UI state and navigation callback still run twice for one attempt.

The load method also does not begin its callback by returning on a failed read. It continues with the shared loaded-data state and version logic. This combines reading, compatibility rejection, state clearing, and caller notification without one terminal outcome.

The load callback has a broken completion contract. A failed check must end the attempt; the caller should receive one final result after all prerequisites are satisfied. Instead, consumers have to reinterpret success themselves. `MainMenuMainPage` already adds an existence check around the load callback. That defensive check is evidence of a boundary that callers cannot trust, not a reason to leave the loader's behavior alone.

`PopulateMapAndPlayerStates` distinguishes offline and online saves. Offline continuation uses the single player record. Online continuation identifies the host's saved record and populates reconnect records for the remaining saved participants. `LoadSavedGameScene` restores ascent and run settings before choosing the corresponding scene-loading path.

`PersistentPlayerDataService` has a different scope. It caches player data by actor number, distributes changes, and clears the dictionaries in `ClearSessionData`. It also forwards retained data when characters register. Appearance persistence and unlock storage are covered in the progression chapter; this service's cache should not be mistaken for the campfire save.

## Reading and changing this code

Treat reconnect records as shared state even though `ReconnectData` is a struct. Some members are mutable reference objects, and a copied outer value does not imply an independent snapshot. State inspection and state transformation need different ownership rules.

Separate save-schema compatibility from world-content compatibility. `TryLoadSave` and `SpawnHistoryIsValid` answer different questions, and retained item history depends on the world it will be applied to. The actor-indexed appearance service has another lifetime again. Changing one representation requires tracing its consumers rather than assuming every object called persistent belongs to the same save.

## Open questions

The source findings do not establish that a particular player has lost a checkpoint or returned with incorrect conditions. Registration timing, accepted refreshes, and the precise moment a participant returns determine which incomplete or mutated record is consumed. These paths have not been exercised in the official Steam build during this analysis.

Cloud synchronization and operating-system write behavior have not been audited. The direct-write finding concerns the application's local commit strategy, not a claim about guarantees supplied by a particular filesystem or platform backup service.

The compatibility checks do not by themselves establish which historical releases can safely share item-spawn history. That requires content and version-history evidence beyond this source snapshot. The empty `_2_CODE.Network.Reconnector` class is not evidence that reconnection is absent; the functioning paths are in `ReconnectHandler` and `CharacterSpawner`.
