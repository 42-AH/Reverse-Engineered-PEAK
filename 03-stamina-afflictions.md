# Stamina and afflictions

The stamina bar makes this look like one system. Inside `CharacterAfflictions`, accumulated amounts and timed effects have separate storage, update rules, and network paths. Fair enough. The trouble starts when the code needs to distinguish changing a value from restoring one it already had.

Stamina pays for movement and climbing. Hunger, injury, and other conditions reduce the available regular bar, while bonus stamina occupies a separate pool. Some conditions recede; others require an item or another intervention. Weight reflects carried objects and scouts. Embedded hazards contribute both visible attachments and status costs. Timed effects can change those processes without being another occupied section of the regular bar. When accumulated status fills that bar, `Character.HandleLife` progresses toward unconsciousness; petrification has a separate route through the same broader life-state machinery.

## Stamina and its consumers

`CharacterData.currentStamina` and `extraStamina` hold the two pools. `Character.UseStamina` is a shared consumption entry point, called by movement and equipment code. `CharacterMovement.SetMovementState` requests stamina while sprinting. `CharacterClimbing.Update` derives a climbing request from movement, wall angle, and modifiers before calling the same method. Rope and vine controllers also request stamina, but use the result to decide whether their attachment can continue.

Regular recovery takes a different path:

- `Character.FixedUpdate` calls `UpdateVariablesFixed`, which advances the timers used by recovery.
- `UpdateVariablesFixed` asks `CanRegenStamina` whether the scout can recover, accounting for recent use and grounding as well as attachment exceptions.
- `AddStamina` changes the owned scout's regular pool and calls `ClampStamina`.
- `ClampStamina` obtains its ceiling from `GetMaxStamina`, which reads `CharacterAfflictions.statusSum`.

Bonus stamina is added and clamped separately. `SetExtraStamina` and `AddExtraStamina` consult `CharacterData.petrifyAmount`; `CharacterData.SetPetrify` reclamps the bonus pool when that amount changes. Petrification is named in the status enum, but its value lives outside the ordinary status array. `BarAffliction.ChangeAffliction` knows about that exception when choosing the value to display.

## Status accumulation

`CharacterAfflictions.InitStatusArrays` creates `currentStatuses` and parallel arrays for pending additions, pending reductions, and timestamps. The enum selects array positions. `statusSum` adds the committed amounts; pending fractions are separate. `Character.HandleLife` reads `shouldPassOut`, while `StaminaBar.Update` reads the committed sum and maximum stamina to size the bar and its overflow presentation.

`AddStatus` handles eligibility and gameplay transformations before accumulation. These include character form, difficulty modifiers, and interactions between opposing temperatures. Committed changes clamp regular stamina and trigger presentation work. `SubtractStatus` has its own accumulator and side effects, including release from trapping conditions. `SetStatus` is a separate setter that clears pending changes for the selected status. `AdjustStatus` dispatches to addition or subtraction according to the requested direction.

The accumulator keeps fractions right up until they become useful, then throws the remainder away.

`AddStatus` accumulates small requests in `currentIncrementalStatuses` until there is enough for a committed increment. It commits whole increments, then clears the accumulator completely. Any remainder above those whole increments is discarded. `SubtractStatus` does the same to `currentDecrementalStatuses` after committing a reduction.

Accumulation is supposed to let small updates contribute without forcing every update into the displayed representation. Clearing the whole accumulator breaks that accounting. The result depends on how exposure or recovery is divided into requests, not only on its total amount. Different request sizes leave different discarded remainders.

`UpdateNormalStatuses` submits time-scaled hunger and recovery requests. `Affliction_AdjustColdOverTime.UpdateEffect` also multiplies its rate by elapsed frame time before reaching `AdjustStatus`. Their integration passes through the lossy accumulators. The multiplication by elapsed time does not undo a remainder discarded later.

