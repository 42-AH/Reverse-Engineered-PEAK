# Multiplayer sessions and replication

There is a session interface, a matchmaking interface, and a named connection state machine. Then the gameplay code reaches straight for Photon. PEAK has done some of the work of separating its networking responsibilities, which makes the places where it stops bothering especially noticeable.

This chapter follows those responsibilities rather than treating multiplayer as one host-controlled loop. Its main findings concern a split abstraction boundary, motion correction without a sample timeline, and instrumentation that mixes units. These have different consequences: maintenance coupling, a constrained smoothing model, and misleading diagnostic comparisons.

Steam lobby integration provides party discovery and invitation handling. Photon supplies the session implementation exposed by `NetCode.Session`. `NetworkingUtilities.MAX_PLAYERS` defaults to four, and both the room options and Steam lobby creation use that value. The default room size is therefore shared configuration rather than a number independently embedded in the two creation paths.

Once players are together, continuous state and discrete actions travel through different mechanisms. Character and physics syncers publish changing state. Inventory changes and gameplay events use explicit operations. Reconnect handling retains character and run-related records so a player's return can be distinguished from a fresh arrival. Joining a room, having a character, and having the latest state are separate stages.

## Sessions and matchmaking

`Peak.Network.NetCode` creates a `PhotonShim` for sessions and a `SteamLobbyAPI` for normal Steam matchmaking. `PhotonShim` exposes readiness, room membership, host identity, and connection events. Its initialization enables scene synchronization and configures the backend for the game's version before connecting.

The connection path has several named layers:

- `ConnectionService` registers states including `HostState`, `JoinSpecificRoomState`, `InRoomState`, and `DisconnectingState`.
- `NetworkConnector.HandleConnectionState` interprets the current state and performs the corresponding room operation.
- `NetworkingUtilities.HostRoomOptions` supplies the room configuration for hosting.
- `NetworkConnector` receives room-creation, join, and failure callbacks and advances the surrounding flow.

`SteamLobbyAPI` delegates lobby work to `SteamLobbyHandler`, retains pending invitations, and exposes matchmaking events. Its `CreateLobby` uses the same player limit as room creation. Steam lobby membership and Photon room membership remain separate properties; neither should be treated as proof that all gameplay objects have arrived.

The division between matchmaking and sessions is useful because an invitation and a replicated world are not the same problem. `ConnectionService` also gives connection intent a named state instead of leaving every callback to infer whether the player was hosting, joining, or leaving.

The interface is tidy until the gameplay code needs to use it.

`NetCode.Session` and `ISessionAPI` provide an interface, but `NetworkConnector.HandleConnectionState` still invokes Photon room and connection operations directly. `CharacterSyncer`, `ItemPhysicsSyncer`, `RopeSyncer`, and `Player` likewise depend on Photon views, callbacks, or serialization. Even `ISessionAPI` exposes Photon's authentication type in its public contract.

This is a partial abstraction with a concrete maintenance cost. A change to session lifecycle or ownership semantics cannot be contained in `PhotonShim`; the connection controller and multiple gameplay components must be reviewed as well. Substituting a test implementation of `ISessionAPI` would not intercept the direct calls made by those consumers. Tests of connection intent would still encounter transport-specific operations outside the adapter.

The issue is not the mere presence of a third-party library. Direct dependencies can be an intentional choice. The problem is that the code offers a session boundary whose scope is narrower than its name suggests, while host and membership checks can come from either the interface or the backend. A reviewer must know which side of that boundary each subsystem actually uses.

A clearer design would choose and enforce the boundary. Connection intent and room lifecycle could go entirely through the session layer, while a separate replication layer owns object identity and updates. Alternatively, the interface could be explicitly limited to the small set of services it really wraps. Either approach would reduce the work of determining whether a change belongs in an adapter, a controller, or a gameplay component. This is an architectural maintenance finding, not a claim that the current backend is incapable of supporting the game.

