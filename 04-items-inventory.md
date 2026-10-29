# Items, inventory, and equipment

An item moves between its world object, inventory entry, and held behavior. Pocket effects can keep running while the object sits in storage. This gives PEAK a flexible item system, though every component must track which part of the item's life cycle it manages. There is quite a lot of tracking involved.

The inventory stores both a definition of an item and the state for that particular instance. This separation means remaining uses, cooking state, fuel, and equipment activation can persist across changes between storage and a visible object. Backpacks have their own slots as well. Carried weight ties inventory choices to the stamina and affliction system described in chapter 03.

Using an item can start an action right away, hold it through a cast, complete it, or cancel it. Food combines effects like hunger restoration with consumption. Feeding brings another character into play as the recipient. Equipment uses the same action framework but can also maintain a longer-lived component: a rope being paid out, a placement preview, a burning fuse, or an activated amulet. Bing Bong adds a timed presentation sequence with sound, subtitles, and an event for listeners.

## Pickup and instance data

The pickup path separates interaction from acceptance:

- `Interaction` dispatches an interaction to the selected `IInteractible`.
- `Item.Interact` checks local eligibility and begins the pickup request.
- `Item.RequestPickup` asks `Player.AddItem` to place the item definition and instance data into a slot.
- A successful result reaches `CharacterItems.OnPickupAccepted`. A failed one reaches `Item.DenyPickupRPC`.

`OnPickupAccepted` decides whether to equip the accepted slot right away, accounting for climbing and the backpack slot, then updates carried weight. The visible world object and the inventory entry are separate participants in this path. Acceptance is not just a renderer toggle.

`ItemSlot` holds the prefab reference and `ItemInstanceData`. The latter owns a dictionary indexed by `DataEntryKey`, with values represented by subclasses of `DataEntryValue`. `Item.GetData<T>` supplies a default entry when the requested key is missing. When an item receives instance data, `Item.SetItemInstanceDataRPC` notifies its `ItemComponent` components through `OnInstanceDataSet`. Components like `RopeSpool`, `Dynamite`, and `ItemCooking` use that hook to sync their own working state with the instance.

Separating definition, instance data, and visible object creates a useful boundary for the inventory. A remaining-fuel value belongs to the item instance and does not need to be encoded in the identity of a newly held object. The tradeoff is reconciliation: each component must restore the working state it maintains outside the dictionary. `OnInstanceDataSet` is where that happens.

The generic API looks far more type-safe than the dictionary behind it actually is.

`ItemInstanceData.TryGetDataEntry<T>` checks whether a dictionary entry exists and then casts it to the type the caller wants. It does not check compatibility before casting. `RegisterEntry<T>` and `RegisterNewEntry<T>` accept the key and value type separately, so the type system cannot prevent two components from disagreeing about what a key means.

`Item.GetData<T>` builds on that contract. It can create a default for a missing key, but a present value of the wrong type throws instead of falling back to the default. This distinction matters to actual consumers: `ItemCooking` expects integer cooking state, `RopeSpool` expects floating-point fuel, and `ItemPocketBehaviorInstanceBase.behaviorEnabled` expects a boolean activation value. Those expectations live at the call sites.

The generic surface promises more safety than the data model delivers. An incompatible key/value pair can be registered without issue and fail only when another component reads it. That is a schema mistake allowed to slip through until a cooking, equipment, or pocket-behavior update. The error should happen at registration, where both sides of the contract are in view. Deferring it to a later gameplay read only makes diagnosis harder with no real benefit.

A typed key should carry its value contract, or a central schema should validate the key/value pairing at registration. Reads should tell absent data apart from incompatible data, with a diagnostic showing the expected and actual types. Silently treating an incompatible value as absent would only hide the disagreement. The same contract should govern restoration so an invalid entry cannot become ordinary working state just because it was restored rather than freshly registered.

## Using and consuming an item

Use follows a different chain:

