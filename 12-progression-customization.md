# Achievements, badges, and appearance

A badge flows through run progress, platform statistics, menu presentation, and the scout's appearance. Each system has its own notion of readiness. By the time the passport applies a cosmetic choice, the same category ordering and fallback rules show up repeatedly across multiple places. A single incorrect variable is enough to demonstrate the cost of that duplication.

Achievements track actions and completed challenges, while badges show that progress in menus and on the scout. Ascent progress unlocks further rewards. The passport picks appearance from available options, and that selection gets applied to the character and shared with other players. Reading guidebook pages builds persistent reading progress. These systems rely on platform-backed state along with data that only exists during the current run; a new badge, a new cosmetic choice, and a saved run involve different operations.

## Progress and platform readiness

`AchievementManager` holds requirement descriptions for run-based and statistic-based achievements. `SubscribeToEvents` hooks tests into gameplay events: item consumption, luggage opening, local status changes, death, and run completion. For instance, `TestItemConsumed` sorts the item and updates the relevant run progress. `SetRunBasedInt` and `SetRunBasedFloat` modify that progress, verify the linked requirements, and signal `Player.localPlayer.OnAchievementProgressChanged`. When reconnecting, a separate copy of this run progress persists, as covered in the saves chapter.

The platform initialization path works differently. `GetUserStatsRoutine` waits for the platform and requests statistics until `GotStats` turns true. `OnUserStatsReceived` receives a successful response, sets up guidebook progress, and runs the one-time ascent check. `GotStats` also requires that the platform stays initialized. An achievement-manager object being present does not mean statistics are available.

`ThrowAchievement` verifies eligibility under current run settings, looks at existing achievement state, updates the run's newly earned collection if needed, asks the platform to award it, and fires `GlobalEvents.OnAchievementThrown`. `StoreUserStatsRoutine` separately invokes the platform's store operation. `CheckNewAchievements` compares current unlock state against the run's previous unlocked set and fires the same event for newly found awards. Statistic-linked achievements get evaluated through their linked statistics rather than purely through a platform achievement flag.

## Badges

Several consumers work with badge presentation:

- `BadgeManager.InitBadges` waits for statistics and creates its `BadgeUI` children from badge definitions.
- `BadgeUI.Init` applies the icon and locked appearance. The end-screen form can display the badge independently of its normal locked look.
- `BadgeManager.selectedBadge` picks the name and description shown when hovering or using controller selection, including special handling for secret badges.
- `CharacterData.SetBadgeStatus` waits for statistics, builds the local badge-status array, and shares that presentation state.
- `BadgeUnlocker.SetBadges` converts the character's badge status into a texture used by the sash material.

Separating the badge definition from the badge renderer means menus and character presentation can share the same achievement identity. It also matters to distinguish between a selected appearance and an unlock requirement: getting another scout's appearance should not demand that the local player has unlocked that scout's rewards.

The texture creation comes with a cost. Who owns its replacement is apparently someone else's concern.

`BadgeUnlocker.SetBadges` makes a new `Texture2D` for each nonempty badge-status update, fills it from the status array, and assigns it to the sash material. The generated texture stays a local variable. The method keeps no owned reference for reuse and does not destroy the old generated texture when replacing the material assignment.

`CharacterData.SyncBadgeStatus` calls `BadgeUnlockVisual`, which calls `SetBadges`. Join-state application triggers the same visual refresh. `PeakHandler` also relies on `SetBadges` for its character presentation. The allocation happens outside the separate debug branch in `BadgeUnlocker.Update`.

Swapping the material's texture reference does not define a lifecycle for the old generated Unity object. Each badge refresh allocates new objects with no explicit release from this owner. The result is unnecessary allocation and deferred cleanup responsibility, even though the status array could often be represented by updating the existing small texture instead.

The code also creates the texture before it checks if the supplied sash renderer is null. In that case it has allocated and filled a texture that never gets assigned anywhere. Validation should come first.

`BadgeUnlocker` allocates resources without managing their replacement lifecycle. Updating one owned texture in place would cut out the repeated allocations; replacing it means explicitly releasing the previous generated texture. Neither of those duties belongs to eventual scene cleanup. The null-renderer case is worse: it allocates and fills a texture before confirming there is anywhere to use it. Validate first. This amounts to careless resource handling in a routine visual refresh.

## Passport choices and appearance

The passport flow starts with `PassportButton.SetButton` and `Click`, which check `CustomizationOption.IsLocked`. That property evaluates the option's requirement category, covering ascent and achievement requirements. `PassportManager.SetOption` routes the selected category to `CharacterCustomization`. These checks control the selection interface, separate from the renderer's job of applying an already chosen appearance.

The matching character setters modify `CharacterCustomizationData`, route it through `PersistentPlayerDataService`, and send the chosen index as a platform statistic. `CharacterCustomization.OnPlayerDataChange` applies the appearance: skin colors, outfit mesh and materials, facial textures, accessories, and sash or medal display. It also refreshes local-body hiding after updating the rig's visible elements.

During initial local setup, `GetCosmeticsFromSteamRoutine` waits for statistics and then calls `TryGetCosmeticsFromSteam`. That method pulls the saved choices by category and applies a fallback when a value does not exist. The chosen values then go through the same setters as passport selections.