## Ownership and movement

Gameplay ownership is distributed by responsibility. `CharacterSyncer` applies remote movement correction only to a character whose view is not locally owned. `ItemPhysicsSyncer` additionally restricts its correction path to a grounded item. `Player.EmptySlot` uses the room's master role and the owning character's role to choose the inventory-change path; `Player.SyncInventoryRPC` applies the received inventory and refreshes dependent weight and pocket behavior. These are different rules because moving a character, simulating an item, and committing inventory changes are different operations.

For continuous state, `PhotonBinaryStreamSerializer<T>.OnPhotonSerializeView` calls the concrete syncer's `ShouldSendData` and `GetDataToWrite`. On receipt it decodes the typed state and invokes `OnDataReceived`. `CharacterSyncer.OnDataReceived` updates movement-related state and adjusts ragdoll velocities. Its `Update` smooths remote look direction, while `FixedUpdate` calls `InterpolateRigPositions` to correct the body's position.

`CharacterSyncer.OnDataReceived` records the current local body position as its previous position and keeps the incoming position as the remote target. `InterpolateRigPositions` then chooses a fixed blend between those values and moves the ragdoll a fraction of the remaining correction each physics step. Large discrepancies take a separate snap path, and vertical correction uses a different rule from horizontal correction.

That is a correction controller, not playback of a timestamped movement history. `PhotonBinaryStreamSerializer.OnPhotonSerializeView` receives message information, but the character interpolation path does not maintain a queue of samples with their source times. The smoother has the latest received target and its local state at receipt, rather than two samples surrounding a chosen presentation time.

The consumer matters because the target is a simulated ragdoll. New arrivals alter both velocity correction and the position target, while fixed updates continue moving the body between arrivals. Arrival timing consequently participates in the correction trajectory. The code cannot recover a consistent delayed sample timeline from state it does not retain. Remote look direction uses another clock, based on elapsed time since receipt and the configured serialization interval, so body and look smoothing do not share one playback model.

The fixed correction fraction also makes convergence depend on physics-step count. It is not converted from an explicit response time using the current timestep. A physics-frequency change would therefore change how much correction occurs over the same elapsed second. The existing large-error and vertical rules further limit any claim that this is simple linear interpolation of remote movement.

A clearer model would separate remote presentation from physical correction. A bounded sample history could define the presentation target at a chosen delayed time; a timestep-aware correction controller could then move the physical body toward that target. Teleports and carrying would remain explicit transitions. This would make the smoothing policy testable in terms of sample age, target time, and correction error instead of only visual tuning of per-step fractions.

## Items and ropes

Items specialize that pattern. `ItemPhysicsSyncer.ShouldSendData` suppresses ordinary updates for sleeping or kinematic items and supports a temporary forced-send period. `Item.OnPlayerEnteredRoom` requests that period on the master client so a newly arrived player can receive item state even when the object is not actively moving. `ItemPhysicsSyncer.OnDataReceived` updates its working position and velocity state, and its fixed update moves the remote body toward the target.

`ItemPhysicsSyncer.Awake` caches the fixed-step increment, the interval derived from the serialization rate, and the allowed angular change per step. `FixedUpdate` continues using those cached values. The general `PhysicsSyncer` performs comparable calculations from the current timestep and serialization rate during its fixed update.

The consequence is a hidden lifetime assumption in the item-specific component: timing configuration must remain unchanged after initialization for those cached values to describe the current update schedule. A later configuration change would affect the general syncer immediately but leave an existing item syncer using its earlier timing. The two classes express similar correction behavior through different configuration lifetimes.

This matters to maintenance even before introducing a new runtime setting. A timing change must be reviewed against component creation order, not just against the place where the rate is assigned. A test that creates the syncer before applying configuration exercises a different case from a test that applies configuration first.

The better contract is explicit: either timing is immutable and established before these components initialize, or rate changes notify every cached consumer. Recomputing a few inexpensive derived values at the update boundary is another option. The implementation should not require a reader to infer immutability from the absence of a refresh method.