- `CharacterItems.DoUsing` interprets primary and secondary input for the held item.
- `Item.StartUsePrimary` and `ContinueUsePrimary` maintain cast state; `FinishCastPrimary` publishes completion.
- `ItemAction.Subscribe` attaches a component's `RunAction` to the selected item delegates.
- An effect component does its work: `Action_RestoreHunger.RunAction`, for instance, reaches `CharacterAfflictions.SubtractStatus`.

The secondary-input path needs careful reading. `DoUsing` calls `StartUseSecondary` while the button is held, but `StartUseSecondary` refuses to restart an already active use. So the caller's repeated invocation does not by itself reset secondary cast progress. Treating the caller alone as a defect would overlook the callee's guard.

Composition also makes food effects independently reusable. Hunger restoration, added stamina, status changes, and consumption need no separate item subclass for every combination. Yet selecting when those components run becomes part of gameplay configuration. Subscription rules thus form part of the behavior contract, not just incidental event plumbing.

An advertised action stage only helps if something subscribes to it.

`ItemAction` exposes an `OnReleased` flag alongside its other lifecycle selectors. Neither `Subscribe` nor `Unsubscribe` reads it. A search of the inspected assembly's source turns up the declaration but no consumer of that flag.

This is not how the adjacent selectors work. `OnPressed`, `OnHeld`, and `OnCastFinished` each wire `RunAction` to a particular delegate. `OnReleased` offers the same configuration surface without providing equivalent behavior. An action set up only through that flag will not get a release subscription from this class.

`CharacterItems.DoUsing` brings the gap into sharper focus: releasing primary input reaches `CancelUsePrimary`, and that cancellation path also runs when an item becomes unusable. `OnCancelled` therefore does not express a distinct successful button-release event. Treating the two names as equivalent would obscure a semantic choice from whoever configures the action.

The advertised action stage is broken. Either release is supported and needs an event with clear cancellation semantics, or it is unsupported and must be rejected during setup. Leaving the selector available while `ItemAction.Subscribe` provides no subscription is the worst path: the setup appears valid and does nothing silently. This is not an obscure runtime tradeoff. The base component advertises behavior it does not implement.

Consumption is also staged. `Action_Consume.RunAction` starts `Item.ConsumeDelayed`; `Item.Consume` handles the consumed-item event, the holder's animation and current slot, and deactivates the item object. Restoring hunger and removing the consumed object are separate concerns, tied together through the configured action lifecycle.

## Backpacks and pocket effects

Backpacks extend the slot system through `BackpackData`. `Backpack.Stash` handles the held item, while `RPCAddItemToBackpack` passes its definition and data into the backpack slot. Pocket behaviors form a separate extension point: `ItemSlot.SetItem` creates their instances, and `UpdatePocketBehaviors` checks whether they work inside a backpack and whether they need a holder. `ItemPocketBehaviorInstanceBase` keeps the instance data and exposes owner-change hooks. This supports an effect whose lifetime follows inventory membership rather than a renderer's visibility.

Pocket behavior draws a useful distinction here. Storage does not necessarily mean inactivity. The owner's hooks give a way to remove an effect from one character and attach it to the next. This lifetime differs from both the held object's lifetime and the lifetime of its underlying instance data.

## Cooking

Cooking produces two connected outputs. `ItemCooking.FinishCookingImpl` changes the stored cooking state and calls `UpdateCookedBehavior`. That method updates how the food looks, applies the pending cooking steps to food action components, and invokes any configured `AdditionalCookingBehavior` objects. A cooked food's effects therefore come partly from changes to other components, not just a value checked at consumption time.

`ItemCooking.UpdateCookedBehavior` saves one default tint for each renderer. It gets that tint from the renderer's first material. `CookVisually` then loops through every material on that renderer and applies a cooked tint derived from the same saved value.

The problem lies in the gap between the size of the saved state and the size of the state being changed. The consumer writes per material slot, but the producer remembers only per renderer. A renderer with differently tinted materials loses those differences when this path colors it: each slot gets the first material's baseline multiplied by the cooking color.

