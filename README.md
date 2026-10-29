# PEAK teardown

This is my written analysis of how PEAK's code is put together, based on reading a decompile of my own purchased Steam copy. It is not affiliated with Landfall, Aggro Crab, or Team PEAK.

This repository contains no game source code, assets, or extracted data dumps. It contains no cheats, ownership bypasses, or instructions for playing outside Steam. Class and method names identify what the analysis discusses; their implementations are not reproduced.

The inspected game uses Unity's Mono backend. Its managed assembly contains ordinary, unencrypted IL, read with standard decompiler tooling. No technical protection was circumvented for that inspection. The distinction matters under the DMCA's anti-circumvention rules: inspecting readable software and defeating an access control are different acts. Sega v. Accolade (1992) and Sony v. Connectix (2000) recognized fair use in software reverse-engineering cases.

Copyright protects expression, including code and art, rather than the underlying ideas, methods, and systems. This repository describes those systems in original prose. It does not distribute their implementation.

The [PEAK EULA](https://store.steampowered.com/eula/3527290_eula_0) includes a reverse-engineering restriction. That is a contract term, distinct from copyright law. Its accompanying developer commentary addresses code theft and copycat games and permits modding. This project is written criticism and documentation of the official game; it supplies neither game files nor a competing implementation. The text license grants rights only to this analysis.

Rights holders with concerns can open an issue in this repository or contact [42-AH on GitHub](https://github.com/42-AH). I'll respond.

Read in order for the connections between systems, or use the index to follow one responsibility. Each chapter establishes the call paths before criticizing them; unresolved runtime questions stay at the end. The source snapshot is Steam app 3527290, build **26155610**, identified by the installed Steam app manifest. The matching managed assembly reports version `0.0.0.0`, which does not identify the displayed game version. This is a static source analysis, not a record of reproduced gameplay defects.

## Index

- [Source inventory](00-inventory.md) — The 87 grouped systems found in the source pass.
- [Character controller](02-character-controller.md) — Movement, climbing, carrying, ragdoll state, and camera responsibilities.
- [Stamina and afflictions](03-stamina-afflictions.md) — Status accounting, timed effects, recovery, and restoration semantics.
- [Items, inventory, and equipment](04-items-inventory.md) — Item identity, action lifecycles, cooking, backpacks, and specialized equipment.
- [Daily islands and generation](05-islands-generation.md) — Prepared-scene selection, bundle readiness, placement, and reproducibility.
- [Weather and environmental hazards](06-weather-environment.md) — Exposure, wind forces, water tracking, sunlight, and physical hazards.
- [Creatures and unusual biomes](07-creatures-biomes.md) — Target lifetimes, Gloom protection, spider scheduling, and contact hazards.
- [Airport, ascent, and ending](08-run-lifecycle.md) — Departure, run identity, campfires, death, revival, and final timing.
- [Multiplayer sessions and replication](09-networking.md) — Session boundaries, ownership, motion correction, and diagnostic accounting.
- [Voice](10-voice.md) — Capture policy, spatial playback, filtering, and mouth animation.
- [Saves and reconnection](11-saves-reconnection.md) — Checkpoint construction, reconnect records, load outcomes, and commit strategy.
- [Achievements, badges, and appearance](12-progression-customization.md) — Platform progress, cosmetic restoration, badge resources, and guidebook state.
- [Menus, HUD, and communication](13-interface.md) — Page lifecycles, inventory caches, prompts, pings, and emote commitment.
- [Everything else](14-everything-else.md) — Settings, localization, rendering support, ordinary audio, and optimization.
- [Platform services and integrity](15-platform-services.md) — Platform readiness, service boundaries, reporting, and integrity responsibilities.
- [Oddities and diagnostic code](16-oddities.md) — Stale inspection state, result observation, and diagnostic lifetimes.

The written analysis is licensed under [CC BY 4.0](LICENSE). PEAK and its code remain the property of their respective rights holders.
