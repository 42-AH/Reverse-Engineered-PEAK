# Stamina and afflictions

Tracing the stamina bar leads to two different systems inside `CharacterAfflictions`: accumulated status amounts and timed effect objects. Both influence the scout, but they are stored, updated, and synchronized differently. Following the mutations through to the HUD and reconnect records exposes structural problems: lost fractional updates, restoration mixed with gameplay rules, and notifications tied to transmission rather than to a completed state change.

## What it does

Stamina pays for movement and climbing. Hunger, injury, and other conditions reduce the available regular bar, while bonus stamina occupies a separate pool. Some conditions recede; others require an item or another intervention. Weight reflects carried objects and scouts. Embedded hazards contribute both visible attachments and status costs. Timed effects can change those processes without being another occupied section of the regular bar. When accumulated status fills that bar, `Character.HandleLife` progresses toward unconsciousness; petrification has a separate route through the same broader life-state machinery.

## How it's built

### Stamina and its consumers

`CharacterData.currentStamina` and `extraStamina` hold the two pools. `Character.UseStamina` is a shared consumption entry point, called by movement and equipment code. `CharacterMovement.SetMovementState` requests stamina while sprinting. `CharacterClimbing.Update` derives a climbing request from movement, wall angle, and modifiers before calling the same method. Rope and vine controllers also request stamina, but use the result to decide whether their attachment can continue.

Regular recovery takes a different path:

- `Character.FixedUpdate` calls `UpdateVariablesFixed`, which advances the timers used by recovery.
- `UpdateVariablesFixed` asks `CanRegenStamina` whether the scout can recover, accounting for recent use and grounding as well as attachment exceptions.
- `AddStamina` changes the owned scout's regular pool and calls `ClampStamina`.
- `ClampStamina` obtains its ceiling from `GetMaxStamina`, which reads `CharacterAfflictions.statusSum`.

Bonus stamina is added and clamped separately. `SetExtraStamina` and `AddExtraStamina` consult `CharacterData.petrifyAmount`; `CharacterData.SetPetrify` reclamps the bonus pool when that amount changes. Petrification is named in the status enum, but its value lives outside the ordinary status array. `BarAffliction.ChangeAffliction` knows about that exception when choosing the value to display.

### Status accumulation

`CharacterAfflictions.InitStatusArrays` creates `currentStatuses` and parallel arrays for pending additions, pending reductions, and timestamps. The enum selects array positions. `statusSum` adds the committed amounts; pending fractions are separate. `Character.HandleLife` reads `shouldPassOut`, while `StaminaBar.Update` reads the committed sum and maximum stamina to size the bar and its overflow presentation.

`AddStatus` handles eligibility and gameplay transformations before accumulation. These include character form, difficulty modifiers, and interactions between opposing temperatures. Committed changes clamp regular stamina and trigger presentation work. `SubtractStatus` has its own accumulator and side effects, including release from trapping conditions. `SetStatus` is a separate setter that clears pending changes for the selected status. `AdjustStatus` dispatches to addition or subtraction according to the requested direction.

The ordinary update loop feeds those entry points. `UpdateNormalStatuses` adds hunger while the local scout is conscious and requests timed recovery for eligible conditions. `UpdateWeight` derives weight from inventory and backpack contents, a carried scout, and attached objects. It writes the result with `SetStatus`; weight is calculated from possessions rather than healed like an exposure amount.

### Timed effects and restoration

An item reaches the timed-effect system through `Action_ApplyAffliction.RunAction`, which calls `CharacterAfflictions.AddAffliction`. The manager copies the supplied `Affliction` and searches for an existing object of the same type. A new instance receives the character reference and `OnApplied`; an existing instance receives `Stack` and `OnStacked`. Each effect class defines its own stacking behavior.

`CharacterAfflictions.Update` invokes `UpdateEffectNetworked` on active effects and ticks their lifetime on the owning character. `Affliction.Tick` advances the lifetime and calls `UpdateEffect` while the effect remains active. For example, `Affliction_AdjustColdOverTime.UpdateEffect` submits a time-scaled request to `AdjustStatus`. On expiry, the manager calls `RemoveAffliction`, which invokes `OnRemoved`.

