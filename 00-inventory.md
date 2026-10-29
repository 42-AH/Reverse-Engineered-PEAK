# PEAK source inventory

This inventory indexes the 1,153 C# files in the game's `Assembly-CSharp` export and reads selected implementations behind the systems below. Related helpers are grouped rather than listed as separate systems. Names in parentheses are source anchors, not reproductions of their code.

The inspected assembly matches the installed Steam copy associated with app 3527290, build 26155610. Its assembly version is `0.0.0.0`, which does not identify the game's displayed version. This is a static inventory: scene attachment, serialized settings, live service responses, and behavior in the official Steam build still need separate verification. The presence of console-platform support does not establish which paths this Steam build executes.

## Character

- Player input and controller handling connect actions, rebinding, sensitivity, and input prompts (`CharacterInput`, `ControllerManager`, `Rebinding`, `InputIcon`).
- Ground movement separates movement state, camera input, ground sampling, and physics updates (`CharacterMovement.SetMovementState`, `FixedUpdate`, `SetWaterMovementModifier`).
- Wall climbing uses surface checks, handles, climb jumps, and angle-dependent stamina consumption (`CharacterClimbing.Update`, `GetAngleUsage`, `ClimbModifierSurface`).
- Helping hands and grabbing attach characters through body-part references and include a run-setting-dependent kick path (`CharacterGrabbing`, `TargetCanBeHelped`).
- Carrying changes collision and animation state, updates carry weight, and resolves competition with the backpack slot (`CharacterCarrying.RPCA_StartCarry`, `RPCA_Drop`).
- The ragdoll registers body parts and colliders, selects physics materials, and changes collision or kinematic state (`CharacterRagdoll`, `Bodypart`, `RigCreator`).
- Rope climbing and vine climbing have separate movement and synchronization paths (`CharacterRopeHandling`, `CharacterVineClimbing`).
- Character animation, hand placement, mouth motion, and eye motion follow gameplay and voice state (`CharacterAnimations`, `HandVisual`, `AnimatedMouth`, `PlayerEyeLook`).
- First-person rendering and the camera use local-player visibility rules, head offsets, camera overrides, and spectator selection (`LocalPlayerRenderer`, `HideTheBody`, `MainCameraMovement`).
- Falling, unconsciousness, death, and revival are character-state transitions with effects on input and physics (`Character`, `FallPost`, `PassOutPost`, `GameOverHandler`).
- Stamina has regular and bonus pools; the regular pool is clamped against the current maximum (`Character.UseStamina`, `ClampStamina`, `GetMaxStamina`, `AddExtraStamina`).
- Status accumulation and recovery share an affliction manager; hunger, environmental exposure, carried weight, and embedded hazards feed into it (`CharacterAfflictions.UpdateNormalStatuses`, `UpdateWeight`, `SetStatus`).
- Timed effects have their own lifecycle and network-facing updates rather than being only HUD values (`Peak.Afflictions.Affliction`, `CharacterAfflictions.AddAffliction`, `RemoveAffliction`).
- Thorns, arrows, and sticky items retain physical attachment state as well as status effects (`ThornOnMe`, `Peak.ArrowOnMe`, `StickyItemComponent`, `ThornSyncData`).

## Items and interactions

