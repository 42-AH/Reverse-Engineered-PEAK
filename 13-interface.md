# Menus, HUD, and communication

The HUD mostly reads state and displays it, which should make it one of the easier places to keep cause and effect straight. Instead, item cells and prompt detectors cache values they haven't quite identified correctly, saving small amounts of work.

The interface connects the main menu and pause pages to the playable scout, then presents inventory, stamina, interaction prompts, climbing feedback, and spectating state. The passport edits appearance; the guidebook renders readable pages onto a held object. Pings and the emote wheel let scouts communicate intent to each other. These interfaces use different focus and lifecycle mechanisms, so closing a page, hiding a window, releasing a wheel, and putting away a book are all distinct operations.

## Pages, windows, and pause

`MainMenuPageHandler` and `PauseMenuHandler` derive from `UIPageHandler`. Main menu back input follows `IHaveParentPage.GetParentPage` and transitions to the returned page. The handler also shows connection state, prepares current map information after the version response, and opens the disconnected modal when connection state requires it. This makes the main menu both a page navigator and a consumer of connection lifecycle events.

`PauseMenuHandler.OnEnable` pauses time in offline mode and activates the main pause page. Its back input path first follows a parent page, then handles confirmation or modal closure, then allows leaving pause if unpause prevention is inactive. `OnDisable` restores the configured time scale for offline play. Online pause does not stop the shared simulation.

`MenuWindow` is a separate window abstraction. `Open` records the window in `AllActiveWindows`, shows its panel, initializes it once, invokes the open hook, and picks a starting UI element. `Close` removes it from that list and invokes its close behavior. Properties describe whether a window blocks player input or displays a cursor. `GUIManager.UpdateWindowStatus` combines those properties and keeps a short input blocking margin after a blocking window closes.

## The observed character and inventory HUD

The in game update path centers on `GUIManager.LateUpdate`:

- A changed `Interaction.currentHovered` triggers `OnInteractChange`, which refreshes the interaction text.
- Character availability sets up the item selection callback and enables local character displays.
- Reticle, throwing, rope, dying, emote wheel, spectating, and pause state all refresh through separate methods.
- `UpdateItems` reads `Character.observedCharacter`, so inventory and bar presentation can follow the spectated scout instead of always following the local one.

Using `observedCharacter` for the HUD works well for spectating. It also means the inventory display must tolerate a character whose player reference isn't available yet. `GUIManager.UpdateItems` explicitly handles that case. The item cell's null path fails to uphold the caller's contract.

The caller has already noticed the missing player. The cell manages to turn that precaution back into a dereference.

`GUIManager.UpdateItems` returns when there is no observed character. When there is an observed character but no associated player, it attempts a deliberate reset: it passes null to each inventory cell and the backpack cell, updates prompts, and hides the temporary slot.

`InventoryItemUI.SetItem` recognizes a null slot and calls `Clear`, but doesn't return. Execution continues into the normal slot logic. An ordinary inventory cell then reads `slot.prefab`; the backpack branch reaches `slot.IsEmpty` unless its carried player presentation returns first. Clearing the old display doesn't make the missing slot usable.

This is a matched caller and callee failure. The caller has a special path for incomplete player state, and the callee advertises handling that path through its opening null check. The handler then proceeds into the state it just recognized as unavailable. An exception there stops that update before the intended reset sequence completes.

`InventoryItemUI.SetItem` has a broken empty state path. Clearing a null slot must end the update before the method dereferences that slot. The caller already handles an unavailable player by asking the cell to clear itself; the cell defeats that precaution. An early return or an explicit clear operation is enough. This is a straightforward null handling mistake in code whose whole purpose includes displaying an empty slot.

`InventoryItemUI.SetItem` receives an `ItemSlot`. It distinguishes backpack presentation, carried scout presentation, and ordinary item slots. It stores the prefab and instance data, chooses the icon, updates cooking tint and name, and maintains fuel and selection feedback. `GUIManager.UpdateItemPrompts` separately derives held item prompts from the local item's UI description and current primary and secondary use eligibility.

The interface's performance strategy is inconsistent. Inventory cells skip some work when the prefab hasn't changed, and item prompts have a change detector, but both caches remember the wrong thing in certain paths. A cache needs an exact statement of what can change independently. Item identity and item type are not the same thing.

Remembering the item type costs less than remembering the item. It's also the wrong cache key for this job.

In the ordinary slot branch, `InventoryItemUI.SetItem` compares the incoming prefab with `_itemPrefab`. If they match, it updates fuel from the incoming slot, calls `UpdateNameText` and `UpdateCookedAmount`, and returns. It doesn't assign `_itemData` from the new slot on that path.

`ItemSlot.SetItem` can replace the slot's `ItemInstanceData` independently of its prefab. Two items of the same kind therefore pass the UI's cache test while carrying different state. `UpdateNameText` passes the retained `_itemData` into the prefab's name calculation, and `UpdateCookedAmount` reads cooking state from that retained data. Fuel, meanwhile, was refreshed from the incoming slot. Different parts of one cell can describe different instances.

