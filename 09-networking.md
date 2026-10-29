# Multiplayer sessions and replication

A session interface, matchmaking interface, and named connection state machine exist alongside gameplay code that reaches straight into Photon. PEAK has separated some of its networking responsibilities, making the gaps in that separation particularly obvious.

Rather than treating multiplayer as one host-controlled loop, this chapter follows those responsibilities. Its main findings concern a split abstraction boundary, motion correction without a sample timeline, and instrumentation that mixes units. These carry different consequences: maintenance coupling, a constrained smoothing model, and misleading diagnostic comparisons.

Steam lobby integration handles party discovery and invitation management. Photon supplies the session implementation exposed by `NetCode.Session`. `NetworkingUtilities.MAX_PLAYERS` defaults to four, and both the room options and Steam lobby creation reference that value. The default room size becomes shared configuration rather than a number embedded independently in each creation path.

Once players are together, continuous state and discrete actions take separate routes. Character and physics syncers publish changing state. Inventory changes and gameplay events go through explicit operations. Reconnect handling saves character and run-related records so a returning player can be distinguished from a new one. Joining a room, getting a character, and receiving the latest state happen in distinct stages.

## Sessions and matchmaking

`Peak.Network.NetCode` creates a `PhotonShim` for sessions and a `SteamLobbyAPI` for Steam matchmaking. `PhotonShim` exposes readiness, room membership, host identity, and connection events. During initialization it enables scene synchronization and configures the backend for the game's version before connecting.

The connection path has several named layers.

- `ConnectionService` registers states including `HostState`, `JoinSpecificRoomState`, `InRoomState`, and `DisconnectingState`.
- `NetworkConnector.HandleConnectionState` interprets the current state and performs the corresponding room operation.
- `NetworkingUtilities.HostRoomOptions` supplies the room configuration for hosting.
- `NetworkConnector` receives room-creation, join, and failure callbacks and advances the surrounding flow.

`SteamLobbyAPI` delegates lobby work to `SteamLobbyHandler`, keeps pending invitations, and exposes matchmaking events. Its `CreateLobby` uses the same player limit as room creation. Steam lobby membership and Photon room membership remain separate; neither proves that all gameplay objects have arrived.

Separating matchmaking from sessions works because an invitation and a replicated world are distinct problems. `ConnectionService` also gives connection intent a named state rather than forcing every callback to guess whether the player was hosting, joining, or leaving.

The interface looks clean until gameplay code needs to use it.

`NetCode.Session` and `ISessionAPI` provide an interface, yet `NetworkConnector.HandleConnectionState` still calls Photon room and connection operations directly. `CharacterSyncer`, `ItemPhysicsSyncer`, `RopeSyncer`, and `Player` all depend on Photon views, callbacks, or serialization. Even `ISessionAPI` exposes Photon's authentication type in its public contract.

This is a partial abstraction with concrete maintenance costs. A change to session lifecycle or ownership semantics cannot stay within `PhotonShim`; the connection controller and multiple gameplay components need review too. Substituting a test implementation of `ISessionAPI` would not intercept direct calls from those consumers. Tests of connection intent would still hit transport-specific operations outside the adapter.

The problem is not simply that a third-party library exists. Direct dependencies can be a deliberate choice. The issue is that the code defines a session boundary narrower than its name, while host and membership checks pull from either the interface or the backend. A reviewer must know which side of that boundary each subsystem actually lives on.

This abstraction falls short of where it needs to go. Either route connection intent and room lifecycle through the session boundary and give replication its own owner, or accept that the interface wraps only a handful of services. The current compromise leaves callers picking between the adapter and Photon directly. It complicates substitution and isolated tests without actually constraining transport dependencies. The backend can support the game; the wrapper remains an incomplete architectural boundary.

## Ownership and movement

