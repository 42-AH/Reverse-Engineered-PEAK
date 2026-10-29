# Oddities and diagnostic code

Diagnostic tools are supposed to make the rest of the game easier to reason about. That is a modest request: show the current state, keep references current, and notice when work fails. Some of PEAK's helpers need that request spelled out more carefully.

Two early candidates illustrate the distinction. The secondary-use caller looked suspicious until its callee's guard was traced. The biome comparison has a demonstrable unequal-length problem, but its production consumer remains unidentified. Neither belongs in a list of observed player-facing bugs.

The diagnostic layer exposes views of item instance data, reconnect records, network accounting, and map-related state. These are different observation scopes. An inventory inspector reads current item slots; a reconnect inspector reads recovery records; a map display obtains information from the active scene and its services. Agreement between them is not automatic because they do not read one shared snapshot.

Authoring helpers also need to report whether asynchronous work has finished successfully. `ForgottenAwaitable` wraps work launched by `LevelGeneration` and `PropGrouper`, retaining a completion callback. Its result feeds validation handling, so the helper is part of error reporting rather than just a convenience for starting work.

## Inspecting item state

The item inspection path has a direct consumer chain:

- `GameHandler` registers a factory for `ItemInstanceDataDebugPage`.
- `ItemInstanceDataDebugPage.Update` reads ordinary item slots from the current players and collects their instance data.
- It creates `ItemInstanceDataUICell` objects keyed by instance identity.
- Each cell retains an `ItemInstanceData` reference and formats that object's entries during `Update`.

Building diagnostic pages against gameplay objects keeps their views close to the state being investigated. `ItemInstanceDataUICell` can show the same data object that an item uses, and `ReconnectDataDebugPage` can inspect the actual recovery record. The corresponding obligation is to define freshness and lifetime. A live reference is useful only while the tool knows which live object it refers to.

An inspector is supposed to clarify the current state. Accumulating old references is an odd way to help.

`ItemInstanceDataDebugPage.Update` builds a fresh list from current item slots, but only uses that list to add missing cells. It never removes cells whose identities have left those slots. It then updates every previously created cell. `ItemInstanceDataUICell` retains the data object supplied to its constructor and has no replacement operation.

The consumer therefore has two independent freshness problems. An item that leaves the inspected slots remains displayed, because the page does not compare existing cells against the new live set. If a current slot later receives another data object with the same identity, the existing cell prevents a new cell from being created and continues reading its earlier reference. Identity equality is being used as a substitute for reference refresh.

There is a concrete path capable of replacing the slot's data object: `Player.SyncInventoryRPC` obtains inventory state and calls `ItemSlot.SetItem`, which assigns the supplied instance data. The inspector does not subscribe to that replacement or rebind its cell during its next scan. Its label can consequently describe a retained object instead of the current slot object associated with the same identity.

This is a diagnostic correctness issue. An observer investigating whether an inventory update arrived can consult a page that still reads an older reference. The page also retains every cell it has discovered for its lifetime, so its work grows with historical discoveries rather than being bounded by the currently inspected inventory.

The item inspector is not maintaining a current view. It adds entries, keeps departed ones, and fails to rebind an existing identity to a replacement data object. Reconcile the live identity-to-data map: remove, add, and refresh references. If history is wanted, label it as history and timestamp it. Mixing a live scan with an unlabeled permanent collection makes the diagnostic misleading, which is worse than simply giving it a narrow scope.

Its coverage also needs a label. The scan visits ordinary player item slots; it does not recursively enumerate backpack contents or all world items. The better tool would state that scope or provide separately identified views. Expanding the title is cheaper than debugging a system from an assumed inventory that was never inspected.

## Recovery and scene diagnostics

`ReconnectDataDebugPage` has a different lifecycle. Each update obtains `ReconnectHandler`, builds text from its current record collection, and replaces the label contents. When no handler exists it shows that absence. The page reads retained recovery state, not the live character directly.

`MapDebugUI.Start` applies an editor/development-build eligibility check before initializing its display. An initialized instance subscribes to `SceneManager.sceneLoaded`; `OnSceneLoad` refreshes its references to scene audio objects. Its fields and methods demonstrate that diagnostic presentation participates in scene transitions, rather than merely drawing an isolated text label.

An initialized `MapDebugUI` subscribes `OnSceneLoad` to the static `SceneManager.sceneLoaded` event. The inspected class contains no matching unsubscription in a destruction or disable handler. `OnSceneLoad` performs scene searches and writes the instance's audio references.

