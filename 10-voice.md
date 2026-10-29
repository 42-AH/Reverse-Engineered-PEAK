# Voice

## What it does

Voice follows the scout through the world, responds to struggling and unconsciousness, and can move away from the speaker when a creature steals it. Tracing those effects leads beyond microphone capture into character state, camera position, audio filters, and mouth animation. The transport boundary is mostly delegated to Photon Voice. The more specific problems are in the game's interpretation of transmission state and its spatial and visual presentation of received audio.

## How it's built

`CharacterVoiceHandler.Start` obtains the character, Photon Voice `Recorder`, microphone and push-to-talk settings, and the `AudioSource` used for playback. It also looks up persistent player data and the listener attached to `MainCamera`. For the locally owned character it requests communication permission through `PlatformManager.Platform.CanCommunicate`. Remote characters receive mixer groups through `PlayerHandler.AssignMixerGroup`; `UpdateAudioLevel` loads the listener's chosen volume for that player's user ID.

The main paths divide as follows:

- `MicrophoneSetting.SetValue` selects and saves a device; `CharacterVoiceHandler.Update` notices a changed device ID and assigns the recorder's microphone.
- `CharacterVoiceHandler.PushToTalk` combines `PushToTalkSetting` with `CharacterInput.pushToTalkPressed`, then changes `Recorder.TransmitEnabled` when the requested state changes.
- The remainder of `CharacterVoiceHandler.Update` applies communication restrictions and playback mute state, separately from that requested transmission state.
- `CharacterVoiceHandler.LateUpdate` moves remote playback to `CharacterVoiceTransformProvider.GetVoicePosition`.
- `VoiceObscuranceFilter.Update` adjusts filtering from character state, world obstruction, distance, and the light volume.
- `CharacterVoiceHandler.OnAudioFilterRead` measures received samples; `AnimatedMouth.ProcessMicData` converts the resulting level into a talking sprite.

The transmission modes include voice activation, push to talk, and push to mute. These select whether recording is allowed to transmit; the handler does not implement a voice codec. The local character's public `transmitting` field is also consumed by `GUIManager`, which uses it to decide whether to show the mute icon outside offline mode.

Playback has a separate policy. During a cinematic, the handler uses nonspatial audio, bypasses reverb zones, and applies the chosen player volume. Outside a cinematic it uses spatial audio and consults consciousness and ghost state. `IsMuffled` can reflect either the speaker struggling or the living local listener struggling. The effect is deliberately listener dependent: the same remote speaker can sound different to two scouts because their own conditions differ.

`VoiceObscuranceFilter` gives muffling precedence over its obstruction linecast. It changes the low-pass cutoff and resonance for struggling, uses another low-pass target for an obstructed path, and changes the high-pass target when a voice is stolen. Distance and `LightVolume.SamplePositionAlpha` affect echo and the animator's obscurance value. A minimum obstruction value can be supplied externally for a frame and is cleared through `LateUpdate` when no writer refreshes it.

The transform provider chooses an override first, then a ghost's transform, then the character's head. `VoiceThief` sets and clears the override through `SetOverride`; the hazard's ownership and victim selection belong to the creatures chapter. This chapter concerns what happens after a source location has been selected.

Local mouth movement receives samples through `MicrophoneRelay`. Remote mouth movement reads the level measured by the voice handler. `AnimatedMouth.ProcessMicData` converts that level into a curve input, maintains an amplitude peak limiter and smoothed peak, and selects a talk sprite. It also suppresses the local mouth response when the selected push mode requests silence.

## Design decisions

Separating the voice location from the character transform is a useful choice. A ghost and a stolen voice can share playback machinery while choosing different origins. The filtering layer likewise lets a game condition alter sound without changing the transport.

The absent boundary is a single account of what the voice system is currently doing. Requested transmission, permitted transmission, playback mute, and mouth activity are represented separately. Those distinctions are legitimate, but their names and consumers need to preserve them. `transmitting` instead presents a requested state as though it described the recorder's final state.

## Where it's weak

### The voice-position smoother saves the wrong position

`CharacterVoiceTransformProvider.GetVoicePosition` computes a movement-limited result from `_lastTarget` toward the newly selected target shortly after an override changes. When `updateTarget` is true, however, it stores the target itself into `_lastTarget`, not the result it just returned. `CharacterVoiceHandler.LateUpdate` calls it with that option enabled for remote characters.

That breaks the intended progression across a large displacement. The first call returns a limited step from the previous position. Its saved state has already jumped to the destination, so the next call starts from the destination rather than the position actually returned. With a stationary target, the second call completes the jump regardless of the remaining distance. For a moving target, the calculation starts from the previous target sample, not from the previous playback position.

The actual consumer is the remote `AudioSource` transform. The limited value is not just a debug estimate: it is assigned to the object from which the voice plays. `VoiceThief` uses `SetOverride` without demanding an immediate jump, and clearing that override also enters the transition interval. Both directions depend on the provider retaining the transition's output.

