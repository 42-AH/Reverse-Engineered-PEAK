# Stamina and afflictions

The stamina bar appears to be a single system, but inside `CharacterAfflictions`, accumulated amounts and timed effects use separate storage, update rules, and network paths. That's reasonable enough. The real problem emerges when code needs to tell the difference between changing a value and restoring one it already had.

Movement and climbing both consume stamina. Hunger, injury, and other conditions reduce the available regular bar, while bonus stamina sits in its own pool. Some conditions go away on their own; others need an item or some other intervention. Weight tracks carried objects and scouts. Embedded hazards add both visible attachments and status costs. Timed effects can change those processes without taking up another slot in the regular bar. When accumulated status fills that bar, `Character.HandleLife` moves toward unconsciousness; petrification takes a separate route through the same broader life-state machinery.

## Stamina and its consumers

`CharacterData.currentStamina` and `extraStamina` store the two pools. `Character.UseStamina` serves as a shared consumption entry point, called by movement and equipment code. `CharacterMovement.SetMovementState` requests stamina while sprinting. `CharacterClimbing.Update` calculates a climbing request from movement, wall angle, and modifiers before calling the same method. Rope and vine controllers also request stamina, but they use the result to decide whether their attachment can continue.

Regular recovery follows a different path:

- `Character.FixedUpdate` calls `UpdateVariablesFixed`, which advances the timers used by recovery.
- `UpdateVariablesFixed` checks `CanRegenStamina` to see whether the scout can recover, taking into account recent use and grounding as well as attachment exceptions.
- `AddStamina` changes the owned scout's regular pool and calls `ClampStamina`.
- `ClampStamina` gets its ceiling from `GetMaxStamina`, which reads `CharacterAfflictions.statusSum`.

Bonus stamina is added and clamped on its own. `SetExtraStamina` and `AddExtraStamina` check `CharacterData.petrifyAmount`; `CharacterData.SetPetrify` reclamps the bonus pool when that amount changes. Petrification appears in the status enum, but its value lives outside the ordinary status array. `BarAffliction.ChangeAffliction` handles that exception when it picks the value to display.

## Status accumulation

`CharacterAfflictions.InitStatusArrays` creates `currentStatuses` and matching arrays for pending additions, pending reductions, and timestamps. The enum picks array positions. `statusSum` adds the committed amounts; pending fractions stay separate. `Character.HandleLife` reads `shouldPassOut`, while `StaminaBar.Update` reads the committed sum and maximum stamina to size the bar and its overflow display.

`AddStatus` checks eligibility and applies gameplay transformations before accumulation. This includes character form, difficulty modifiers, and interactions between opposing temperatures. Committed changes clamp regular stamina and trigger display updates. `SubtractStatus` has its own accumulator and side effects, including release from trapping conditions. `SetStatus` is a standalone setter that clears pending changes for the chosen status. `AdjustStatus` routes to addition or subtraction based on the requested direction.

The accumulator preserves fractions right up until they become useful, then discards the remainder.

`AddStatus` gathers small requests in `currentIncrementalStatuses` until there is enough for a committed increment. It commits whole increments and then clears the accumulator entirely. Any remainder above those whole increments disappears. `SubtractStatus` does the same to `currentDecrementalStatuses` after committing a reduction.

Accumulation should let small updates contribute without forcing every update into the displayed representation. Clearing the whole accumulator breaks that accounting. The result depends on how exposure or recovery is split into requests, not just on its total amount. Different request sizes leave different discarded remainders.

`UpdateNormalStatuses` submits hunger requests scaled by time, along with recovery requests. `Affliction_AdjustColdOverTime.UpdateEffect` also multiplies its rate by elapsed frame time before calling `AdjustStatus`. Their integration passes through the lossy accumulators. Multiplying by elapsed time does not recover a remainder that gets discarded later.

This is incorrect accounting in actual gameplay state, not a cosmetic rounding issue. The committed amounts feed `statusSum`, `Character.GetMaxStamina`, and `shouldPassOut`; `StaminaBar` is just one consumer. `AddStatus` and `SubtractStatus` throw away exposure or recovery that has already been accumulated. Keeping the remainder would fix the loss. Separating continuous state from quantized display would be even cleaner. The current approach takes the effort to accumulate fractions only to throw part of them away.

