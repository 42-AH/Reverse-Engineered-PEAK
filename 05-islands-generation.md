# Daily islands and generation

Today's island comes from a prepared catalogue of scenes. The generation machinery runs separately, with its own structured stages and checks. Simply boarding the plane does not trigger island regeneration from a seed for every client. The code has real enough complications without adding that one.

The most significant issues centre on boundaries. Bundle loading loses its success result before scene loading tries to use it. Reading the daily countdown also kicks off refresh work. Generation uses shared random state and, in one placement path, accepts a candidate even when the requested ground probe fails. These mechanisms need different tests because they happen at different stages of content preparation and play.

`NextLevelService` keeps the current island index and the time until the next selection. `MapBaker` converts an index into its scene catalogue. `AirportCheckInKiosk.LoadIslandMaster` uses those services to pick the destination, with a fallback scene if the catalogue lookup returns nothing. `LoadingScreenHandler` loads the relevant bundle before opening the scene.

The source also includes tools for assembling content: seeded placement stages, biome markers and variants, constraints on spawned props, deferred generation steps, lighting bakes, and validation. These responsibilities go beyond choosing tomorrow's island. They explain how scene content gets prepared and checked, not just how it appears in the departure menu.

## Choosing and loading an island

The departure chain being examined is straightforward:

- `AirportCheckInKiosk.LoadIslandMaster` gets `NextLevelService.NextLevelIndexOrFallback`.
- `MapBaker.GetLevel` wraps the index into the scene catalogue and gives back the selected scene name.
- `AirportCheckInKiosk.BeginIslandLoadRPC` passes the destination to `LoadingScreenHandler.LoadSceneProcess`.
- `LoadingScreenHandler.EnsureBundleLoaded` calls `MapBundleService.LoadBundlesNearestIndex` to get the bundle and waits for its loading flag to clear.

`MapBundleService` finds local map bundles and pairs each with an index range. Despite what its name suggests, `LoadBundlesNearestIndex` picks a range containing the requested index rather than calculating a nearest available fallback. `MapBundleData.Load` starts the file load asynchronously and records whether the bundle was found. This result differs from scene activation, character spawning, or reaching playable terrain.

Using prepared scenes separates the destination from generation work. The departure code needs the catalogue entry and its bundle. It does not need to replay every prop placement decision. Generation can include lighting, deferred work, and validation stages without putting those duties in the kiosk.

The loading flag has served its purpose. The bundle still needs to exist.

`MapBundleData.Load` returns a success value. If the file load fails, it marks the bundle as unloaded, clears the loading flag, and returns failure. `LoadBundlesNearestIndex` starts that work but gives back the bundle metadata without passing the task result to its caller.

`LoadingScreenHandler.EnsureBundleLoaded` then waits until `bundle.Loading` turns false. Both success and failure meet that condition. It also exits if no bundle is found. Its caller moves to scene loading without a result confirming the bundle is ready.

The loader knows what went wrong but forgets that at the next boundary. The code detects a missing or unreadable bundle inside the loader, yet the scene loading layer cannot tell it apart from an available bundle based on what it checks. The next step is told to load the named scene anyway. A failure that could have been caught as a missing requirement becomes a scene loading problem later on.

`MapBundleData.Load` has another contract issue: when loading is already running, a second call returns success right away. Success can mean either loaded or just already loading. The current loading screen client works around this by separately watching the flag, but the method does not give other callers a reliable completion contract.

The loading contract is broken. Callers piece together readiness from a return value that means different things and a separate loading flag. `MapBundleData.Load` should show the same in flight operation and eventual result to all callers. `LoadingScreenHandler` should only move forward with a usable bundle. Logging a failure and letting the dependent operation run anyway is not error handling. It just moves the failure somewhere harder to diagnose.

## The daily countdown

Daily timing starts with a response stored by `NextLevelService.NewData`. `NextLevelData` tracks when that response arrived and subtracts elapsed real time when the countdown is read. If no response has come in, `NextLevelIndexOrFallback` uses a locally computed date based selection. `NextLevelUI.Update` reads the right countdown and formats it for the menu. The date based fallback and the response backed selection feed into the same catalogue lookup.

A countdown label has somehow taken on a role in service recovery.

`NextLevelData.SecondsLeft` is a property, but when time runs out it calls `CloudAPI.CheckVersion`. The callback updates island data and asks for the bundle. `NextLevelUI.Update` reads this property every frame, so displaying the countdown also schedules service work.

There is a safeguard in place: the getter checks `QueryingGameTimeStatus`, and `CloudAPI.CheckVersion` sets it before making the request. This stops every expired frame from starting another request simultaneously. The problem comes at the end. `CloudAPI.CheckVersion` clears the status when it finishes. On a connection error it does not run the response callback. The data stays expired, so the next UI read can trigger another refresh. There is no retry delay in this path other than how long the previous request took plus the time until the next read.

A menu timer controls retry timing for a service that also affects content choice and bundle setup. Adding another display reader or calling `ToString`, which also checks `SecondsLeft`, can start the same service work. A method that looks like it just reads a value can cause a request and later content loading.

Hiding refresh and retry logic inside a countdown read is poor API design. `NextLevelService` should handle the request, its in flight state, retry timing, and stale data rules. A label should just ask for a number. Instead, adding a display consumer can change service behaviour, and testing recovery means tracking which UI happens to read the property. A getter that triggers this work hides important side effects behind a simple read.

## Generation order and placement

There are two generation entry points to identify. `MapGenerator.GenerateAll` can optionally set Unity's random state and runs its active `MapGenerationStage` components in the order they appear. Each stage clears its earlier output, makes candidate nodes, shifts probabilities through positional and proximity passes, then tries to place objects. The proximity passes check objects from earlier stages, so stage order has spatial importance.