- Interactable detection and prompts are separated from the objects that implement interaction (`Interaction`, `IInteractible`, `UI_Interaction`).
- Item identity, state, instance data, and component behavior form the common item framework (`Item`, `ItemDatabase`, `ItemInstanceData`, `ItemComponent`).
- Item use distinguishes starting, continuing, cancelling, and completing actions, including use on another scout (`CharacterItems.DoUsing`, `ItemActionBase`, `UseItemOnFriend`).
- Inventory management handles equipped slots, pickups, throws, and dropped objects (`CharacterItems`, `ItemSlot`, `InventorySyncData`).
- Backpacks store item slots and have separate worn, held, and selection-wheel behavior (`Backpack.Stash`, `BackpackData`, `CharacterBackpackHandler`, `BackpackWheel`).
- Cooking modifies item effects and appearance, with additional behaviors for items that depart from the common rules (`ItemCooking.ChangeStatsCooked`, `UpdateCookedBehavior`, `AdditionalCookingBehavior`).
- Luggage and loose-item spawning use spawn pools, item state, and opening history (`Luggage.OpenLuggageRPC`, `SpawnPool`, `SingleItemSpawner`, `Peak.SpawnedItemTracker`).
- Construction previews and placement checks precede creation of usable objects (`Constructable.TryUpdatePreview`, `ConstructablePreview`, `CheckpointConstructable`).
- Rope equipment has anchors, segments, deployment, rendering, and state replication (`RopeAnchor`, `RopeSpool`, `RopeShooter`, `RopeSyncer`).
- Movement equipment applies item-specific forces or drag rather than replacing the character controller (`Glider`, `Parasol`, `Parachute`, `Rocketpack`, `CharacterBalloons`).
- Explosives and projectile tools share impact and area-effect machinery but retain specialized behaviors (`Dynamite`, `ExplosionEffect`, `AOE`, `ArrowShooter`, `ScoutCannon`).
- Light and heat sources include held equipment and environmental emitters (`ItemTorch`, `Lantern`, `Candle`, `CharacterHeatEmission`, `StatusEmitter`).
- Magical items and transformations dispatch actions into character, item, and world systems (`Action_StrangeGem`, `Action_BecomeSkeleton`, `MagicBean`, `MagicBugle`).
- Amulets maintain activation state and connect to petrification and statue interactions (`Peak.AmuletBase`, `Peak.HealingAmulet`, `Peak.PetrifiedScout`, `Peak.ScoutStatue`).
- Bing Bong has distinct physics, status, audio, and ability components (`BingBongPhysics`, `BingBongStatus`, `BingBongPowers`, `BingBongForceAbilities`); which ability paths are ordinary play remains to be traced.

## Island and environment

- Daily island selection consumes a level index and time-to-change response, then looks up a baked scene; this should not be described as a fresh daily procedural seed without further evidence (`NextLevelService`, `MapBaker.GetLevel`, `TodaysBiomes`).
- Map bundles load around the selected index and have explicit unload handling (`MapBundleService.LoadBundlesNearestIndex`, `MapBundleData.Load`, `UnLoad`).
- Generation code supports seeded stages, prop grouping, and biome detection (`MapGenerator.GenerateAll`, `LevelGeneration.Generate`, `MapGenerationStage`); the inventory does not assume these authoring paths run during ordinary play.
- Prop placement combines constraints, modifiers, deferred work, and validation (`PropSpawner`, `PropSpawnerConstraint`, `PropSpawnerMod`, `Peak.ProcGen.PropSpawnValidation`).
- Terrain, rock, vegetation, and structure builders have separate placement or mesh helpers (`TerrainSplatMesh`, `HeightmapMesh`, `RockSpawner`, `BasicGrassSpawner`, `WallPieceSpawner`, `SwampMeshGen`).
- Biome selection and segment progression connect variants, campfires, walls, and spawn history (`BiomeSelector`, `Peak.ProcGen.BiomeSelection`, `MapHandler.MapSegment`, `MapHandler`).
- Campfires coordinate burning, nearby scouts, morale, cooking, and segment-related progress (`Campfire`, `Campfire_Set_Segment`, `MoraleBoost`).
- Time of day drives lighting and environment parameters, with biome profiles and status interactions (`DayNightManager`, `DayNightProfile`, `CharacterAfflictions.UpdateNormalStatuses`).
- Advancing fog has movement, waiting, synchronization, and cold exposure logic (`Fog.Movement`, `Wait`, `MakePlayerCold`).
- Water volumes and waterfalls affect characters through dedicated movement and trigger components (`WaterZone`, `WaterfallPusher`, `WaterfallTrigger`).
- Slippery jellyfish trigger a replicated fall and poison effect (`SlipperyJellyfish.OnTriggerEnter`, `Trigger`).
- Bees, spiders, and trapping plants have separate targeting, attack, or capture behavior (`BeeSwarm`, `Spider`, `SpiderManager`, `VenusFlyTrap`).
- Cold wind applies exposure and physical force; roots-related wind points to the same wind-zone machinery (`WindChillZone.ApplyStatus`, `AddWindForceToCharacter`, `RootsWind`).
- Ice and unstable terrain respond through breakage and collision logic (`ShakyIcicleIce`, `ShakyIcicleIce2`, `ShakyRock`, `BreakableBridge`, `SandStoneClimb`).
- Heat exposure checks sunlight and obstruction; desert hazards also include moving storms, ambushers, and mirages (`HotSun.Update`, `Tornado`, `Antlion`, `Scorpion`, `MirageManager`).
- Lava contact, rising lava, tides, and eruptions are distinct components (`Lava`, `LavaRising`, `LavaTides`, `EruptionSpawner`).
- Gloom tracks field membership and scout visibility, with safe-zone exclusions (`Peak.StatusFieldGloom`, `Peak.GloomSafeZone`, `Peak.FogSafeZoneManager`).
- Voice-stealing and confusion hazards interact with the voice system (`Peak.Island.VoiceThief`, `Peak.Island.GloomVoiceConfuser`, `VoiceObscuranceFilter`).
- Mushroom infection and zombie behavior have targeting, spawning, and bite paths (`MushroomZombie`, `MushroomZombieSpawner`, `ZombieManager`, `Peak.Afflictions.Affliction_ZombieBite`).
- Scoutmaster pursuit has its own target selection, spawning, and animation state (`Scoutmaster`, `ScoutmasterSpawner`, `ScoutmasterAnimVars`).
- Traps and physical obstacles include timed or contact-driven machinery (`SwingingAxe`, `Peak.SpikeTrap`, `Peak.SpikeRoller`, `Peak.MovingSawBlade`).
- The void/Nadir path tracks activation, spawn points, and soul-related progress (`Peak.VoidBiome`, `Peak.ScoutmasterSoulPillar`, `Peak.PeakGatePortal`).