The better state is the last emitted voice position, updated with the returned result. Target history can be stored separately if it is needed for another purpose. This implementation keeps target history in the slot that the interpolation treats as playback history. The movement limit survives only the first step of a stationary transition.

### The mute icon reads intent after the recorder has been denied

`CharacterVoiceHandler.PushToTalk` assigns `transmitting` from the selected mode and current input. Later in the same `Update`, failed communication permission or blocked player data forces `Recorder.TransmitEnabled` off and clears `m_currentlyTransmitting`. It does not clear `transmitting`.

`GUIManager` reads that public field directly to control its mute icon. In the denied state with transmission requested, the recorder is disabled while the icon's input still says transmission is active. The contradiction follows from two consumers reading different versions of the same decision. The restrictions are applied to the recorder; the presentation omits them.

There is another consequence inside the handler. Because the denial clears `m_currentlyTransmitting`, the next requested-transmission update sees a change again. It enables the recorder in `PushToTalk` and disables it again later in `Update`. This is repeated property churn for a condition already known to the handler. It does not establish that an audio packet escapes the restriction; the source shows competing writes within the update, not the recorder's scheduling.

Transmission intent should remain a distinct value, with the effective permission-and-input result assigned once to the recorder and exposed to the UI. A denial reason could be retained separately for settings or feedback. Here, the public state and the operational state disagree precisely when accurate feedback matters.

### Mouth levels depend on sample polarity

`CharacterVoiceHandler.OnAudioFilterRead` takes the largest signed sample in the received buffer and then squares it. That is not the largest sample magnitude: a large negative excursion can be ignored in favor of a smaller positive one. The local path in `AnimatedMouth.MicrophoneLevelMax` has the same directional bias in another form, rejecting samples below the local noise threshold before squaring the remaining samples.

The consumer is `AnimatedMouth.ProcessMicData`. Its curve, peak limiter, and talk threshold operate on the supplied level; they cannot recover an excursion that the measurement discarded. Two blocks with equal energy but different distributions between positive and negative samples need not produce the same mouth response. Local and remote measurements also use different sampling paths, making agreement between the local face and remote face a separate calibration problem.

This is a presentation measurement defect, not evidence that the transmitted audio itself is corrupted. The samples are still passed through the audio system. What loses information is the number used to animate the face.

An absolute peak or a windowed energy measurement would express the intended loudness proxy without favoring one polarity. The noise threshold should then be applied to that nonnegative measure. The choice between peak and energy would remain an artistic decision, but the measurement would have a defined relationship to amplitude. Squaring the largest signed value after selection does not provide that relationship.

### Cinematic filtering has no local restoration path

When `VoiceObscuranceFilter.Update` detects a character voice during a cinematic, it disables the high-pass, low-pass, reverb, and echo components and returns. Outside that branch, the same method continues adjusting their parameter values but does not re-enable them. `CharacterVoiceHandler.Update`, by comparison, assigns spatial blend and reverb-zone bypass in both cinematic and ordinary states.

For a surviving filter instance that exits cinematic mode, parameter updates therefore do not restore the components the cinematic branch disabled. The handler has reversible playback policy while the filter has a one-way component-state change. That difference matters to the consumer chain: struggling, obstruction, and stolen-voice calculations still execute, but updating a disabled filter's cutoff does not put that filter back into the signal path.

The proper restoration cannot simply enable every component unconditionally. `VoiceObscuranceFilter.Start` separately disables several effects in the airport. A transition must restore the applicable baseline, including that scene policy, rather than assume all filters started enabled. Capturing the baseline or explicitly assigning the desired enabled state in both modes would make the lifecycle local and inspectable. The current method relies on something outside itself to end the lifetime of the disabled state.

## For modders

Keep capture, transmission permission, and listener playback separate when investigating voice. `AudioLevelSlider` changes a per-player playback level through `AudioLevels`; it is not a microphone-gain control. Its update event reaches `CharacterVoiceHandler.UpdateAudioLevel`, while microphone selection reaches the recorder through a different path.

Spatial effects need the transform provider and filter policy inspected together. Changing a source's location changes the obstruction and distance inputs even when the audio samples are unchanged. Mouth animation is another consumer of voice, with its own measurement and suppression rules; it should not be treated as proof that the recorder is transmitting.

## Open questions

The serialized recorder configuration, codec settings, platform-specific capture implementation, and audio mixer routing need separate inspection before making latency or sound-quality claims. This chapter does not establish how often particular players encounter a microphone or permission problem.

The cinematic filter finding is conditional on an instance surviving the transition back to ordinary play. The relevant scene and object lifetimes have not been established here. External animation or component management could also restore enabled states; the local filter method does not.

The voice-location error has a direct source path, but its perceived sound depends on the displacement and audio configuration. The mouth-level finding likewise establishes a measurement bias without establishing how noticeable it is during typical speech.