The standard update loop calls those entry points. `UpdateNormalStatuses` adds hunger while the local scout is conscious and requests timed recovery for eligible conditions. `UpdateWeight` calculates weight from inventory and backpack contents, a carried scout, and attached objects. It writes the result with `SetStatus`; weight comes from possessions rather than being healed like an exposure amount.

## Timed effects and restoration

Items enter the timed-effect system through `Action_ApplyAffliction.RunAction`, which calls `CharacterAfflictions.AddAffliction`. The manager copies the supplied `Affliction` and looks for an existing object of the same type. A new instance gets the character reference and `OnApplied`; an existing instance gets `Stack` and `OnStacked`. Each effect class defines how it stacks.

Keeping a lasting condition separate from an effect that produces or modifies it makes sense. Poison exposure and a timed source of poison need different lifetimes. Copying the effect supplied by an item also prevents the item's definition from becoming the live character's mutable timer.

`CharacterAfflictions.Update` calls `UpdateEffectNetworked` on active effects and ticks their lifetime on the owning character. `Affliction.Tick` advances the lifetime and calls `UpdateEffect` while the effect is still active. For instance, `Affliction_AdjustColdOverTime.UpdateEffect` submits a time-scaled request to `AdjustStatus`. When it expires, the manager calls `RemoveAffliction`, which invokes `OnRemoved`.

`UpdateNormalStatuses` requests natural poison recovery after its recovery delay. Those requests reach `SubtractStatus`. Its timed-effect clearing branch gets selected by the reduction request and condition type, without excluding natural recovery through `decreasedNaturally`.

That branch calls `ClearPoisonAfflictions`. The method scans the effect list, removes matching objects, and calls `PushAfflictions` regardless of whether the scan found anything. `PushAfflictions` then copies the list, serializes it, and requests an immediate send for the local character. It has no check for unchanged state.

As a result, repeated recovery requests can repeatedly serialize and send an unchanged collection, even an empty one. The committed status amount need not change on every request because its reduction is accumulated separately. The extra effect work is not limited to meaningful status increments or actual removals.

`ClearPoisonAfflictions` does unnecessary work and requests unnecessary sends. The code already knows how to batch statuses: `PushStatuses` marks a pending update and `ActuallyPushStatuses` processes it. Effect removal even has a way to skip individual sends. Yet a scan that removes nothing still builds and serializes the collection through `PushAfflictions`. This is an inefficient update path with a simple fix: report whether removal changed anything, then schedule a send only when it did. There is no need to speculate about bandwidth to object to allocating and serializing an unchanged list.

Ordinary statuses and timed effects follow separate transmission paths. `PushStatuses` marks status data for a later send through `ActuallyPushStatuses`. `PushAfflictions` serializes and sends the effect collection right away. Received status data goes from `SyncStatusesRPC` to `SetAll`; reconnect restoration also uses `SetAll` through `ApplyReconnectData`. `ReconnectHandler` subscribes to `StatusesUpdated` and builds a fresh record through `ReconnectData.CreateFromCharacter`.

The missing separation is between a gameplay request and an already-decided state. A damage request should pass through difficulty and immunity rules. Restoring a recorded amount should simply establish that amount. Yet both currently route through the same mutation routines. Since presentation and transmission are mixed into those routines as well, it becomes difficult to reason about a status change without tracing through unrelated consumers.

Restoring a saved amount should not require processing it through gameplay mechanics a second time.

`SetAll` takes a collection of status amounts, compares them against the current collection, and routes differences through `AddStatus` or `SubtractStatus`. This approach reuses ordinary gameplay behavior for restoration. The call path is shared by received status updates and `ApplyReconnectData`, so its semantics affect both remote representation and recovered state.

When called from this restoration path, `AddStatus` still applies difficulty processing to affected types. Its network-origin flag modifies some checks and sending behavior, but it does not turn the operation into a simple assignment. `Ascents.etcDamageMultiplier` can deviate from its neutral value in custom runs. When this happens, a difference calculated from an already-established status amount gets transformed again before being committed. In effect, a restored value is treated as a fresh gameplay request.