The ordinary caller, `GUIManager.UpdateItems`, runs this method repeatedly. Polling more often doesn't repair the mismatch: as long as the prefab stays equal, the early return keeps preserving the old data reference. A change of type or a separate clear is needed to reach the assignment below the shortcut.

The inventory cache uses the wrong level of identity. An unchanged prefab can justify keeping an icon; it can't justify keeping another instance's data. Refresh the instance model before deciding which visual work to skip. By putting the shortcut ahead of that assignment, `InventoryItemUI` preserves stale state while congratulating itself on avoiding an update. The optimization is incorrect, not just incomplete.

`GUIManager.TestUpdateItemPrompts` reads `CanUsePrimary` and `CanUseSecondary` from the held item. It compares those values with `canUsePrimaryPrevious` and `canUseSecondaryPrevious` and refreshes prompts when either differs. At the end, it assigns the primary history field twice: first from primary eligibility, then from secondary eligibility. The secondary history field doesn't get updated there.

When secondary use is available, the unchanged secondary history makes the comparison keep reporting a change. Other combinations leave the primary history describing secondary eligibility rather than primary eligibility. The method's stored state can't represent the pair it is comparing.

`UpdateItemPrompts` does text lookup and sets the active state of several prompt objects. The broken detector introduces repeated calls to those operations without a corresponding change in eligibility. It also makes the detector unreliable as the sole invalidation mechanism for future changes.

The current HUD has another path that limits the visible consequence: `UpdateItems` calls `UpdateItemPrompts` again during normal updates. This means the defect doesn't establish a permanently stale prompt in the current implementation. Instead, a broken change detector sits alongside unconditional refresh, so the interface pays for both designs while relying on the less selective one for correctness.

The prompt code runs a broken change detector alongside unconditional refresh. That creates redundant complexity without delivering the promised savings. You need to pick a direction: either poll and remove the dead bookkeeping, or fix both remembered values and invalidate when items and observed characters change. The unconditional path keeps visible damage to a minimum today, but it also conceals a defect that's waiting for someone to strip out what looks like an unnecessary refresh.

## Passport and guidebook

The passport extends `MenuWindow`. `PassportManager.ToggleOpen` stops the window from reopening while its delayed close animation plays. Tabs shift the preview camera and fill selection buttons; `SetOption` updates character customization and refreshes the dummy preview. The progression chapter covers persistence and unlock requirements beyond this UI.

`Guidebook` stays as an item while you have it open. Its owner tracks the open state, and `ToggleGuidebook_RPC` updates the collider, animation, page-renderer textures, and page display. For whoever holds it locally, `LateUpdate` moves the open book toward where the camera can read it. Individual pages report when you've seen them. The progression chapter handles guidebook progress and any readiness issues.

## Pings and emotes

`PointPinger` watches for local input, checks consciousness and cooldown, raycasts from the mouse or a controller-centered screen position, and sends a shared ping request. The presentation that receives it checks distance and obstruction between scouts before it creates the marker. `EmoteWheel` instead connects pointer or gamepad choices to an emote definition. Closing the wheel calls `Choose`, which sends the selected animation to `CharacterAnimations.PlayEmote`.

`EmoteWheel.OnDisable` always runs `Choose`. `Choose` plays whichever emote you've hovered over, assuming you selected one. There's no way to tell whether you meant to confirm or cancel. `GUIManager.UpdateEmoteWheel` closes the wheel when you stop meeting its conditions, which covers eligibility changes and climbing state shifts as well as simply letting go.

Release-to-select works fine as a wheel interaction. The real issue is putting that commitment inside a generic component lifecycle callback. Any owner that closes the wheel gets automatic selection, even when they're just hiding an interface that isn't available. The wheel has no separate cancel operation that an owner can call instead.

`EmoteWheel.OnDisable` treats hiding a control as confirming your choice. That's the wrong lifecycle hook for a gameplay action. Every new reason to hide the wheel now triggers a possible call to `CharacterAnimations.PlayEmote`, whether you actually meant to confirm anything. Split confirm and cancel into separate operations and let disable handle cleanup. This shortcut turns a normal UI change into something that requires understanding an unrelated action's side effects.

## Reading and changing this code

Keep the local character separate from the observed character. Input and held-use commands belong to local input, while inventory and status can track a spectated character. Grabbing whatever global character reference is closest will shift what the interface actually does.

Respect each UI's lifecycle abstraction before you change what's visible. `MenuWindow.Close` refreshes the active-window registry, page transitions handle navigation, and disabling the emote wheel confirms its current selection. Hiding something isn't a general cancel operation across all these systems.

## Open questions

The null-slot defect has an explicit caller, but this analysis has not measured when ordinary scene or spectating transitions expose that missing-player state. The stale instance-data finding depends on a slot receiving another data object with the same prefab; it does not establish a particular incorrect item label reported by players.

The wheel finding identifies a lifecycle coupling, not a demonstrated unwanted animation. `CharacterAnimations.PlayEmote` and current character eligibility can further constrain the visible result. Scene wiring, controller navigation order, and modal focus still require observation beyond the component source.

Guidebook render-texture configuration, page layout, font legibility, and accessibility presentation have not been visually audited here. This chapter does not infer those outcomes from the presence of the corresponding UI classes.