This is incorrect accounting in gameplay state, not a cosmetic rounding choice. The committed amounts feed `statusSum`, `Character.GetMaxStamina`, and `shouldPassOut`; `StaminaBar` is only one consumer. `AddStatus` and `SubtractStatus` throw away exposure or recovery that has already been accumulated. Keeping the remainder would fix the loss. Separating continuous state from quantized display would be cleaner still. The current implementation takes the trouble to accumulate fractions and then deletes part of the result.

The ordinary update loop feeds those entry points. `UpdateNormalStatuses` adds hunger while the local scout is conscious and requests timed recovery for eligible conditions. `UpdateWeight` derives weight from inventory and backpack contents, a carried scout, and attached objects. It writes the result with `SetStatus`; weight is calculated from possessions rather than healed like an exposure amount.

## Timed effects and restoration

An item reaches the timed-effect system through `Action_ApplyAffliction.RunAction`, which calls `CharacterAfflictions.AddAffliction`. The manager copies the supplied `Affliction` and searches for an existing object of the same type. A new instance receives the character reference and `OnApplied`; an existing instance receives `Stack` and `OnStacked`. Each effect class defines its own stacking behavior.

Separating a lasting condition from an effect that produces or modifies it is correct. Poison exposure and a timed source of poison need different lifetimes. Copying the effect supplied by an item also keeps the item's definition from becoming the live character's mutable timer.

`CharacterAfflictions.Update` invokes `UpdateEffectNetworked` on active effects and ticks their lifetime on the owning character. `Affliction.Tick` advances the lifetime and calls `UpdateEffect` while the effect remains active. For example, `Affliction_AdjustColdOverTime.UpdateEffect` submits a time-scaled request to `AdjustStatus`. On expiry, the manager calls `RemoveAffliction`, which invokes `OnRemoved`.

`UpdateNormalStatuses` requests natural poison recovery after its recovery delay. Those requests reach `SubtractStatus`. Its timed-effect clearing branch is selected by the reduction request and condition type, without excluding natural recovery through `decreasedNaturally`.

That branch calls `ClearPoisonAfflictions`. The method scans the effect list, removes matching objects, and calls `PushAfflictions` whether or not the scan found anything. `PushAfflictions` then copies the list, serializes it, and requests an immediate send for the local character. It has no unchanged-state check.

Consequently, repeated recovery requests can repeatedly serialize and send an unchanged collection, including an empty one. The committed status amount need not change on every request because its reduction is accumulated separately. The extra effect work is not limited to meaningful status increments or actual removals.

`ClearPoisonAfflictions` does pointless work and requests pointless sends. The code already knows how to batch statuses: `PushStatuses` marks a pending update and `ActuallyPushStatuses` drains it. Effect removal even has a way to suppress individual sends. Yet a scan that removes nothing still builds and serializes the collection through `PushAfflictions`. This is a wasteful update path with a straightforward fix: report whether removal changed anything, then schedule a send only when it did. There is no need to speculate about bandwidth to criticize allocating and serializing an unchanged list.

Ordinary statuses and timed effects have separate transmission paths. `PushStatuses` marks status data for a later send through `ActuallyPushStatuses`. `PushAfflictions` serializes and sends the effect collection immediately. Received status data goes from `SyncStatusesRPC` to `SetAll`; reconnect restoration also uses `SetAll` through `ApplyReconnectData`. `ReconnectHandler` subscribes to `StatusesUpdated` and creates a fresh record through `ReconnectData.CreateFromCharacter`.

The missing separation is between a gameplay request and an already-decided state. A damage request should pass through difficulty and immunity rules; restoring a recorded amount should establish that amount. Both currently reach the same mutation routines. Presentation and transmission are mixed into those routines as well, so a status change is difficult to reason about without following unrelated consumers.

Restoring a saved amount should not require putting it through gameplay a second time.

`SetAll` receives a collection of status amounts, compares them with the current collection, and routes the differences through `AddStatus` or `SubtractStatus`. That reuses ordinary gameplay behavior for restoration. The call path is shared by received status updates and `ApplyReconnectData`, so its semantics affect both remote representation and recovered state.