This matters when an author adds a food with a separate wrapper, filling, or decoration material. Keeping those original tint differences requires a workaround outside the current default cooking path. Changing the cooking multiplier cannot recover information that was never stored.

`ItemCooking` loses information because its saved baseline is less detailed than the material writes it performs. This is bad restoration design. Save a baseline per affected slot or explicitly name which slots receive cooking color, then apply the cooking contribution there. A different color constant will not fix the mismatch. The code needs to stop flattening distinctions it later claims to restore.

## Equipment and delayed actions

Specialized equipment keeps this structure but adds its own state machine. `RopeSpool.Update` manages the held rope object and input; `FixedUpdate` adjusts its length, while `RopeFuel` stores remaining capacity in instance data. `Constructable` maintains a preview and checks the candidate surface before `FinishConstruction` creates the placed object and retires the held item. `Dynamite.Update` advances its active fuse and coordinates the explosion and item cleanup. `Action_LightLantern` delegates to `Lantern.ToggleLantern`, while `Action_Balloon` delegates attachment to `CharacterBalloons`. These represent distinct consumers of the item framework, not a single universal equipment algorithm.

Magical effects reuse afflictions through `Action_ApplyAffliction`; amulets add activation state through `AmuletBase`. Its `OnInstanceDataSet` restores the gem's state, and its activation methods sync that state with presentation. `Action_AskBingBong` instead picks a response and launches a delayed sequence through `Ask` and `AskRoutine`, with `SubtitleRoutine` controlling the accompanying text.

`Action_AskBingBong.Ask` checks `askRoutine` and tries to stop the previous coroutine before starting another `AskRoutine`. The return value from the new `StartCoroutine` call does not get assigned to that field. Since the field has no other assignment in the inspected class, the cancellation branch cannot track the response sequence it is meant to stop.

The downstream work is not a harmless delay. `AskRoutine` updates the attached item while it waits, then controls the audio source and starts `SubtitleRoutine`. It also raises `OnAsk` when playing a response. The class correctly keeps a handle for its subtitle coroutine, which makes the missing handle for the outer routine stand out more sharply: subtitle replacement is managed, but the delayed sequence that starts it is not.

So invoking another ask does not cancel the prior outer sequence through this guard. Its remaining work can continue, including later presentation operations. The spam check limits which sequences reach a spoken response; it does not fix cancellation, and it happens after the initial attachment-update loop.

`Ask` contains cancellation logic without saving the handle that would make it work. This is broken replacement behavior, not just sloppy bookkeeping. Save the outer coroutine handle, clear it when complete, and decide what ends the sequence when the holder or item disappears. As it stands, the method suggests that a new request cancels the previous response, but it does not enforce that guarantee.

## Reading and changing this code

Keep item definition, instance state, and held-object state as separate concerns. A visual change does not prove that fuel, activation, or cooking state restores correctly when the item returns from inventory. Check the relevant `OnInstanceDataSet` implementation before assuming that changing an instance entry also updates every component cache.

For action changes, follow the selected lifecycle delegate to `RunAction` and then to the effect's consumer. The difference between completion, consumption, and cancellation matters. For passive equipment, follow `ItemSlot` and the pocket behavior's owner hooks along with the visible component. These checks reveal compatibility obligations without requiring changes to multiplayer authority or transport.

## Open questions

These findings come from source inspection, without a completed official gameplay test. Which shipped items expose the conditional tint problem, whether stock components ever disagree about entry types, and whether the ineffective Bing Bong cancellation produces a visible or audible symptom remain unverified.

The source does not establish which shipped food renderers combine multiple differently tinted materials, or whether any serialized action selects the inert release flag. Those are content-reference checks, separate from the implementation findings.

Cooking applies outstanding steps against mutable component values. Its expected monotonic progression is clear, but the permitted lifecycle for replacing instance data with an earlier cooking state needs a broader call-site audit before treating the absence of reversal as a defect. Backpack transfers likewise need timing analysis beyond their individual methods before making any claim about item loss.