The problem stems from using raw category indices across platform persistence, session data, and renderer lookup. Every reader needs to maintain the same catalog ordering and fallback rules. Those rules appear in the loader, setters, index getters, and `CharacterCustomizationData.CorrectValues`. The repetition has already introduced a cross-field error in the load path.

The accessory choice gets its own lookup, then must pass the mouth's test.

In `CharacterCustomization.TryGetCosmeticsFromSteam`, the accessory statistic is read into its own result, but the corresponding unavailable-value check tests the previous mouth result instead. The accessory lookup's success therefore combines with a condition about a different category.

This shifts which branch restores the accessory. An unavailable mouth result sends an otherwise good accessory choice to the random fallback. Meanwhile, the condition rejecting an unavailable accessory value ignores that accessory value. The category-local pattern used for other cosmetic fields gets broken when a copied variable reference substitutes.

`SetCharacterAccessory` writes the selected index to session appearance data and sends it back as the saved accessory statistic. `PersistentPlayerDataService` notifies `OnPlayerDataChange`, which resolves the accessory and sets its texture and related display properties. The error affects both what displays and what gets submitted for persistence later; it is not just an unused intermediate check.

The defensive bounds checks do not consistently fix every invalid value. `SetCharacterAccessory` and `GetAccessoryIndex` correct values above the catalog's upper bound but fail to check the lower bound. `CharacterCustomizationData.CorrectValues` follows the same pattern, only validating the upper end. As a result, values outside the acceptable lower range remain invalid even after passing through these correction methods.

`TryGetCosmeticsFromSteam` has a straightforward field-check bug, and the surrounding code makes such mistakes easy to make. Nearly identical read blocks keep unrelated temporary values in scope, while the correction methods only validate one side of the acceptable range. Combine the category's value, bounds, and fallback into a single read-and-validate step. The current duplication provides no added flexibility; instead it creates a mouth-dependent accessory decision and lets invalid values leak downstream.

## Guidebook progress

`Guidebook` progress lives in `Peak.GuidebookData`. `Guidebook` and `Action_SpawnGuidebookPage` report seen pages via `AchievementManager.TriggerSeenGuidebookPage`. This entry point updates the local page record and syncs its persistent progress. When platform statistics arrive, `InitGuidebookData` creates the record and asks it to load any previously seen pages.

`GuidebookData.LoadGuidebookPagesSeen` calls `AchievementManager.GetSteamStatInt` to fetch the saved page record but ignores the boolean return value. After the call, it merges the returned value into its local record and sets `HasInitialized` to true. The guard condition is `AchievementManager.Initialized`, which gets set during run-value initialization rather than the manager's actual `GotStats` state.

The statistic wrapper itself reports failure when the platform is offline. Other parts of the code can tell the difference between a failed read and a successfully retrieved value. The guidebook code discards this distinction and treats its record as loaded anyway.

`PushGuidebookPagesSeen` checks `HasInitialized` to decide whether to submit the current page record and reading progress. Since `IsPageSeen` and `CountPagesSeen` also rely on the same record, a failed load is not just a diagnostic flag. It becomes the input for page visibility and progress calculations. What gets returned on failure depends on the platform implementation; the real problem is treating that value as if the read succeeded.

There is also an ownership issue with early page observations. `AchievementManager.TriggerSeenGuidebookPage` can update the current `guidebookData`, but `OnUserStatsReceived` calls `InitGuidebookData`, which replaces the object before loading. `GuidebookData.LoadGuidebookPagesSeen` intentionally merges observations already in its own instance, but that merge cannot recover observations held by the old instance that just got discarded. The code defines a merge policy then creates a fresh object at exactly the point where the policy is needed.

`Guidebook` initialization claims success without establishing it. A rejected read should leave the record in a pending state, and observations collected during loading should survive until they can be merged. Replacing the record once statistics arrive discards the early observations the merge was designed to protect. Either reuse the existing record or keep a pending set of observations. The current approach undermines both readiness and preservation while displaying a reassuring `HasInitialized` flag.

## Reading and changing this code

Check the achievement's actual source of truth before relying on its displayed badge. Statistic-linked achievements, explicit platform awards, and the run's newly earned list are separate sources. The sash's badge-status array is just a display copy; changing how it looks does not create progression.

Cosmetic catalog ordering is part of the save contract. Reordering the catalog and swapping a visual have different effects because saved and shared choices are stored as indices. Keep selection eligibility separate from applying received appearances, and review the existing correction code rather than assuming the method names promise validated data.

## Open questions

No platform award, rejected statistic request, or cosmetic restoration failure was exercised here. The accessory finding identifies a wrong-field comparison; its player-visible outcome depends on which values the platform supplies. The guidebook findings require observations before successful statistics initialization to expose their early-loading consequences.

Texture allocation is source-visible, but its memory impact over a session and the timing of unused-resource cleanup have not been measured. The serialized badge atlas and material behavior also need separate inspection before attaching a specific visual defect to the sash.

The chapter does not establish that a submitted award has been durably stored when the local event fires. Platform acknowledgement and retry behavior require tracing the platform implementation. Unlock requirements and numeric thresholds have deliberately not been reproduced as a catalog or an unlock procedure.