`AddStatus` still applies difficulty processing to affected types when called from this restoration path. Its network-origin flag changes some checks and sending behavior; it does not turn the operation into assignment. `Ascents.etcDamageMultiplier` can differ from its neutral value in custom runs. In that case, a difference calculated from an already-established status amount is transformed again before being committed. A restored value is being treated as a fresh gameplay request.

Reduction is not a plain correction either. `SubtractStatus` includes condition-specific behavior beyond changing one array entry, including interactions with other conditions and timed effects. Those behaviors are defensible for healing. They are a poor contract for a method whose job is to bring a collection into agreement with recorded state.

`SyncStatusesRPC` raises `StatusesUpdated` after `SetAll`, and `ReconnectHandler.UpdateReconnectData` builds a record from the character's resulting values. `StaminaBar.Update` and `BarAffliction.ChangeAffliction` likewise read the resulting character state. They do not retain a separate, exact copy of the incoming amounts to correct the mutation's interpretation.

`SetAll` is the wrong abstraction for restoration because it delegates assignment to gameplay mutation. A recorded amount should not be reinterpreted as new damage or healing under whatever rules happen to be active now. Restoration needs its own validated assignment path and a refresh of derived state. Reusing `AddStatus` and `SubtractStatus` here saves a separate implementation by giving up the one property restoration needs: agreement with the supplied state.

`SetStatus`, committed branches in `AddStatus` and `SubtractStatus`, and the stamina methods in `Character` call `GUIManager.instance.bar.ChangeBar` directly. `ChangeBar` updates the `BarAffliction` components using `Character.observedCharacter`, which can be a different scout from the object whose state changed. The gameplay owner reaches into a global presentation object to refresh whichever character is currently observed.

`AddStatus` owns much more than that refresh. It also handles web-wrap animation, struggle prompts, particle playback, sound selection, and biome-related achievement tracking. These are separate consumers of a status event, but their work is embedded in the status mutation. A change to status accounting must preserve scene references and presentation behavior even when the desired test concerns only amounts and caps. Isolated state tests require a surrounding Unity setup or replacement dependencies because the arithmetic is not isolated from the HUD.

The existing event does not solve this boundary. `StatusesUpdated` is raised by `ActuallyPushStatuses` and by `SyncStatusesRPC`; it is not the common completion event of the mutators. `ReconnectHandler` depends on that event to refresh recorded player state. Notification timing is therefore coupled to the send path for an owned character and to the receive path for a remote one. Changing when data is batched changes when this consumer is told about state.

`CharacterAfflictions` is overburdened, and the coupling actively obstructs testing. Arithmetic, HUD refresh, effects, network scheduling, and notification are tangled in the same manager. A completed local mutation should publish its resulting value; presentation, reconnect recording, and transmission should consume that event separately. Instead, changing send timing changes when another system learns that state changed. That is a bad dependency to bake into the core status model.

## Reading and changing this code

Keep accumulated statuses distinct from timed `Affliction` instances. Removing a producer and removing the status it has already produced are different operations. `AddAffliction` clones the supplied definition; its live instance is the one held in `afflictionList`, with the character reference assigned by the manager. Individual `Stack` implementations determine whether another application extends duration or changes an existing parameter.

Treat display and observation as separate concerns from ownership. `BarAffliction` reads the observed character, while the manager's mutation and sending checks concern the character holding the state. A visual extension should preserve that distinction. Names such as `SetAll` are not enough to infer assignment semantics; its consumers and the mutation methods establish the actual contract. No network invocation or state-alteration procedure is provided here.

## Open questions

The fractional-loss finding follows from the accumulator writes. Its size during ordinary play depends on the serialized rates and update timing, which have not been measured in the official Steam build. No particular exhaustion time or difference between frame rates has been established.

The restoration finding establishes that gameplay transformations remain in the path. The visible consequences under custom-run settings, and the frequency with which later updates reduce or repeat a discrepancy, still need official-build observation. This chapter does not establish a reported multiplayer symptom or a corrupt save.

The recovery path contains unchanged effect sends, but their bandwidth and frame-time impact have not been profiled. The event placement establishes a coupling to transmission; it does not by itself establish a lost reconnect record. These are distinct claims.
