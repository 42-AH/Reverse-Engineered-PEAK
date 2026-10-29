# Items, inventory, and equipment

An item changes hands between its world object, inventory entry, and held behavior. Pocket effects can keep running while the object is stored. That gives PEAK a flexible item system, provided every component remembers which part of the item's life it owns. There is quite a lot of remembering involved.

The inventory holds a definition of an item and the state belonging to that particular instance. That distinction lets remaining uses, cooking state, fuel, and equipment activation survive changes between storage and a visible object. Backpacks contain slots of their own. Carried weight connects inventory choices to the stamina and affliction system described in chapter 03.

Using an item can start an action immediately, hold it through a cast, complete it, or cancel it. Food combines effects such as hunger restoration with consumption. Feeding introduces another character as the recipient. Equipment uses the same action framework but can also maintain a longer-lived component: a rope being paid out, a placement preview, a burning fuse, or an activated amulet. Bing Bong adds a timed presentation sequence with sound, subtitles, and an event for listeners.

## Pickup and instance data

The pickup path separates interaction from acceptance:

- `Interaction` dispatches an interaction to the selected `IInteractible`.
- `Item.Interact` checks local eligibility and begins the pickup request.
- `Item.RequestPickup` asks `Player.AddItem` to put the item definition and instance data into a slot.
- A successful result reaches `CharacterItems.OnPickupAccepted`; an unsuccessful one reaches `Item.DenyPickupRPC`.

`OnPickupAccepted` decides whether to equip the accepted slot immediately, accounting for climbing and the backpack slot, then refreshes carried weight. The visible world object and the inventory entry are separate participants in this path. Acceptance is not just a renderer toggle.

`ItemSlot` holds the prefab reference and `ItemInstanceData`. The latter owns a dictionary indexed by `DataEntryKey`, with values represented by subclasses of `DataEntryValue`. `Item.GetData<T>` supplies a default entry when the requested key is absent. When an item receives instance data, `Item.SetItemInstanceDataRPC` notifies its `ItemComponent` components through `OnInstanceDataSet`. Components such as `RopeSpool`, `Dynamite`, and `ItemCooking` use that hook to reconcile their own working state with the instance.

Separating definition, instance data, and visible object gives the inventory a useful boundary. A remaining-fuel value belongs to the item instance; it need not be encoded in the identity of a newly held object. The cost is reconciliation: each component must restore the working state it keeps outside the dictionary. `OnInstanceDataSet` is the explicit place to do that.

The generic API looks rather more type-safe than the dictionary behind it actually is.

`ItemInstanceData.TryGetDataEntry<T>` checks whether a dictionary entry exists and then casts it to the caller's requested type. It does not check compatibility before the cast. `RegisterEntry<T>` and `RegisterNewEntry<T>` accept the key and value type independently, so the type system does not prevent two components from disagreeing about the meaning of a key.

`Item.GetData<T>` builds on that contract. It can create a default for an absent key, but a present value of the wrong type throws instead of reaching the default path. The distinction is relevant to real consumers: `ItemCooking` expects integer cooking state, `RopeSpool` expects floating-point fuel, and `ItemPocketBehaviorInstanceBase.behaviorEnabled` expects a boolean activation value. Those expectations live at the call sites.

The concrete cost is a runtime failure boundary for an internal schema mistake. A component can register a value successfully and fail later when another component reads it. The error then appears in cooking, equipment updates, or pocket behavior rather than at the incompatible write. This moves a schema maintenance error into a gameplay update.

A typed key should carry its value contract, or a central schema should validate the key/value pairing at registration. Reads should distinguish absent data from incompatible data, with a diagnostic identifying the expected and actual types. Silently treating an incompatible value as absent would only conceal the disagreement. The same contract should cover restoration so an invalid entry cannot become ordinary working state merely because it was restored rather than freshly registered.

## Using and consuming an item

Use follows a different chain:

- `CharacterItems.DoUsing` interprets primary and secondary input for the held item.
- `Item.StartUsePrimary` and `ContinueUsePrimary` maintain cast state; `FinishCastPrimary` publishes completion.
- `ItemAction.Subscribe` attaches a component's `RunAction` to the selected item delegates.
- An effect component performs its work: `Action_RestoreHunger.RunAction`, for example, reaches `CharacterAfflictions.SubtractStatus`.

The secondary-input path deserves a precise reading. `DoUsing` calls `StartUseSecondary` while the button is held, but `StartUseSecondary` refuses to restart an already active use. The caller's repeated invocation therefore does not, by itself, reset secondary cast progress. Treating the caller alone as a defect would miss the callee's guard.

Composition also makes food effects independently reusable. Hunger restoration, added stamina, status changes, and consumption do not require a separate item subclass for every combination. But selecting when those components run becomes part of gameplay configuration. Subscription rules are consequently part of the behavior contract, not incidental event plumbing.

An advertised action stage is only useful if something subscribes to it.

`ItemAction` exposes an `OnReleased` flag alongside its other lifecycle selectors. Neither `Subscribe` nor `Unsubscribe` reads it. A search of the inspected assembly's source finds the declaration but no consumer of that flag.

This is not how the neighboring selectors behave. `OnPressed`, `OnHeld`, and `OnCastFinished` each connect `RunAction` to a particular delegate. `OnReleased` presents the same configuration surface without supplying equivalent behavior. An action configured only through that flag will not acquire a release subscription from this class.

`CharacterItems.DoUsing` makes the gap more consequential to understand: releasing primary input reaches `CancelUsePrimary`, and that cancellation path is also used when an item becomes unusable. `OnCancelled` therefore does not express a distinct successful button-release event. Treating the two names as interchangeable would hide a semantic choice from whoever configures the action.

The repair depends on the intended contract. If release is a supported stage, it needs an explicit event and a defined relationship to cancellation and completed casts. If it is unsupported, the inert selector should be removed or clearly rejected by validation. Leaving it exposed makes an apparently valid configuration fail silently. The base component advertises a behavior it does not implement.

Consumption is also staged. `Action_Consume.RunAction` starts `Item.ConsumeDelayed`; `Item.Consume` handles the consumed-item event, the holder's animation and current slot, and deactivates the item object. Restoring hunger and removing the consumed object are different responsibilities, connected through the configured action lifecycle.

## Backpacks and pocket effects

Backpacks extend the slot system through `BackpackData`. `Backpack.Stash` handles the held item, while `RPCAddItemToBackpack` passes its definition and data into the backpack slot. Pocket behaviors are a separate extension point: `ItemSlot.SetItem` creates their instances, and `UpdatePocketBehaviors` checks whether they work inside a backpack and whether they need a holder. `ItemPocketBehaviorInstanceBase` retains the instance data and exposes owner-change hooks. That supports an effect whose lifetime follows inventory membership rather than a renderer's visibility.

Pocket behavior makes another useful distinction. Being stored does not necessarily mean being inactive. Its owner hooks provide a place to remove an effect from the previous character and attach it to the next one. That lifetime differs from both the held object's lifetime and the lifetime of its underlying instance data.

## Cooking

Cooking has two connected outputs. `ItemCooking.FinishCookingImpl` changes the stored cooking state and calls `UpdateCookedBehavior`. That method updates appearance, applies the outstanding cooking steps to food action components, and invokes the configured `AdditionalCookingBehavior` objects. A cooked food's effects are therefore partly the result of changes to other components, not just a value consulted at consumption time.

`ItemCooking.UpdateCookedBehavior` records one default tint for each renderer. It obtains that tint from the renderer's first material. `CookVisually` then iterates over every material on that renderer and applies a cooked tint derived from the same recorded value.

The mismatch is between the size of the saved state and the size of the state being modified. The consumer writes per material slot, but the producer remembers only per renderer. A renderer with differently tinted materials loses those differences when this path colors it: each slot receives the first material's baseline multiplied by the cooking color.

