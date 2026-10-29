# Settings, presentation, and supporting systems

Settings, localization, rendering support, and ordinary audio do not each need a chapter just because they have classes. Most of this is the work of moving authored values into Unity components. The sound paths deserve more space because they manage to disagree about what those values mean.

Settings control input, display quality, audio levels, and accessibility options. Localization selects text and platform-specific wording. Scene presentation combines sky, fog, screen effects, grass, and environmental sound. Footsteps choose sounds from the surface under the scout, while the sound-effect player manages short effects and reusable sources. Item optimization reduces work for inactive objects without treating every object as permanently disabled.

## Preferences and presentation

`SettingsHandler` constructs setting objects, then `AddSetting` loads and applies each one. `SaveSetting` delegates persistence to `CustomSettingsSaveLoad` and explicitly writes preferences to disk. That adapter uses Unity's preference storage. Input binding overrides follow a separate path through `Rebinding.LoadRebindingsFromFile` and `SaveRebindingsToFile`, which serialize the input system's binding overrides into preference storage despite the method names referring to a file.

`LocalizedText.GetText` checks the active platform-specific table before the main table, selects the current language, and falls back to the base-language entry when the selected text is empty. Text components can use a `LocalizedText` component for refresh behavior. This is separate from proving that a translated label fits its layout or that a font contains every needed glyph.

`ScreenVFX.Play` drives a material intensity through tweens. Its photosensitivity branch reduces intensity and uses a different transition sequence; `Start` also adjusts duration from the cached GUI setting. `PhotosensitiveSetting.ShouldShow` restricts changing that setting while in a room. The setting's limited availability and the effect's cached inputs belong to the same lifecycle, rather than being independent controls.

`AmbienceManager.Update` writes sky and fog presentation and derives ambient light from the day/night state. `GrassRenderer` obtains grass data from its provider, manages compute buffers, and selects the gameplay camera for rendering. `ItemOptimizationManager.Update` visits registered items and generic optimizers. For a grounded kinematic item it disables selected scripts, including physics synchronization and impact sound; becoming active restores those scripts.

## Sound playback

Ordinary audio uses both shared and dedicated sources. `SFX_Instance.Play` enters `SFX_Player.PlaySFX`, which checks eligibility and range, obtains a pooled source, and starts `IPlaySFX`. `FootStepPlayer` selects surface sounds through `StepSoundCollection`; a supplied source takes the collection through `SFX_Instance.PlayFromSource`. `AmbienceAudio` separately uses environmental state and its animator to control ambience. Voice remains in its own chapter because its spatial origin and filtering also participate in hazards.

Setting objects keep preference-specific behavior out of the storage adapter. A reusable sound-source pool likewise provides a place to manage simultaneous effects. Dedicated-source and pooled playback implement overlapping settings independently, and the pooled path maintains a second estimate of playback completion rather than using the state actually assigned to the source.

Volume and pitch have separate properties. This path uses the volume property twice.

`SFX_Instance.PlayFromSource` selects a clip and assigns `GetVolume` to `AudioSource.volume`. Its next assignment writes `GetPitch` into that same volume property. The computed loudness is immediately discarded, and the method never assigns the requested pitch to `AudioSource.pitch`.

`StepSoundCollection` uses this method when a source override is supplied, `MandrakeScreamFX` uses it for the scream source, and `MushroomZombie` uses it for a grunt. The error is not confined to an unused convenience method. Those callers pass authored sound settings to a path that substitutes the pitch calculation for their loudness.

The pooled implementation in `SFX_Player.IPlaySFX` assigns volume and pitch separately, so the same sound definition has different setting semantics depending on which playback path receives it. A change to the sound's pitch settings can change dedicated-source loudness instead, while its actual pitch retains the source's existing value.

Both paths should use the same source-configuration operation. Correcting the assignment would fix this defect; centralizing the common settings would also remove the duplicated implementation that allowed the paths to diverge. The audible magnitude still depends on the authored settings and mixer, but the property mismatch is direct.

The pool then times playback using a pitch other than the one it actually chose.

`SFX_Player.IPlaySFX` assigns a varied pitch to the audio source. Its coroutine then advances its playback-duration counter using the unvaried base `settings.pitch`. When the counter reaches the clip's duration, the coroutine calls `SFX_Source.StopPlaying` and returns that source to the pool.

For a nonlooping clip whose assigned pitch is below the base pitch, the timer reaches completion before playback at that assigned pitch has finished. The source is stopped early. With a higher assigned pitch, the timer retains the source after the clip's natural completion. The state used to decide source availability therefore disagrees with the state used to play the sound.

`GetAvailibleSource` selects sources using `isPlaying`, and `OnStopped` updates the shared count and active-handle list. Those are the consumers that make the timing error matter beyond the clip itself. A retained silent source still occupies the pool's logical active state; an early stop truncates the slower playback before making the source available again.

The scheduling should follow the actual assigned pitch or the source's playback completion, with looping sounds handled explicitly. If game time and audio time need different pause behavior, that policy must also be deliberate. The current coroutine introduces a separate clock without keeping its inputs aligned with the source.

## Reading and changing this code

Distinguish preference storage from applying a setting. `SettingsHandler` performs both operations through different calls, while input rebinding has its own save/load path. A value in preferences does not prove that every existing component has refreshed its cached interpretation.

Identify the audio entry point before diagnosing a sound definition. Shared effects, dedicated sources, ambience, and voice do not all apply the same settings through the same methods. Rendering support likewise cannot be assessed from C# parameter names alone; the material, shader, and scene references determine what those parameters do.

## Open questions

No audio comparison or pool-occupancy measurement was performed. The dedicated-source assignment is wrong in the source, and the pooled-duration mismatch is conditional on pitch variation; this chapter does not identify a particular audible cutoff from play.

Localization completeness, font coverage, accessibility effectiveness, and GPU rendering quality have not been established by this pass. The optimization code's intended work reduction is visible, but its frame-time benefit is unmeasured. These supporting systems remain merged because the traced defects fit within this chapter and do not support padding each helper into its own teardown.
