# Achievements, badges, and appearance

Following a badge from gameplay to the passport exposes several different records: run-local achievement progress, platform statistics, the badge display, and selected cosmetic indices. They are related but do not share one readiness or ownership model. The most concrete defects appear where those records meet: cosmetic restoration tests the wrong field, guidebook loading accepts an unsuccessful read, and badge rendering replaces generated textures without owning their cleanup.

## What it does

Achievements record actions and completed challenges, while badges display that progress on menus and the scout. Ascent progress contributes further rewards. The passport selects appearance from the available choices, and that selection is applied to the character and shared with other participants. Reading guidebook pages contributes persistent reading progress. These systems use platform-backed state as well as data that belongs only to the current run; a new badge, a new cosmetic choice, and a saved run are not the same operation.

## How it's built

`AchievementManager` owns requirement descriptions for run-based and statistic-based achievements. `SubscribeToEvents` attaches tests to gameplay events including item consumption, luggage opening, local status changes, death, and run completion. For example, `TestItemConsumed` classifies the item and updates the relevant run progress. `SetRunBasedInt` and `SetRunBasedFloat` update that progress, check the linked requirements, and notify `Player.localPlayer.OnAchievementProgressChanged`. Reconnection retains a separate representation of this run progress, as described in the saves chapter.

Platform initialization is another path. `GetUserStatsRoutine` waits for the platform and requests statistics until `GotStats` becomes true. `OnUserStatsReceived` accepts a successful response, initializes guidebook progress, and performs the one-time ascent check. `GotStats` also requires that the platform remains initialized. An achievement-manager object existing is not equivalent to statistics being available.

`ThrowAchievement` checks eligibility under the current run settings, consults existing achievement state, updates the run's newly earned collection where applicable, requests the platform award, and emits `GlobalEvents.OnAchievementThrown`. `StoreUserStatsRoutine` separately calls the platform's store operation. `CheckNewAchievements` compares current unlock state with the run's previously unlocked set and emits the same event for newly detected awards. Statistic-linked achievements are evaluated through their associated statistics rather than exclusively through a platform achievement flag.

The badge presentation has several consumers:

- `BadgeManager.InitBadges` waits for statistics and initializes its `BadgeUI` children from badge definitions.
- `BadgeUI.Init` applies the icon and locked appearance; the end-screen form can show the badge independently of its ordinary locked presentation.
- `BadgeManager.selectedBadge` chooses the name and description shown on hover or controller selection, including secret-badge treatment.
- `CharacterData.SetBadgeStatus` waits for statistics, creates the local badge-status array, and shares that presentation state.
- `BadgeUnlocker.SetBadges` turns the character's badge status into a texture used by the sash material.

The passport path begins with `PassportButton.SetButton` and `Click`, which consult `CustomizationOption.IsLocked`. That property evaluates the option's requirement category, including ascent and achievement requirements. `PassportManager.SetOption` dispatches the selected category to `CharacterCustomization`. These are permission checks for the selection interface, distinct from the renderer's task of applying an already selected appearance.

The corresponding character setters change `CharacterCustomizationData`, distribute it through `PersistentPlayerDataService`, and submit the chosen index as a platform statistic. `CharacterCustomization.OnPlayerDataChange` applies the appearance: skin colors, outfit mesh and materials, facial textures, accessories, and sash or medal presentation. It also refreshes local-body hiding after changing the rig's visible parts.

On initial local setup, `GetCosmeticsFromSteamRoutine` waits for statistics and then calls `TryGetCosmeticsFromSteam`. That method reads the saved choices by category and uses a fallback where a value is unavailable. The chosen values then pass through the same setters as passport selections.

Guidebook progress is represented by `Peak.GuidebookData`. `Guidebook` and `Action_SpawnGuidebookPage` report seen pages through `AchievementManager.TriggerSeenGuidebookPage`. That entry point updates the local page record and submits its persistent progress. `InitGuidebookData` creates the record when platform statistics arrive and asks it to load previously seen pages.

## Design decisions

The badge definition and badge renderer are separate, allowing menus and character presentation to use the same achievement identity. The distinction between a selected appearance and an unlock requirement is also necessary: receiving another scout's appearance should not require that the local player has earned that scout's rewards.

The costly choice is using raw category indices across platform persistence, session data, and renderer lookup. Every reader must preserve the same catalog ordering and fallback rules. Those rules are repeated in the loader, setters, index getters, and `CharacterCustomizationData.CorrectValues`; repetition has already produced a cross-field mistake in the load path.

## Where it's weak

### Accessory restoration tests the mouth result

In `CharacterCustomization.TryGetCosmeticsFromSteam`, the accessory statistic is read into its own result, but the accompanying unavailable-value check tests the preceding mouth result. The success of the accessory lookup is therefore combined with a condition about another category.

This changes which branch restores the accessory. An unavailable mouth result sends an otherwise valid accessory choice to the random fallback. Conversely, the condition used to reject the accessory's unavailable value does not inspect that accessory value. The category-local pattern used for neighboring cosmetic fields has been broken by a copied variable reference.