That distinction matters for an author adding a food with a separate wrapper, filling, or decoration material. Preserving those original tint differences requires a workaround outside the current default cooking path. Changing the cooking multiplier cannot recover information that was never stored.

The replacement should preserve a baseline for each material slot it intends to modify, or explicitly designate the slots that receive cooking coloration. A rendering parameter layer could then apply the cooking contribution without overwriting unrelated material choices. The essential fix is matching the granularity of the saved baseline to the granularity of the writes. Replacing one color constant with another does not address this mechanism.

## Equipment and delayed actions

Specialized equipment keeps this structure but adds its own state machine. `RopeSpool.Update` manages the held rope object and input; `FixedUpdate` adjusts its length, while `RopeFuel` stores remaining capacity in instance data. `Constructable` maintains a preview and checks the candidate surface before `FinishConstruction` creates the placed object and retires the held item. `Dynamite.Update` advances its active fuse and coordinates the explosion and item cleanup. `Action_LightLantern` delegates to `Lantern.ToggleLantern`, while `Action_Balloon` delegates attachment to `CharacterBalloons`. These are distinct consumers of the item framework, not one universal equipment algorithm.

Magical effects reuse afflictions through `Action_ApplyAffliction`; amulets add activation state through `AmuletBase`. Its `OnInstanceDataSet` restores the gem's state, and its activation methods coordinate that state with presentation. `Action_AskBingBong` instead chooses a response and launches a delayed sequence through `Ask` and `AskRoutine`, with `SubtitleRoutine` controlling the accompanying text.

`Action_AskBingBong.Ask` checks `askRoutine` and attempts to stop the previous coroutine before starting another `AskRoutine`. The return value of the new `StartCoroutine` call is not assigned to that field. The field has no other assignment in the inspected class, so the cancellation branch cannot track the response sequence it is meant to stop.

The downstream work is not a harmless delay. `AskRoutine` updates the attached item while it waits, then controls the audio source and starts `SubtitleRoutine`. It also raises `OnAsk` when playing a response. The class correctly retains a handle for its subtitle coroutine, which makes the missing handle for the outer routine especially concrete: subtitle replacement is managed, but the delayed sequence that initiates it is not.

The result is that invoking another ask does not cancel the prior outer sequence through this guard. Its remaining work can continue, including reaching later presentation operations. The spam check reduces which sequences proceed to a spoken response; it does not repair cancellation, and it occurs after the initial attachment-update loop.

Retaining the outer coroutine handle would make the existing cancellation logic effective. A complete lifetime contract would also define what happens when the item is stashed, consumed, or loses its holder during the delay, and would clear the handle when the sequence finishes. The cost of the present implementation is a presentation sequence that appears to have replacement semantics in `Ask` but does not actually enforce them.

## Reading and changing this code

Treat item definition, instance state, and held-object state as separate concerns. A visual change does not establish that fuel, activation, or cooking state restores correctly when the item returns from inventory. Inspect the relevant `OnInstanceDataSet` implementation before assuming that changing an instance entry also updates every component cache.

For action changes, trace the selected lifecycle delegate to `RunAction` and then to the effect's consumer. The difference between completion, consumption, and cancellation matters. For passive equipment, follow `ItemSlot` and the pocket behavior's owner hooks as well as the visible component. These checks identify compatibility obligations without requiring changes to multiplayer authority or transport.

## Open questions

These findings come from source inspection, without a completed official gameplay test. Which shipped items expose the conditional tint problem, whether stock components ever disagree about entry types, and whether the ineffective Bing Bong cancellation produces a visible or audible symptom remain unverified.

The source does not establish which shipped food renderers combine multiple differently tinted materials, or whether any serialized action selects the inert release flag. Those are content-reference checks, separate from the implementation findings.

Cooking applies outstanding steps against mutable component values. Its expected monotonic progression is clear, but the permitted lifecycle for replacing instance data with an earlier cooking state needs a broader call-site audit before treating the absence of reversal as a defect. Backpack transfers likewise need timing analysis beyond their individual methods before making any claim about item loss.