## Run lifecycle and persistence

- Bootstrapping and scene transitions initialize services, process invitations, and coordinate loading (`GameBooter`, `GameHandler`, `SceneSwitchingStatus`, `LoadingScreenHandler`).
- The airport includes check-in, invitations, photobooth interaction, and readiness to depart (`AirportCheckInKiosk`, `AirportInviteFriendsKiosk`, `AirportPhotoboothKiosk`, `ReadyUpToTrigger`).
- Character and player registration establish participants and spawn state (`PlayerHandler`, `CharacterSpawner`, `Player`).
- Run timing and end-of-run checks respond to participant changes and restored progress (`RunManager.StartRun`, `CheckForGameEnd`, `RunStarter`).
- Ascents and custom runs change rules, expose selectable options, and affect achievement eligibility (`Ascents`, `RunSettings`, `CustomOptionsWindow`, `DisableBasedOnRunSettings`).
- Summit completion coordinates rescue, the ending sequence, and results UI (`PeakSequence.CheckGameComplete`, `PeakHandler.SummonHelicopter`, `EndScreen`).
- Ghosts retain appearance and follow a selected scout while the camera supports spectating (`PlayerGhost.RPCA_InitGhost`, `RPCA_SetTarget`, `MainCameraMovement.Spectate`).
- Reconnection records player state separately from the active character object (`ReconnectHandler`, `ReconnectData`).
- Quicksaves serialize run and player progress, check compatibility, and restore spawn history (`Peak.Quicksave`, `RunManager.Start`, `MapHandler.PopulateSpawnHistories`).
- Session-persistent appearance data is cached and synchronized to participants (`PersistentPlayerDataService`, `SyncPersistentPlayerDataPackage`); its name alone does not mean disk persistence.
- Achievement evaluation combines run-based values with platform statistics and platform unlock calls (`AchievementManager`, `SerializableRunBasedValues`, `ACHIEVEMENTTYPE`).
- Badges expose achievement-linked display and locked-state behavior (`BadgeManager.InitBadges`, `BadgeData`, `BadgeUI`).
- Customization and passport interaction select and apply appearance options and unlock gating (`Customization`, `CustomizationOption`, `CharacterCustomization`, `PassportManager`).
- Guidebook pages have opening, display, and seen-page persistence (`Guidebook`, `GuidebookSpread`, `Peak.GuidebookData`).

## Networking and platform services