The consumer chain makes the consequence specific. `SetCharacterAccessory` writes the selected index into session appearance data and submits it back as the saved accessory statistic. `PersistentPlayerDataService` notifies `OnPlayerDataChange`, which resolves the accessory and assigns its texture and related presentation properties. The mistake affects both the displayed choice and the value subsequently submitted for persistence; it is not an unused intermediate comparison.

The defensive bounds checks do not consistently repair every unavailable value. `SetCharacterAccessory` and `GetAccessoryIndex` correct values above the catalog's upper bound but do not establish a lower bound. `CharacterCustomizationData.CorrectValues` repeats that upper-bound-only policy. A value outside the accepted lower range therefore is not made valid merely by passing through those named correction points.

A single category-specific read-and-validate operation would keep the retrieved value, fallback, and catalog bounds together. The current sequence repeats near-identical read blocks while carrying all their temporary results through one method. The wrong field is a direct consequence of that organization, and the downstream setters do not provide a complete validation boundary.

### Guidebook initialization ignores whether the statistic was read

`GuidebookData.LoadGuidebookPagesSeen` asks `AchievementManager.GetSteamStatInt` for the saved page record, but does not check its boolean result. After the call it combines the returned value with its local record and marks `HasInitialized` true. Its guard is `AchievementManager.Initialized`, which is set during run-value initialization, rather than the manager's distinct `GotStats` state.

The statistic wrapper itself reports failure when the platform is unavailable. Other callers can distinguish an unavailable read from a successfully retrieved value. The guidebook path discards that distinction and labels its record loaded anyway.

`PushGuidebookPagesSeen` uses `HasInitialized` to decide whether to submit the current page record and reading progress. `IsPageSeen` and `CountPagesSeen` also consume the same record, so an unsuccessful load is not confined to a diagnostic flag. It becomes the basis for page visibility and progress calculations. The exact returned value on failure depends on the platform path; the structural error is accepting that value as a completed read.

There is a second ownership problem around early page observations. `AchievementManager.TriggerSeenGuidebookPage` can update the current `guidebookData`, while `OnUserStatsReceived` calls `InitGuidebookData`, which replaces that object before loading. `GuidebookData.LoadGuidebookPagesSeen` deliberately merges observations already present in its own instance, but that merge cannot preserve observations held by the discarded instance. The code provides a merge policy and then creates a fresh object at the boundary where that policy is needed.

A failed read should leave initialization incomplete, and observations gathered while loading should remain in an owned pending set until a successful merge. Reusing the existing record at statistics readiness would retain those observations. The current arrangement gives object creation and data readiness overlapping responsibility for what counts as loaded.

### Badge textures have no replacement owner

`BadgeUnlocker.SetBadges` allocates a new `Texture2D` for each nonempty badge-status update, fills it from the status array, and assigns it to the sash material. The generated texture is a local variable. The method does not keep an owned reference for reuse or destroy the previous generated texture when replacing the material assignment.

This path has ordinary consumers. `CharacterData.SyncBadgeStatus` calls `BadgeUnlockVisual`, which calls `SetBadges`; join-state application calls the same visual refresh. `PeakHandler` also uses `SetBadges` for its character presentation. The allocation is not confined to the separate debug branch in `BadgeUnlocker.Update`.

Replacing the material's texture reference does not express a lifecycle for the previous generated Unity object. Repeated badge refreshes allocate replacement objects with no explicit release in this owner. The cost is avoidable allocation and deferred cleanup responsibility, even though the status array could often be represented by updating the existing small texture.

The implementation also creates the texture before checking whether the supplied sash renderer is null. In that branch it has allocated and filled a texture that will not be assigned anywhere. Validation belongs before allocation.

A per-renderer owned texture could be updated in place and released when its owner is destroyed. If replacement is necessary, the owner should release its previous generated texture. Either option would make the resource lifetime explicit rather than relying on eventual scene or unused-resource cleanup.

## For modders

Follow the achievement's actual source of truth before treating its displayed badge as the record. Statistic-linked achievements, explicit platform awards, and the run's newly earned list are different inputs. The sash's badge-status array is a presentation copy; changing its appearance does not establish progression.

Cosmetic catalog ordering is part of the persistence contract. A visual replacement and a catalog reorder have different consequences because saved and shared choices are indices. Keep selection eligibility separate from applying received appearance, and inspect the existing correction paths rather than assuming the method names guarantee validated data.

## Open questions

No platform award, rejected statistic request, or cosmetic restoration failure was exercised here. The accessory finding identifies a wrong-field comparison; its player-visible outcome depends on which values the platform supplies. The guidebook findings require observations before successful statistics initialization to expose their early-loading consequences.

Texture allocation is source-visible, but its memory impact over a session and the timing of unused-resource cleanup have not been measured. The serialized badge atlas and material behavior also need separate inspection before attaching a specific visual defect to the sash.

The chapter does not establish that a submitted award has been durably stored when the local event fires. Platform acknowledgement and retry behavior require tracing the platform implementation. Unlock requirements and numeric thresholds have deliberately not been reproduced as a catalog or an unlock procedure.