Gameplay ownership splits by responsibility. `CharacterSyncer` applies remote movement correction only to characters it does not own locally. `ItemPhysicsSyncer` further restricts its correction to grounded items. `Player.EmptySlot` uses the room's master role and the owning character's role to decide the inventory-change path; `Player.SyncInventoryRPC` applies the received inventory and refreshes dependent weight and pocket behavior. These are different rules because moving a character, simulating an item, and committing inventory changes are different operations.

For continuous state, `PhotonBinaryStreamSerializer<T>.OnPhotonSerializeView` calls the concrete syncer's `ShouldSendData` and `GetDataToWrite`. On receipt it decodes the typed state and calls `OnDataReceived`. `CharacterSyncer.OnDataReceived` updates movement-related state and adjusts ragdoll velocities. Its `Update` smooths remote look direction, while `FixedUpdate` calls `InterpolateRigPositions` to correct the body's position.

`CharacterSyncer.OnDataReceived` stores the current local body position as its previous position and keeps the incoming position as the remote target. `InterpolateRigPositions` then picks a fixed blend between those values and moves the ragdoll a fraction of the remaining correction each physics step. Large mismatches take a separate snap path, and vertical correction follows a different rule than horizontal correction.

This is a correction controller, not playback of a timestamped movement history. `PhotonBinaryStreamSerializer.OnPhotonSerializeView` gets message information, but the character interpolation path does not keep a queue of samples with their source times. The smoother has only the latest received target and its local state at receipt, rather than two samples bracketing a chosen presentation time.

The target matters because it is a simulated ragdoll. New arrivals change both velocity correction and the position target, while fixed updates continue moving the body between arrivals. Arrival timing thus shapes the correction path. The code cannot reconstruct a consistent delayed sample timeline from state it does not store. Remote look direction uses its own clock based on elapsed time since receipt and the configured serialization interval, so body and look smoothing use different playback models.

The fixed correction fraction also ties convergence to physics-step count. It is not derived from an explicit response time using the current timestep. A change to physics frequency would therefore alter how much correction happens over the same elapsed second. The existing large-error and vertical rules add further limits to any claim this is simple linear interpolation of remote movement.

The correction code lacks a clear temporal model. A bounded sample history should define a target at an explicit delayed time, and a timestep-aware controller should correct the physical body toward it. Teleports and carrying need their own transitions. Instead, receipt state and per-step fractions do too much work defining smoothness. That makes sample age and convergence harder to reason about, and a physics-frequency change can shift correction behavior over the same elapsed second.

## Items and ropes

Items specialize that pattern. `ItemPhysicsSyncer.ShouldSendData` suppresses ordinary updates for sleeping or kinematic items and supports a temporary forced-send period. `Item.OnPlayerEnteredRoom` requests that period on the master client so a newly arrived player can receive item state even when the object is not actively moving. `ItemPhysicsSyncer.OnDataReceived` updates its working position and velocity state, and its fixed update moves the remote body toward the target.

`ItemPhysicsSyncer.Awake` caches the fixed-step increment, the interval derived from the serialization rate, and the allowed angular change per step. `FixedUpdate` then uses those cached values. The general `PhysicsSyncer` performs comparable calculations from the current timestep and serialization rate during its fixed update.

The issue is a hidden lifetime assumption in the item-specific component: timing configuration must remain unchanged after initialization for those cached values to reflect the current update schedule. A configuration change later would affect the general syncer immediately but leave an existing item syncer stuck with its earlier timing. The two classes express similar correction behavior through different configuration lifetimes.

This matters to maintenance before you even introduce a new runtime setting. A timing change needs review against component creation order, not just the place where the rate gets assigned. A test that creates the syncer before applying configuration exercises a different case from one that applies configuration first.

The timing configuration has an undocumented lifetime contract. Either establish it as immutable before initialization, notify every cached consumer when it changes, or recompute the cheap derived values. `ItemPhysicsSyncer` and the general syncer should not silently disagree about when configuration takes effect. Inferring immutability from a missing refresh path is a design defect, not documentation.