Ropes use another policy. `RopeSyncer.ShouldSendData` considers whether there are rope segments, whether characters are nearby, and how long the climbable rope has existed. Older ropes reduce their update frequency. Received rope state is handed to `Rope.SetSyncData`, rather than being treated as one ordinary rigidbody transform.

Replication is deliberately specialized. Sleeping-item suppression avoids treating a stationary object like a moving character, and a rope's collection of segments needs a different update policy from a held object. Inventory restoration additionally triggers gameplay consumers such as carried weight. A general transform replicator would not account for those dependencies.

## State restoration and diagnostics

Inventory and afflictions have their own consumers. `Player.SyncInventoryRPC` restores slots from `InventorySyncData`, then updates dependent presentation and behavior. `CharacterAfflictions` publishes its status state separately from `CharacterSyncer`'s movement-oriented updates. `ReconnectHandler.RegisterPlayer` observes character-state, status, warp, and inventory events, while its periodic update also refreshes stored reconnect data. `UpdateReconnectData` builds a record from the character and retains it when valid. The reconnect record is a recovery view of several systems, not the same object as a live movement update.

The cost is that consistency depends on several lifecycles agreeing. A room can be ready before a character is registered for recovery, and an item can exist before a late arrival has received its settled state. The forced item refresh and reconnect observers are explicit attempts to cover those transitions.

Even the diagnostics ask the reader to sort out the units.

`NetworkStats.Update` computes deltas from the peer's byte counters. `NetworkStatsPage.Update` multiplies the aggregate incoming and outgoing deltas by eight before formatting them, but formats the per-type deltas without that conversion. Both groups pass through `ToPrettySize`, and the aggregate label still calls its values bytes.

The resulting comparison is inconsistent. The aggregate and per-type entries are not expressed in the same unit even though they appear together with the same rate suffix. Their totals also have different coverage: the per-type accounting is registered by the binary serialization path, while aggregate peer counters cover traffic outside that path. A reader comparing the two needs both distinctions, and the display communicates neither clearly.

Timing introduces another measurement error. `NetworkStats.Update` accumulates frame delta until its sampling threshold, then subtracts one second and reports the raw counter difference. It does not divide the count by the actual elapsed sampling interval. After a long frame, the value displayed as a per-second rate can cover a longer interval; subsequent updates can then report shorter intervals while the timer catches up.

These are diagnostic defects rather than gameplay replication defects, but they directly affect investigation of replication cost. A bandwidth change can be misread when units differ or the observation window changes. The fix is to record the actual sampling duration, normalize every rate against that duration, and label bytes or bits consistently. Per-type payload accounting should be identified as such instead of being presented as a decomposition of the entire peer total.

## Reading and changing this code

Trace both the state owner and the consumer of a change. Updating a local inventory display is different from updating slot state, and restoring slot state has weight and pocket-behavior consequences. A settled object's late-join refresh is also a different lifecycle from its ordinary movement updates.

For compatibility review, identify whether a component uses `NetCode`, Photon callbacks, or a specialized syncer. The interface alone does not describe every dependency. Keep diagnostics' measurement scope separate from gameplay evidence: lower reported payload volume does not establish better latency or smoother remote motion.

## Open questions

This chapter does not establish remote smoothness, loss recovery, or four-player performance through a live session. The correction model's visible behavior needs controlled latency and arrival-variation observations. The cached item timing finding is conditional on configuration changing after component initialization; no ordinary shipped setting that triggers that condition was established in this pass.

Host changes have component-specific handling, including rope callbacks and reconnect records. An empty callback in one class is not enough to conclude that the whole game loses recovery state when the master changes. That needs a complete transition trace and an official session observation.

The server's implementation and internal validation are outside the inspected client source. This chapter describes client responsibilities without inferring those remote internals or treating client-side checks as a full account of service behavior.
