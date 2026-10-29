# Menus, HUD, and communication

The HUD mostly reads state and displays it. That should make it one of the easier places to keep cause and effect straight. Instead, the item cells and prompt detectors save little bits of work by caching values they have not quite identified correctly.

The interface connects the main menu and pause pages to the playable scout, then presents inventory, stamina, interaction prompts, climbing feedback, and spectating state. The passport edits appearance; the guidebook renders readable pages onto a held object. Pings and the emote wheel communicate intent to other scouts. These interfaces use different focus and lifecycle mechanisms, so closing a page, hiding a window, releasing a wheel, and putting away a book are distinct operations.

## Pages, windows, and pause

`MainMenuPageHandler` and `PauseMenuHandler` derive from `UIPageHandler`. Main-menu back input follows `IHaveParentPage.GetParentPage` and transitions to the returned page. The handler also presents connection state, prepares current map information after the version response, and opens the disconnected modal when the connection state requires it. That makes the main menu both a page navigator and a consumer of connection lifecycle events.

`PauseMenuHandler.OnEnable` pauses time in offline mode and activates the main pause page. Its back-input path first follows a parent page, then handles confirmation or modal closure, then permits leaving pause if unpause prevention is inactive. `OnDisable` restores the configured time scale for offline play. Online pause does not stop the shared simulation.

`MenuWindow` is a separate window abstraction. `Open` records the window in `AllActiveWindows`, shows its panel, initializes it once, invokes the open hook, and chooses a starting UI element. `Close` removes it from that list and invokes its close behavior. Properties describe whether a window blocks player input or displays a cursor. `GUIManager.UpdateWindowStatus` aggregates those properties and retains a short input-blocking margin after a blocking window closes.

## The observed character and inventory HUD

The in-game update path is centered on `GUIManager.LateUpdate`:

- A changed `Interaction.currentHovered` triggers `OnInteractChange`, which refreshes the interaction text.
- Character availability establishes the item-selection callback and enables local-character displays.
- Reticle, throwing, rope, dying, emote-wheel, spectating, and pause state are refreshed through separate methods.
- `UpdateItems` reads `Character.observedCharacter`, allowing inventory and bar presentation to follow the spectated scout rather than always following the local one.

Using `observedCharacter` for the HUD is appropriate for spectating. It also means the inventory display must tolerate a character whose player reference is not available yet. `GUIManager.UpdateItems` explicitly recognizes that state. The item cell's null path fails to uphold the caller's contract.

The caller has already noticed the missing player. The cell manages to turn that precaution back into a dereference.

`GUIManager.UpdateItems` returns when there is no observed character. When there is an observed character but no associated player, it attempts a deliberate reset: it passes null to each inventory cell and the backpack cell, updates prompts, and hides the temporary slot.

`InventoryItemUI.SetItem` recognizes a null slot and calls `Clear`, but does not return. Execution continues into the normal slot logic. An ordinary inventory cell next reads `slot.prefab`; the backpack branch reaches `slot.IsEmpty` unless its carried-player presentation returns first. Clearing the old display does not make the missing slot usable.

This is a matched caller-and-callee failure. The caller has a special path for incomplete player state, and the callee advertises handling that path through its opening null check. The handler then proceeds into the state it just recognized as unavailable. An exception there aborts that update before the intended reset sequence completes.

`InventoryItemUI.SetItem` has a broken empty-state path. Clearing a null slot must end the update before the method dereferences that slot. The caller already handles an unavailable player by asking the cell to clear itself; the cell defeats that precaution. An early return or an explicit clear operation is enough. This is a straightforward null-handling mistake in code whose entire purpose includes displaying an empty slot.

`InventoryItemUI.SetItem` receives an `ItemSlot`. It distinguishes backpack presentation, carried-scout presentation, and ordinary item slots. It stores the prefab and instance data, chooses the icon, updates cooking tint and name, and maintains fuel and selection feedback. `GUIManager.UpdateItemPrompts` separately derives held-item prompts from the local item's UI description and current primary/secondary-use eligibility.

The interface's performance strategy is inconsistent. Inventory cells avoid some work when the prefab is unchanged, and item prompts have a change detector, but both caches remember the wrong thing in particular paths. A cache needs an exact statement of what can change independently. Item identity and item type are not interchangeable.

Remembering the item type is cheaper than remembering the item. It is also the wrong cache key for this job.

In the ordinary-slot branch, `InventoryItemUI.SetItem` compares the incoming prefab with `_itemPrefab`. If they match, it updates fuel from the incoming slot, calls `UpdateNameText` and `UpdateCookedAmount`, and returns. It does not assign `_itemData` from the new slot on that path.

`ItemSlot.SetItem` can replace the slot's `ItemInstanceData` independently of its prefab. Two items of the same kind therefore satisfy the UI's cache test while carrying different state. `UpdateNameText` passes the retained `_itemData` into the prefab's name calculation, and `UpdateCookedAmount` reads cooking state from that retained data. Fuel, meanwhile, was refreshed from the incoming slot. Different parts of one cell can describe different instances.