Ropes use a different policy. `RopeSyncer.ShouldSendData` considers whether there are rope segments, whether characters are nearby, and how long the climbable rope has existed. Older ropes reduce their update frequency. Received rope state goes to `Rope.SetSyncData`, rather than being treated as one ordinary rigidbody transform.

Replication is deliberately specialized. Sleeping-item suppression avoids treating a stationary object like a moving character, and a rope's collection of segments needs a different update policy from a held object. Inventory restoration also triggers gameplay consumers such as carried weight. A general transform replicator would not account for those dependencies.

## State restoration and diagnostics

Inventory and afflictions have their own consumers. `Player.SyncInventoryRPC` restores slots from `InventorySyncData`, then updates dependent presentation and behavior. `CharacterAfflictions` publishes its status state separately from `CharacterSyncer`'s movement-oriented updates. `ReconnectHandler.RegisterPlayer` observes character-state, status, warp, and inventory events, while its periodic update refreshes stored reconnect data. `UpdateReconnectData` builds a record from the character and retains it when valid. The reconnect record is a recovery view of several systems, not the same object as a live movement update.

The price is that consistency depends on several lifecycles working together. A room can be ready before a character is registered for recovery, and an item can exist before a late arrival has received its settled state. The forced item refresh and reconnect observers are explicit attempts to cover those transitions.

Even the diagnostics ask you to sort out the units.

`NetworkStats.Update` computes deltas from the peer's byte counters. `NetworkStatsPage.Update` multiplies the aggregate incoming and outgoing deltas by eight before formatting them, but formats the per-type deltas without that conversion. Both groups pass through `ToPrettySize`, and the aggregate label still calls its values bytes.

The result is inconsistent. The aggregate and per-type entries are not in the same unit even though they appear together with the same rate suffix. Their totals also have different coverage: the per-type accounting is registered by the binary serialization path, while aggregate peer counters cover traffic outside that path. A reader comparing the two needs both distinctions, and the display communicates neither clearly.

Timing adds another measurement error. `NetworkStats.Update` accumulates frame delta until its sampling threshold, then subtracts one second and reports the raw counter difference. It does not divide the count by the actual elapsed sampling interval. After a long frame, the value displayed as a per-second rate can span a longer interval; subsequent updates then report shorter intervals while the timer catches up.

The network diagnostics are unreliable as a comparison tool when their units and sampling windows disagree. These defects do not prove broken replication; they make investigating replication cost harder. Record the actual interval, normalize rates against it, and label bytes and bits consistently. Per-type payload counts also need identifying as payload counts, not passed off as a breakdown of the entire peer total. A performance display that demands this much interpretation is failing at its job.

## Reading and changing this code

Trace both the state owner and the consumer of a change. Updating a local inventory display differs from updating slot state, and restoring slot state has weight and pocket-behavior consequences. A settled object's late-join refresh is also a different lifecycle from its ordinary movement updates.

For compatibility review, identify whether a component uses `NetCode`, Photon callbacks, or a specialized syncer. The interface alone does not describe every dependency. Keep diagnostics' measurement scope separate from gameplay evidence: lower reported payload volume does not establish better latency or smoother remote motion.

## Open questions

This chapter does not establish remote smoothness, loss recovery, or four-player performance through a live session. The correction model's visible behavior needs controlled latency and arrival-variation observations. The cached item timing finding is conditional on configuration changing after component initialization; no ordinary shipped setting that triggers that condition was established in this pass.

Host changes have component-specific handling, including rope callbacks and reconnect records. An empty callback in one class is not enough to conclude that the whole game loses recovery state when the master changes. That needs a complete transition trace and an official session observation.

The server's implementation and internal validation are outside the inspected client source. This chapter describes client responsibilities without inferring those remote internals or treating client-side checks as a full account of service behavior.