`LevelGeneration.Generate` works differently. It clears through `PropGrouper`, syncs transforms, sets the seed, optionally bakes lighting, and waits for `PropGrouper.RunAll`. After that it refreshes the fake item list and calls `MapHandler.DetectBiomes`.

`PropGrouper.RunAll` collects `LevelGenStep` components and splits them by whether their group runs early or late. It runs early steps, does the right deferred work, optionally does an intermediate light bake, then runs late steps and their deferred work. Next it calls `ILevelGenFinalizer` components, optionally checks the result, and does the final light bake if asked. Its async path pauses between work portions. The sync or batch path skips those frame waits.

The staged prop pipeline reveals dependencies clearly. Early placement can establish surfaces or objects that later placement will need, and deferred work can remain pending until its group finishes. Validation uses its own result type instead of relying solely on exceptions, which proves valuable when a generation step completes but the content it produces still violates a constraint.

Both `MapGenerator.GenerateAll` and `LevelGeneration.Generate` initialize Unity's shared random state. `MapGenerationStage` then uses that same stream for offsets, placement probability, and object variation. `MapGenerator` doesn't create a separate stream for each stage or reset the prior state when done.

This creates coupled reproducibility. An early stage's random choices alter the state that later stages receive, even stages whose configuration hasn't changed. Reordering stages carries both the expected spatial effect of changing their prerequisites and an additional effect of shifting their random input. A seed only identifies the result when paired with the sequence of consumers that consume the stream.

`LevelGeneration` introduces an asynchronous path through `PropGrouper`. Since generation can yield between work portions while using shared state, other random consumers sit outside the generation API's isolation. The contract offers no private stream reserved for this generation run alone.

The shared random stream unnecessarily couples generation stages. Each stage should receive its own generator derived from the root seed plus a stable identifier, so changing one stage's random draw count stops affecting unrelated stages. Spatial order still matters; there's no need to pretend it doesn't. What remains avoidable is making reproducibility hinge on every prior consumer's random bookkeeping. The current design makes diagnosing local generation changes harder than necessary.

The prop layer does more than provide a random prefab list. `PropSpawner` extends `LevelGenStep` and exposes placement modifiers, constraints, post-placement constraints, and post-spawn behaviors. `PropGrouper.DoValidation` gathers results from child `IValidatable` implementations and keeps failed or unknown results. Execution ordering and validation are separate; completing a step doesn't guarantee the generated content passed every applicable check.

`MapGenerationStage.TrySpawnObject` ties together the ground probe and raycast success in one conditional. Height limits, allowed surface tags, and alignment to the hit surface get evaluated within that same conditional. If the probe fails, that block is skipped. The method then reaches its standard validity and probability check with the original candidate position still at hand.

This creates a specific gap between configuration and placement. When downward placement is requested, a missed surface doesn't itself reject the candidate. It can still reach `SpawnObject` at the stage's starting height. Surface-tag restrictions also require a successful hit before evaluation. Missing a surface therefore differs from hitting a forbidden surface: the latter rejects the node, the former doesn't.

`SpawnObject`, the consumer, takes the supplied position and normal, creates the object, and logs it with the stage's output. Later proximity passes can reference that output. A failed grounding constraint can affect both that object's placement and subsequent stages treating it as an established spatial reference.

A missed supporting-surface probe should reject a surface-dependent candidate. Proceeding as if the candidate remained valid is a failed validation boundary. If free-space placement is desired, make it an explicit mode rather than the unintended result of a failed query. Otherwise the placement stage cannot clearly separate intentional output from output that followed a prerequisite failure.

## Biome discovery

Biome discovery follows scene structure. `MapHandler.DetectBiomes` examines active biome components at a specific hierarchy depth and notes their types. `LevelGeneration.RandomizeBiomeVariants` holds a variant-selection routine, while `MapBaker` carries biome metadata with scene paths. None of these names should replace the actual departure chain discussed above.

The implementation relies on several implicit contracts: bundle completion signals readiness, scene hierarchy determines biome discovery, and a shared random stream connects stages. These contracts restrict changes far beyond the method being edited.

## Reading and changing this code

Before treating a map name, daily index, or seed as equivalent, identify which layer is changing. Catalogue selection, bundle availability, prepared scene content, and generation inputs serve different consumers. A prop-generation change calls for placement and validation checks; a catalogue change calls for scene and bundle consistency checks.

Within the generation machinery, maintain timing relationships alongside component references. A deferred step may depend on its group's output, and biome detection depends on active hierarchy structure. When investigating reproducibility, record stage order and generator configuration together with the seed. This addresses compatibility, not a reason to alter the game's online checks.

## Open questions

The inspected source establishes the kiosk's prepared-scene departure path. It does not establish which generation entry points participate in the production authoring workflow for every shipped island. `RandomizeBiomeVariants` has no direct caller in the inspected assembly source. Serialized invocation, editor-only tooling, and the provenance of individual prepared scenes require separate evidence.

The failure-path findings above have not been reproduced in an official Steam session. The frequency of bundle failures, expired-countdown retries, or surface-probe misses in shipped content is unknown. The staged placement analysis establishes what the method does when a probe misses; it does not establish that a particular floating prop in the game came from this component.

`MapBaker.BiomeResult.IsIdenticalTo` compares the shared prefix of two biome lists without checking equal length, but no production consumer of that comparison was found in this pass. Its practical effect belongs in a later authoring-tool trace, rather than being presented as a daily-island defect.
