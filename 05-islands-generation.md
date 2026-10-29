# Daily islands and generation

Today's island is selected from a catalogue of prepared scenes. The generation machinery exists elsewhere, with its own ordered stages and validation. Boarding the plane does not, by itself, mean every client regenerates an island from a seed. The code has enough real complications without inventing that one.

The most consequential findings concern boundaries. Bundle loading loses its success result before scene loading consumes it. Reading the daily countdown also initiates refresh work. Generation uses shared random state and, in one placement path, accepts a candidate even when the requested ground probe misses. These mechanisms deserve different tests because they occur at different stages of content preparation and play.

`NextLevelService` retains the current island index and the time remaining until the next selection. `MapBaker` maps an index into its scene catalogue. `AirportCheckInKiosk.LoadIslandMaster` uses those services to choose the destination, with a fallback scene when the catalogue lookup produces no name. `LoadingScreenHandler` prepares the relevant bundle before loading the scene.

The source also contains tools for assembling content: seeded placement stages, biome markers and variants, constraints on spawned props, deferred generation steps, lighting bakes, and validation. These responsibilities are broader than choosing tomorrow's destination. They describe how scene content can be prepared and checked, not just how it is presented in the departure menu.

## Choosing and loading an island

The inspected departure chain is explicit:

- `AirportCheckInKiosk.LoadIslandMaster` obtains `NextLevelService.NextLevelIndexOrFallback`.
- `MapBaker.GetLevel` wraps the index into the scene catalogue and returns the selected scene name.
- `AirportCheckInKiosk.BeginIslandLoadRPC` hands the destination to `LoadingScreenHandler.LoadSceneProcess`.
- `LoadingScreenHandler.EnsureBundleLoaded` asks `MapBundleService.LoadBundlesNearestIndex` for the containing bundle and waits for its loading flag to clear.

`MapBundleService` discovers local map bundles and associates each with an index range. Despite its name, `LoadBundlesNearestIndex` selects a range containing the requested index; it does not calculate a nearest available alternative. `MapBundleData.Load` starts the asynchronous file load and records whether the resulting bundle exists. That result is distinct from scene activation, character spawning, or reaching playable terrain.

Selecting prepared scenes decouples the destination from generation work. The departure code needs the catalogue entry and its bundle; it does not need to replay every prop-placement decision. Generation can have lighting, deferred work, and validation stages without putting those responsibilities inside the kiosk.

The loading flag has finished its job. The bundle still has to exist.

`MapBundleData.Load` returns a success value. On a failed file load it records that the bundle is not loaded, clears the loading flag, and returns failure. `LoadBundlesNearestIndex` starts that task but returns the bundle metadata object without carrying the task's result to its caller.

`LoadingScreenHandler.EnsureBundleLoaded` then waits only until `bundle.Loading` is false. Both successful completion and failed completion satisfy that condition. The routine also exits when no bundle matches. Its consumer proceeds to scene loading without a successful bundle-readiness result from this boundary.

The loader knows what went wrong, then loses that information at the next boundary. A missing or unreadable required bundle is detected inside the loader, but the scene-loading layer cannot distinguish it from an available bundle using the information it checks. The next operation is asked to load the named scene anyway. A failure that could have been handled as an unavailable prerequisite becomes a later scene-loading problem.

`MapBundleData.Load` has a related contract problem: when loading is already in progress, a second call returns success immediately. That means success can mean either loaded or merely already being loaded. The current loading-screen consumer compensates by separately polling the flag, but the method itself does not provide a reliable completion contract to another caller.

The service should retain the in-flight task and give every caller the same eventual result. Scene loading should require a positive readiness result, with missing-range and load-failure outcomes handled explicitly. This would preserve the distinction between requesting work, finishing work, and producing a usable bundle. Logging the error inside `Load` is not a substitute for returning that distinction to the operation that depends on it.

## The daily countdown

Daily timing starts with a response stored by `NextLevelService.NewData`. `NextLevelData` remembers when that response was received and subtracts elapsed real time when its countdown is read. If no response has been received, `NextLevelIndexOrFallback` uses a locally calculated date-based selection. `NextLevelUI.Update` reads the appropriate countdown and formats it for the menu. The date-based fallback and the response-backed selection are two inputs to the same catalogue lookup.

A countdown label has somehow acquired a job in service recovery.

`NextLevelData.SecondsLeft` is a property getter, but when time has expired it calls `CloudAPI.CheckVersion`. Its callback updates island data and requests the relevant bundle. `NextLevelUI.Update` reads that property every frame, so a display read is also a scheduler for service work.

There is an in-flight guard: the getter checks `QueryingGameTimeStatus`, and `CloudAPI.CheckVersion` sets it before sending the request. This prevents the simplistic claim that every expired frame starts another simultaneous request. The problem appears at completion. `CloudAPI.CheckVersion` clears the status when the operation finishes; on a connection error it does not invoke the response callback. The expired data therefore remains expired, and the next UI read can request another refresh. There is no retry delay in this traced path beyond the duration of the preceding request and the next read.

A menu timer determines retry cadence for a service that also influences content selection and bundle preparation. Adding another display consumer or calling `ToString`, which also reads `SecondsLeft`, can initiate the same service work. A method that appears to inspect a value can cause a request and later content-loading activity.

Refresh should belong to `NextLevelService`, with an explicit in-flight operation, a next-attempt time, and a policy for stale data. The countdown accessor should calculate a display value. Separating these responsibilities would allow the UI to refresh frequently without implicitly controlling network retry behavior, and would make failed-refresh tests independent of whether a menu happens to be visible.

## Generation order and placement