The event publisher outlives an ordinary scene component. Without removing the subscription, destroying the component does not remove its delegate from that publisher's invocation list. A later scene-loaded notification can continue invoking the retained managed callback. A subsequent initialized display adds another subscriber rather than replacing the old one.

`MapDebugUI` registers a scene callback without owning its teardown. That retains instance state and can repeat scene searches after the display's intended lifetime. The fix belongs beside initialization: unregister the handler, and make repeated cleanup safe for partial setup. A diagnostic component does not get an exemption from lifecycle management just because it is used to inspect somebody else's code.

## Generation completion

Generation result handling follows another short path:

- `PropGrouper.Run` calls `RunAll` and passes the resulting operation to `ForgottenAwaitable.Wrap`.
- It also passes a completion callback that sets `ValidationState` to failed when work fails.
- `ForgottenAwaitable` either handles an already-completed operation immediately or registers `OnCompleted` for a pending one.
- `LevelGeneration.RunAsync` uses the same helper for its generation operation.

The shared awaitable wrapper similarly centralizes completion handling. Its consumers should not each need a separate exception-reporting loop. But centralizing that responsibility makes consistency between its immediate and deferred paths essential: callers receive the same success boolean from both.

Finishing quickly should not exempt an operation from having its result checked.

`ForgottenAwaitable.Wrap` checks whether an incoming operation is already completed. In that branch it clears its retained operation and invokes the callback with success. It does not retrieve the operation's result. The deferred branch behaves differently: `OnCompleted` retrieves the result inside a try/catch, invokes the appropriate callback, and records whether an exception occurred.

Completion and success are different facts. The immediate branch promotes the first into the second without the result check that the deferred branch performs. An operation already completed with an exception receives the success treatment at this boundary, whereas a pending operation that later faults reaches the failure handling.

`PropGrouper.Run` makes the consequence specific. Its callback marks validation failed when the helper reports failure. A failure classified as immediate success does not trigger that path. The reporting behavior depends on whether completion was observed before or after the wrapper examined it, rather than only on the operation's outcome.

The helper's exception flag also has asymmetric maintenance. The deferred path sets or clears `ThrewException`, but the immediate-success branch does not update it. A wrapper previously marked as having thrown can retain that flag while invoking a new success callback. The callback and stored diagnostic state then describe different outcomes.

`ForgottenAwaitable` is an unreliable success reporter. Already-completed work still needs its result observed; skipping that step lets a completed failure receive success treatment. Both branches must use one routine that retrieves the result, updates outcome state, and invokes the callback once. The current split makes timing determine whether an exception is recognized. That is particularly bad in a helper used by validation code: the tool can report success because it failed to look.

The source also contains command registration and specialized inspection helpers. Their presence establishes development infrastructure in the assembly, not universal availability through the retail interface. No activation procedure is needed to analyze their responsibilities.

## Reading and changing this code

Treat diagnostic output as a view with a defined source, not as an authoritative account of the entire game. The item page's slot scan, the reconnect page's recovery records, and the network page's counters answer different questions. Chapter 09 covers the network page's unit and sampling problems; those should be accounted for before comparing its displayed rates.

For lifecycle review, pair registration with teardown and pair completion with result observation. These checks apply to diagnostic code as directly as they apply to gameplay. A tool that keeps a stale reference can make a correctly updated system look broken.

## Open questions

The item-page and map-display mechanisms were inspected in source; no official-session display defect or accumulation measurement was recorded. `MapDebugUI`'s eligibility check limits which builds initialize this path. The scene lifetimes that repeatedly create and destroy eligible instances need confirmation before estimating retained objects or duplicate callback counts.

The immediate-completion issue in `ForgottenAwaitable` needs a focused test with successful and faulted operations that finish before wrapping, as well as operations that finish later. This pass establishes the missing result observation and its validation consumer, not the frequency of such failures in the authoring workflow.

`MapBaker.BiomeResult.IsIdenticalTo` compares entries only while both lists still have entries. Unequal-length lists with an identical shared prefix therefore compare equal. No direct caller of that overload was found in the inspected assembly source, so its operational consequence remains unestablished. It is not evidence that daily island selection compares or rejects maps incorrectly.

The secondary-use candidate is resolved: `CharacterItems.DoUsing` repeatedly reaches `StartUseSecondary` while input is held, but the callee guards active use. Chapter 04 records that result. The other confirmed gameplay findings remain in their owning chapters rather than being duplicated here.