Ordinary statuses and timed effects have separate transmission paths. `PushStatuses` marks status data for a later send through `ActuallyPushStatuses`. `PushAfflictions` serializes and sends the effect collection immediately. Received status data goes from `SyncStatusesRPC` to `SetAll`; reconnect restoration also uses `SetAll` through `ApplyReconnectData`. `ReconnectHandler` subscribes to `StatusesUpdated` and creates a fresh record through `ReconnectData.CreateFromCharacter`.

## Design decisions

Separating a lasting condition from an effect that produces or modifies it is correct. Poison exposure and a timed source of poison need different lifetimes. Copying the effect supplied by an item also keeps the item's definition from becoming the live character's mutable timer.

The missing separation is between a gameplay request and an already-decided state. A damage request should pass through difficulty and immunity rules; restoring a recorded amount should establish that amount. Both currently reach the same mutation routines. Presentation and transmission are mixed into those routines as well, so a status change is difficult to reason about without following unrelated consumers.

## Where it's weak

### Quantization discards the remaining amount

`AddStatus` accumulates small requests in `currentIncrementalStatuses` until there is enough for a committed increment. It commits whole increments, then clears the accumulator completely. Any remainder above those whole increments is discarded. `SubtractStatus` does the same to `currentDecrementalStatuses` after committing a reduction.

Accumulation is supposed to let small updates contribute without forcing every update into the displayed representation. Clearing the whole accumulator breaks that accounting. The result depends on how exposure or recovery is divided into requests, not only on its total amount. Different request sizes leave different discarded remainders.

This is connected to real producers. `UpdateNormalStatuses` submits time-scaled hunger and recovery requests. `Affliction_AdjustColdOverTime.UpdateEffect` also multiplies its rate by elapsed frame time before reaching `AdjustStatus`. Their integration passes through the lossy accumulators. The multiplication by elapsed time does not undo a remainder discarded later.

The consumers are regular stamina and life state. Committed status amounts feed `statusSum`, `Character.GetMaxStamina`, and `shouldPassOut`. Lost fractions therefore affect more than animation smoothness in `StaminaBar`. Keeping the fractional remainder after committing a whole increment would preserve the amount already accrued. A separate continuous simulation value with quantized presentation would also make the distinction explicit. The current code quantizes state by throwing part of it away.

### Restoration repeats gameplay processing

`SetAll` receives a collection of status amounts, compares them with the current collection, and routes the differences through `AddStatus` or `SubtractStatus`. That reuses ordinary gameplay behavior for restoration. The call path is shared by received status updates and `ApplyReconnectData`, so its semantics affect both remote representation and recovered state.

`AddStatus` still applies difficulty processing to affected types when called from this restoration path. Its network-origin flag changes some checks and sending behavior; it does not turn the operation into assignment. `Ascents.etcDamageMultiplier` can differ from its neutral value in custom runs. In that case, a difference calculated from an already-established status amount is transformed again before being committed. A restored value is being treated as a fresh gameplay request.

Reduction is not a plain correction either. `SubtractStatus` includes condition-specific behavior beyond changing one array entry, including interactions with other conditions and timed effects. Those behaviors are defensible for healing. They are a poor contract for a method whose job is to bring a collection into agreement with recorded state.

The inconsistency reaches concrete consumers. `SyncStatusesRPC` raises `StatusesUpdated` after `SetAll`, and `ReconnectHandler.UpdateReconnectData` builds a record from the character's resulting values. `StaminaBar.Update` and `BarAffliction.ChangeAffliction` likewise read the resulting character state. They do not retain a separate, exact copy of the incoming amounts to correct the mutation's interpretation.

A dedicated restoration operation could validate the collection, establish committed values, and refresh derived state without replaying damage or healing policy. Ordinary gameplay requests could keep their existing checks. These are different operations and need different contracts. Reusing the mutation API here makes agreement depend on which gameplay rules are active during restoration.

### Recovery sends unchanged effect collections