The ordinary caller, `GUIManager.UpdateItems`, runs this method repeatedly. Polling more often does not repair the mismatch: as long as the prefab remains equal, the early return continues to preserve the old data reference. A change of type or a separate clear is needed to reach the assignment below the shortcut.

The inventory cache uses the wrong level of identity. An unchanged prefab can justify keeping an icon; it cannot justify keeping another instance's data. Refresh the instance model before deciding which visual work to skip. By putting the shortcut ahead of that assignment, `InventoryItemUI` preserves stale state while congratulating itself on avoiding an update. The optimization is incorrect, not merely incomplete.

`GUIManager.TestUpdateItemPrompts` reads `CanUsePrimary` and `CanUseSecondary` from the held item. It compares those values with `canUsePrimaryPrevious` and `canUseSecondaryPrevious` and refreshes prompts when either differs. At the end, it assigns the primary history field twice: first from primary eligibility, then from secondary eligibility. The secondary history field is not updated there.

When secondary use is available, the unchanged secondary history makes the comparison keep reporting a change. Other combinations leave the primary history describing secondary eligibility instead of primary eligibility. The method's stored state cannot represent the pair it is comparing.

`UpdateItemPrompts` performs text lookup and sets the active state of several prompt objects. The broken detector introduces repeated calls to those operations without a corresponding change in eligibility. It also makes the detector unreliable as the sole invalidation mechanism for future changes.

The current HUD has another path that limits the visible consequence: `UpdateItems` calls `UpdateItemPrompts` again during normal updates. That means this defect does not establish a permanently stale prompt in the current implementation. Instead, a broken change detector sits alongside unconditional refresh, so the interface pays for both designs while relying on the less selective one for correctness.

The prompt code maintains a broken change detector alongside unconditional refresh. That is redundant complexity without the promised saving. Choose a policy: poll and remove the dead bookkeeping, or fix both remembered values and invalidate on item and observed-character changes. The unconditional path limits the visible damage today, but it also hides the defect waiting for someone to remove the apparently unnecessary refresh.

## Passport and guidebook

The passport extends `MenuWindow`. `PassportManager.ToggleOpen` prevents reopening while its delayed close animation is running. Tabs alter the preview camera and populate selection buttons; `SetOption` changes character customization and refreshes the dummy preview. The progression chapter traces persistence and unlock requirements beyond that UI boundary.

`Guidebook` remains an item while open. Its owner requests the open state, and `ToggleGuidebook_RPC` changes the collider, animation, page-renderer textures, and page display. For the local holder, `LateUpdate` moves the open book toward the camera's reading position. Single pages report their seen state when opened. Guidebook progress and its readiness defects are covered in the progression chapter.

## Pings and emotes

`PointPinger` checks local input, consciousness, and cooldown, raycasts from the mouse position or controller-centered screen position, and requests a shared ping. The receiving presentation evaluates distance and obstruction between scouts before creating the marker. `EmoteWheel` instead maps pointer or gamepad selection to an emote definition. Disabling the wheel calls `Choose`, which passes the selected animation to `CharacterAnimations.PlayEmote`.

`EmoteWheel.OnDisable` always calls `Choose`. `Choose` plays the hovered emote when one has been selected; there is no parameter distinguishing confirmation from cancellation. `GUIManager.UpdateEmoteWheel` disables the wheel when the hold condition no longer applies, which includes changes in eligibility and climbing state as well as ordinary release.

Release-to-select is a valid wheel interaction. The problem is placing that commitment in a general component lifecycle callback. Any owner that disables the wheel inherits selection commitment even when its purpose is to remove an unavailable interface. There is no separate cancellation operation in the wheel for that owner to use.

`EmoteWheel.OnDisable` confuses hiding a control with confirming the player's choice. That is a bad lifecycle hook for a gameplay action. Every new reason to hide the wheel inherits a possible call to `CharacterAnimations.PlayEmote`, whether confirmation was intended or not. Give confirm and cancel separate operations and leave disabling to cleanup. The current shortcut makes an ordinary UI change require knowledge of an unrelated action side effect.

## Reading and changing this code

Distinguish the local character from the observed character. Interaction and held-use commands belong to local input, while inventory and status presentation can follow a spectated character. Reading whichever global character reference is closest changes the interface's meaning.

Follow each UI's lifecycle abstraction before changing visibility. `MenuWindow.Close` updates the active-window registry, page transitions manage page navigation, and disabling the emote wheel commits its current selection. Hiding an object is not a uniform cancellation operation across these systems.

## Open questions

The null-slot defect has an explicit caller, but this analysis has not measured when ordinary scene or spectating transitions expose that missing-player state. The stale instance-data finding depends on a slot receiving another data object with the same prefab; it does not establish a particular incorrect item label reported by players.

The wheel finding identifies a lifecycle coupling, not a demonstrated unwanted animation. `CharacterAnimations.PlayEmote` and current character eligibility can further constrain the visible result. Scene wiring, controller navigation order, and modal focus still require observation beyond the component source.

Guidebook render-texture configuration, page layout, font legibility, and accessibility presentation have not been visually audited here. This chapter does not infer those outcomes from the presence of the corresponding UI classes.