- Session and matchmaking interfaces separate Photon session handling from Steam lobby integration (`Peak.Network.NetCode`, `PhotonShim`, `SteamLobbyAPI`).
- Room creation uses a default four-player limit (`Peak.Network.NetworkingUtilities.MAX_PLAYERS`, `HostRoomOptions`); this is a room configuration limit, not proof that every subsystem scales to another size.
- Connection states cover hosting, joining, room membership, and disconnection (`ConnectionService`, `HostState`, `JoinSpecificRoomState`, `InRoomState`, `DisconnectingState`).
- Authority is distributed across object ownership and host checks; character and world paths need to be traced separately (`CharacterClimbing.Update`, `CharacterCarrying.RPCA_StartCarry`, `RunManager.Start`).
- Character replication transmits movement and gameplay state and adjusts remote physics (`CharacterSyncer.GetDataToWrite`, `OnDataReceived`, `CharacterSyncData`).
- Items, ropes, status, and inventory use their own synchronization structures alongside RPCs (`ItemPhysicsSyncer`, `RopeSyncer`, `StatusSyncData`, `InventorySyncData`, `PhotonBinaryStreamSerializer`).
- Voice connects Photon recording and playback to character state and distance/obstruction filtering (`VoiceClientHandler`, `CharacterVoiceHandler`, `VoiceObscuranceFilter`).
- Platform integration includes authentication, rich presence, communication checks, and cross-platform configuration (`PlatformBootstrap`, `SteamAuthTicketService`, `RichPresenceService`, `CloudAPI.CrossplayConfig`); internal platform-provider behavior remains a separate review.
- Player blocking, reporting, and sanction UI are present (`Portningsbolaget.BlocklistPopup`, `ReportPlayerPopup.OnReportPlayer`, `MainMenuMainPage.HandleSanctions`).
- Version and lobby verification use service requests (`CloudAPI.CheckVersion`, `VerifyLobby`); server implementation and data retention cannot be established from these callers.
- Local mod-state detection compares file checksums (`Portningsbolaget.ModUtility.InitialiseAsync`); this is evidence of an integrity-related path, not evidence of a complete anti-cheat system.

## Presentation and support

- Main menus and pause menus use page handlers with separate play, join, settings, and confirmation views (`MainMenuPageHandler`, `MainMenuPlayPage`, `MainMenuJoinRoomPage`, `PauseMenuHandler`).
- HUD components display stamina, afflictions, inventory, item-use progress, ropes, and player names (`GUIManager`, `UI_Stamina`, `InventoryItemUI`, `UI_UseItemProgress`, `UIPlayerNames`).
- Pings and emotes have their own input and display paths (`PointPinger`, `PointPing`, `EmoteWheel`, `EmoteWheelSlice`).
- Localization selects platform-specific tables and updates text, fonts, and spacing (`LocalizedText`, `AutoLocalizer`, `FontFallbackSwapper`, `Peak.LanguageLineSpacing`).
- Settings registration loads and applies persisted values; individual classes implement graphics, input, sound, and accessibility choices (`SettingsHandler.AddSetting`, `SaveSetting`, `CustomSettingsSaveLoad`).
- Screen effects and post-processing react to gameplay, with photosensitivity handling in effect activation (`ScreenVFX.StartFX`, `PostHandler`, `PassOutPost`, `LavaPost`).
- C# rendering support supplies material parameters, terrain colors, grass drawing, and lighting-volume interactions (`ShaderParameters`, `SetRockColors`, `GrassRenderer`, `DayNightManager`); shader source and extracted art are outside this text repository.
- Audio uses reusable effect playback, surface footsteps, ambience, music triggers, and mixer settings (`SFX_Player`, `FootStepPlayer`, `AmbienceAudio`, `BiomeMusicCheck`, `Peak.Settings.AudioMixerUnifier`).
- Performance helpers throttle or disable item work and cull particles, lights, and other scene work (`Peak.ItemOptimizationManager`, `GenericOptimizer`, `ParticleCuller`, `LightCuller`, `GrassChunking`).
- Debugging and authoring support includes inspection pages, generation tools, and command registration (`NetworkStatsPage`, `ItemInstanceDataDebugPage`, `MapDebugUI`, `Zorro.Core.CLI.ConsoleCommands`); no activation instructions belong in the write-up.
- The oddities chapter examines unequal-length biome-list comparison in `MapBaker.BiomeResult.IsIdenticalTo`. The items chapter follows secondary-use dispatch in `CharacterItems.DoUsing` through its consumers; repeated dispatch alone is not evidence of a player-visible bug.

## Limits of this pass

This is a coverage inventory, not a completed audit of every method. The file index includes framework and helper classes; method bodies were sampled around gameplay responsibilities and cross-system calls. The chapters trace selected calls; further work must check serialized scene usage before making claims about which hazards occur in each biome, debug-code reachability, timing, or player-visible defects.

Managed platform dependencies, native plugins, and remote services have not received the same review as `Assembly-CSharp`. The observed service callers, reporting UI, and checksum utility do not establish whether other telemetry or integrity checks exist. No claim of their absence is justified yet. The daily selector and seeded generation code also need to remain separate subjects until their production call paths are established.

The chapter index groups these systems by responsibility; the inventory is not a claim that every method or authored scene has been audited.