Subtraction is not a plain correction either. `SubtractStatus` includes condition-specific behavior beyond updating a single array entry, including interactions with other conditions and timed effects. These behaviors make sense for healing. They represent a poor contract for a method intended to bring a collection into agreement with recorded state.

`SyncStatusesRPC` raises `StatusesUpdated` after `SetAll`, and `ReconnectHandler.UpdateReconnectData` builds a record from the character's resulting values. `StaminaBar.Update` and `BarAffliction.ChangeAffliction` likewise read the resulting character state. They do not keep a separate, exact copy of the incoming amounts to verify the mutation's interpretation.

`SetAll` is the wrong abstraction for restoration because it delegates assignment to gameplay mutation. A recorded amount should not be reinterpreted as new damage or healing under whatever rules happen to be active. Restoration requires its own validated assignment path and a refresh of derived state. While reusing `AddStatus` and `SubtractStatus` avoids a separate implementation, it sacrifices the one property restoration depends on: agreement with the supplied state.

`SetStatus`, committed branches in `AddStatus` and `SubtractStatus`, and the stamina methods in `Character` all call `GUIManager.instance.bar.ChangeBar` directly. `ChangeBar` updates the `BarAffliction` components using `Character.observedCharacter`, which may be a different scout from the object whose state actually changed. The gameplay owner reaches into a global presentation object to refresh whichever character is currently on screen.

`AddStatus` does far more than that refresh. It handles web-wrap animation, struggle prompts, particle playback, sound selection, and biome-related achievement tracking. These represent separate consumers of a status event, yet their work is baked into the status mutation itself. Any change to status accounting must preserve scene references and presentation behavior, even when a test only cares about amounts and caps. Isolated state tests therefore require either a surrounding Unity setup or replacement dependencies, since the arithmetic is inseparable from the HUD.

The existing event does not bridge this gap. `StatusesUpdated` is raised by `ActuallyPushStatuses` and by `SyncStatusesRPC`, but not as the common completion event from the mutators. `ReconnectHandler` depends on that event to refresh recorded player state. Notification timing becomes coupled to the send path for an owned character and to the receive path for a remote character. Changing how data gets batched changes when this consumer learns about state.

`CharacterAfflictions` is overburdened, and the coupling actively blocks testing. Arithmetic, HUD refresh, effects, network scheduling, and notification sit tangled together in the same manager. A completed local mutation should publish its resulting value. Presentation, reconnect recording, and transmission should then consume that event separately. Instead, changing send timing changes when another system learns the state has changed. That represents a bad dependency to encode into the core status model.

## Reading and changing this code

Keep accumulated statuses distinct from timed `Affliction` instances. Removing a producer and removing the status it produced are different operations. `AddAffliction` clones the supplied definition, and its live instance is the one held in `afflictionList`, with the character reference assigned by the manager. Individual `Stack` implementations decide whether another application extends duration or modifies an existing parameter.

Treat display and observation as separate concerns from ownership. `BarAffliction` reads the observed character, while the manager's mutation and sending checks concern the character holding the state. Any visual extension should maintain that distinction. Names like `SetAll` do not reveal assignment semantics on their own. Its consumers and the mutation methods establish the actual contract. No network invocation or state-alteration procedure gets provided here.

## Open questions

The fractional-loss finding follows from the accumulator writes. Its size during ordinary play depends on the serialized rates and update timing, which have not been measured in the official Steam build. No particular exhaustion time or difference between frame rates has been established.

The restoration finding establishes that gameplay transformations remain in the path. The visible consequences under custom-run settings, and the frequency with which later updates reduce or repeat a discrepancy, still need official-build observation. This chapter does not establish a reported multiplayer symptom or a corrupt save.

The recovery path contains unchanged effect sends, but their bandwidth and frame-time impact have not been profiled. The event placement establishes a coupling to transmission; it does not by itself establish a lost reconnect record. These are distinct claims.