`UpdateNormalStatuses` requests natural poison recovery after its recovery delay. Those requests reach `SubtractStatus`. Its timed-effect clearing branch is selected by the reduction request and condition type, without excluding natural recovery through `decreasedNaturally`.

That branch calls `ClearPoisonAfflictions`. The method scans the effect list, removes matching objects, and calls `PushAfflictions` whether or not the scan found anything. `PushAfflictions` then copies the list, serializes it, and requests an immediate send for the local character. It has no unchanged-state check.

Consequently, repeated recovery requests can repeatedly serialize and send an unchanged collection, including an empty one. The committed status amount need not change on every request because its reduction is accumulated separately. The extra effect work is not limited to meaningful status increments or actual removals.

This contradicts the batching already used for statuses: `PushStatuses` merely sets a pending flag, which `CharacterAfflictions.Update` drains through `ActuallyPushStatuses`. Effect removal already supports suppressing individual sends during a batch, but `ClearPoisonAfflictions` still issues its final send when the batch removed nothing. The concrete cost is unnecessary list construction, serialization, and transmission requests. A removal result should decide whether the effect collection became dirty, with one scheduled notification after actual changes. No bandwidth measurement is needed to establish that the unchanged work occurs on this path.

### State changes require presentation, and notification requires sending

`SetStatus`, committed branches in `AddStatus` and `SubtractStatus`, and the stamina methods in `Character` call `GUIManager.instance.bar.ChangeBar` directly. `ChangeBar` updates the `BarAffliction` components using `Character.observedCharacter`, which can be a different scout from the object whose state changed. The gameplay owner reaches into a global presentation object to refresh whichever character is currently observed.

`AddStatus` owns much more than that refresh. It also handles web-wrap animation, struggle prompts, particle playback, sound selection, and biome-related achievement tracking. These are separate consumers of a status event, but their work is embedded in the status mutation. A change to status accounting must preserve scene references and presentation behavior even when the desired test concerns only amounts and caps. Isolated state tests require a surrounding Unity setup or replacement dependencies because the arithmetic is not isolated from the HUD.

The existing event does not solve this boundary. `StatusesUpdated` is raised by `ActuallyPushStatuses` and by `SyncStatusesRPC`; it is not the common completion event of the mutators. `ReconnectHandler` depends on that event to refresh recorded player state. Notification timing is therefore coupled to the send path for an owned character and to the receive path for a remote one. Changing when data is batched changes when this consumer is told about state.

The better option is a completed local mutation event carrying the changed status and its resulting value. The HUD, effects, and reconnect recorder can observe that event, while transmission remains a separate subscriber or scheduled operation. That would also allow accounting tests without constructing the game's presentation layer. Here, the manager owns the state, its effects, and the announcement that other systems need to keep their records current.

## For modders

Keep accumulated statuses distinct from timed `Affliction` instances. Removing a producer and removing the status it has already produced are different operations. `AddAffliction` clones the supplied definition; its live instance is the one held in `afflictionList`, with the character reference assigned by the manager. Individual `Stack` implementations determine whether another application extends duration or changes an existing parameter.

Treat display and observation as separate concerns from ownership. `BarAffliction` reads the observed character, while the manager's mutation and sending checks concern the character holding the state. A visual extension should preserve that distinction. Names such as `SetAll` are not enough to infer assignment semantics; its consumers and the mutation methods establish the actual contract. No network invocation or state-alteration procedure is provided here.

## Open questions

The fractional-loss finding follows from the accumulator writes. Its size during ordinary play depends on the serialized rates and update timing, which have not been measured in the official Steam build. No particular exhaustion time or difference between frame rates has been established.

The restoration finding establishes that gameplay transformations remain in the path. The visible consequences under custom-run settings, and the frequency with which later updates reduce or repeat a discrepancy, still need official-build observation. This chapter does not establish a reported multiplayer symptom or a corrupt save.

The recovery path contains unchanged effect sends, but their bandwidth and frame-time impact have not been profiled. The event placement establishes a coupling to transmission; it does not by itself establish a lost reconnect record. These are distinct claims.