There are also two identifiable generation entry points. `MapGenerator.GenerateAll` optionally initializes Unity's random state and runs its active `MapGenerationStage` components in list order. Each stage clears earlier output, creates candidate nodes, adjusts probabilities through positional and proximity passes, then attempts to place objects. Its proximity passes consult objects produced by earlier stages, so stage order carries spatial meaning.

`LevelGeneration.Generate` instead clears through `PropGrouper`, synchronizes transforms, initializes the seed, optionally bakes lighting, and awaits `PropGrouper.RunAll`. Afterward it refreshes the fake-item list and calls `MapHandler.DetectBiomes`.

`PropGrouper.RunAll` gathers `LevelGenStep` components and separates them according to their group's early or late timing. It executes early steps, runs the applicable deferred work, optionally performs an intermediate light bake, then executes late steps and their deferred work. Finally it invokes `ILevelGenFinalizer` components, optionally validates the result, and performs the final light bake when requested. Its asynchronous path yields between portions of work; the synchronous or batch path avoids those frame waits.

The staged prop pipeline also makes dependencies visible. Early placement can establish surfaces or objects needed by later placement, and deferred work can wait until a group is complete. Validation has its own result type rather than relying only on exceptions. That distinction is useful when a generation step completes but produces content that still fails a constraint.

Both `MapGenerator.GenerateAll` and `LevelGeneration.Generate` initialize Unity's shared random state. `MapGenerationStage` then consumes that same stream for offsets, placement probability, and object variation. `MapGenerator` does not derive a separate stream for each stage or restore the prior state when it finishes.

The cost is coupled reproducibility. Changing how many random choices an early stage makes changes the state seen by later stages, including stages whose own configuration was untouched. Reordering stages has both the intentional spatial effect of changing their prerequisites and the additional effect of shifting their random input. A seed identifies the result only together with the sequence of consumers that advance the stream.

`LevelGeneration` adds an asynchronous execution path through `PropGrouper`. Because generation can yield between portions of work while using shared state, other random consumers are not isolated by the generation API. The contract provides no private stream that belongs exclusively to this generation run.

Independent random generators, derived from a root seed and stable stage identifiers, would narrow the effect of a local change. Each stage could then reproduce its own decisions without depending on how many random draws an unrelated earlier step happened to make. Spatial dependencies would still need to be preserved; separating random streams does not remove the fact that proximity passes consume prior placement. It removes an extra dependency that makes diagnosing changed output harder.

The prop layer provides more than a random prefab list. `PropSpawner` derives from `LevelGenStep` and exposes placement modifiers, constraints, post-placement constraints, and post-spawn behaviors. `PropGrouper.DoValidation` collects results from child `IValidatable` implementations and retains failed or unknown results. Execution ordering and validation are separate concerns: finishing a step does not itself establish that the generated content passed every applicable check.

`MapGenerationStage.TrySpawnObject` combines the need for a ground probe with the success of the raycast in one conditional. Height limits, allowed surface tags, and alignment to the hit surface are evaluated inside that conditional. If the probe misses, that block is skipped. The method then reaches its ordinary validity and probability check with the original candidate position still available.

This produces a specific mismatch between configuration and placement. With downward placement requested, a missed surface does not by itself invalidate the candidate. It can still reach `SpawnObject` at the stage's initial height. Surface-tag restrictions also depend on a successful hit before they are checked. The absence of a surface is consequently different from hitting a disallowed surface: the latter rejects the node, while the former does not.

The consumer, `SpawnObject`, accepts the supplied position and normal, creates the object, and records it among the stage's output. Later proximity passes can then consult that output. A missed grounding constraint can affect both the individual object's placement and subsequent stages that treat it as an established spatial reference.

The stage should distinguish placement modes before evaluating candidates. If a mode requires a supporting surface, a missed probe should produce a failed candidate with a recorded reason. If free-space placement is intended, it should be an explicit mode rather than the accidental continuation of a failed surface query. This also gives validation something useful to count: rejected candidates and the constraints that rejected them.

## Biome discovery

Biome detection follows scene structure. `MapHandler.DetectBiomes` inspects active biome components at a particular hierarchy depth and records their types. `LevelGeneration.RandomizeBiomeVariants` contains a variant-selection routine, while `MapBaker` carries biome metadata alongside scene paths. None of those names should be substituted for the actual departure chain above.

The implementation nevertheless uses several implicit contracts: bundle completion is treated as readiness, scene hierarchy determines biome discovery, and a shared random stream connects stages. Those contracts constrain changes far beyond the method being edited.

## Reading and changing this code

Identify the layer being changed before treating a map name, daily index, or seed as interchangeable. Catalogue selection, bundle availability, prepared scene content, and generation inputs have different consumers. A prop-generation change needs placement and validation checks; a catalogue change needs scene and bundle consistency checks.

Within the generation machinery, preserve timing relationships as well as component references. A deferred step may depend on the output of its group, and biome detection depends on active hierarchy structure. For reproducibility investigations, record stage order and the generator configuration alongside the seed. This is a compatibility concern, not a reason to change the game's online checks.

## Open questions

The inspected source establishes the kiosk's prepared-scene departure path. It does not establish which generation entry points participate in the production authoring workflow for every shipped island. `RandomizeBiomeVariants` has no direct caller in the inspected assembly source. Serialized invocation, editor-only tooling, and the provenance of individual prepared scenes require separate evidence.

The failure-path findings above have not been reproduced in an official Steam session. The frequency of bundle failures, expired-countdown retries, or surface-probe misses in shipped content is unknown. The staged placement analysis establishes what the method does when a probe misses; it does not establish that a particular floating prop in the game came from this component.

`MapBaker.BiomeResult.IsIdenticalTo` compares the shared prefix of two biome lists without checking equal length, but no production consumer of that comparison was found in this pass. Its practical effect belongs in a later authoring-tool trace, rather than being presented as a daily-island defect.
