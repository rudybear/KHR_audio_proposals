# Gap Analysis: KHR_audio_emitter (PR #2137)

Benchmarks: W3C Web Audio API 1.0 (REC 2021) PannerNode/AudioBufferSourceNode, X3D 4.0 (ISO/IEC 19775-1:2023) Sound component, plus the modern-engine baseline where relevant. Severity: **H** = blocks mainstream use cases or interop; **M** = expected by modern runtimes, workable without; **L** = polish/nice-to-have.

## What is already at parity (messaging ammunition)

- Distance model (linear/inverse/exponential + refDistance/maxDistance/rolloffFactor) and cone model (inner/outer angle, outerGain) are **verbatim Web Audio PannerNode semantics**, which are also X3D 4.0 SpatialSound semantics. Formulas are identical across all three.
- Radians + -Z forward matches glTF conventions (KHR_lights_punctual, cameras) — a deliberate, defensible divergence from Web Audio's degrees/+X.
- Split of `audio` (data, image-like) / `sources` (playback settings) / `emitters` (spatial behavior) is cleaner than X3D (which fuses source+panner in SpatialSound) and strictly richer than USD's SpatialAudio.
- Object Model pointer table (animatable gain, cone, distances via KHR_animation_pointer / KHR_interactivity) — X3D needs ROUTEs for this; USD can only time-sample `gain`.
- Multiple sources per emitter and multiple emitters per node/scene = basic declarative mixing.

## Gaps

### E1. Event-driven playback / interactivity integration — **H**
Only `autoplay` + `loop` exist. No play/stop/pause verbs, no "playback finished" signal. KHR_interactivity audio nodes (e.g. `audio/start`, `audio/stop`, an `onEnded` event) are undefined — raised in PR discussion (Apr 2025), unanswered. X3D gives every source declarative transport (startTime/stopTime/pauseTime/resumeTime + isActive/elapsedTime events); engines are 100% event-driven; USD has playbackMode + startTime/endTime tied to the stage timeline.
**Recommendation:** keep the base data-only, but (a) add `state` or start/stop semantics to the Object Model pointer table (the graph PR already smuggles in a mutable `state` string pointer — this belongs in the base or in a KHR_interactivity companion), and (b) publish a companion KHR_interactivity node set as part of the layered-architecture message. This is the #1 question every engine implementor will ask.

### E2. Codec: MP3-only — **H**
MP3 cannot loop gaplessly (encoder padding — acknowledged in-thread, Oct 2025), caps at 2 channels, and is poor for ambient beds. WAV was removed (no formal spec — defensible). But a *looping-capable, royalty-free* codec is a base requirement for the primary use case (looping ambience/SFX).
**Recommendation:** promote Opus (RFC 6716, in Ogg per RFC 7845) into the base spec alongside MP3, or fold OMI_audio_opus in as a KHR-blessed layered extension referenced normatively from the README. Messaging point: MP3 = ubiquity/decode-everywhere, Opus = looping/multichannel/quality.

### E3. Channel semantics for positional emitters undefined — **H**
What happens when a stereo (or 5.1) source feeds a positional emitter? Web Audio's PannerNode downmixes to mono-equivalent before spatialization (clamped-max 2); X3D has an explicit `spatialize` flag. Discussed in 2022 ("positional MUST be mono") but never landed in the spec.
**Recommendation:** normative text: positional emitters downmix input to mono per Web Audio mixing rules before spatialization; global emitters play channels as authored. One sentence closes the hole.

### E4. Loop points (`loopStart`/`loopEnd`) missing from base — **M**
Web Audio AudioBufferSourceNode and X3D BufferAudioSource both have them; engines have loop regions. The graph extension adds them as *extended source properties*, which means a base-only implementation can't express a loop region.
**Recommendation:** either lift loopStart/loopEnd (+ `offset`) into the base source, or explicitly document the layering decision ("loop regions require KHR_audio_graph") so it reads as intent, not omission.

### E5. `playbackRate` couples pitch and speed; no `detune` — **L/M**
Web Audio and X3D expose `detune` (cents) alongside playbackRate. Fine to defer, but state the rationale (avoids mandating a resampler-independent pitch shifter in the base).

### E6. No priority / voice management — **M**
X3D SpatialSound has `priority` [0,1]; the graph's extended source props have `priority` 0–256 (0 = highest); Omniverse has per-sound priority + stage-level Concurrent Voices; every engine virtualizes voices. Two problems: it's absent from the base, and the two proposals that do have it disagree on range/polarity.
**Recommendation:** pick one scale (suggest X3D-style normalized, or document 0–256 as Wwise-like), put it in the base emitter or source, and state that voice-limit behavior is implementation-defined but priority ordering is normative.

### E7. No Doppler anywhere in the stack — **M**
Web Audio removed it, but X3D 4.0 re-added `dopplerEnabled` (Level 3), MPEG-I includes it, Omniverse has per-sound enable + global scale/limit. Currently only a non-normative "implementors may add effects" note.
**Recommendation:** don't put it in the emitter; assign it to KHR_audio_environment (listener-relative concern) and say so in the emitter README's future-work section. See [05](05-KHR_audio_environment-proposals.md).

### E8. Listener is never defined — **M**
The base spec spatializes "relative to the listener" without defining one. Web Audio has AudioListener (position/forward/up); X3D tracks the active viewpoint and additionally offers ListenerPointSource.
**Recommendation:** one normative default in the base ("the listener is the active camera / viewer pose; extensions may override"), full listener object in KHR_audio_environment.

### E9. `maxDistance` default 0 (= no maximum) vs linear model constraint — **L**
Linear model requires maxDistance > refDistance, but the default (0) violates that; the "0 = infinite" sentinel also diverges from Web Audio (default 10000, no sentinel). Add a validation rule: `distanceModel == "linear"` requires explicit `maxDistance > refDistance`; define behavior otherwise.

### E10. Node-scale behavior undefined — **M**
Punted to [issue #2162](https://github.com/KhronosGroup/glTF/issues/2162). Non-uniform scale on an emitter node, and AR "scene scale," change audible results across implementations. KHR_lights_punctual-style "distances are in world units after transform; scale does not affect gain" is one sentence — adopt it.

### E11. No source extent / spread — **M** (defer, but reserve)
Point sources "snap" across the head at close range. Engines: spread/focus curves; MPEG-I: source extent; X3D classic Sound: asymmetric ellipsoid (minBack/minFront/maxBack/maxFront). The `shapeType` enum is already extensible — good. Reserve `"extent"`/spread for the environment layer or a future shape; name it in future-work so reviewers see it was considered.

### E12. Property naming vs glTF-family conventions — **L**
donmccurdy's open point: `coneInnerAngle`/`maxDistance` (Web Audio verbatim) vs `innerConeAngle`/`range` (KHR_lights_punctual). Either is defensible; decide once, document rationale ("names track Web Audio for implementor familiarity; units track glTF"), and close the thread.

## X3D features consciously *not* mirrored (document as rationale, not gaps)

- `intensity` separate from `gain` (redundant), MIDI, MicrophoneSource/StreamAudioSource/StreamAudioDestination (live-I/O is runtime, not asset, territory), Analyser (runtime introspection), per-node `description` strings (glTF has `name`), declarative transport on *every* node (covered by Object Model + interactivity instead).

## Feedback priority order for PR #2137 authors

1. E3 channel rule (one sentence, unblocks conformance tests)
2. E1 interactivity/playback answer (coordinate with KHR_interactivity WG)
3. E2 codec (Opus normative reference)
4. E10 scale rule (adopt lights-punctual precedent)
5. E6 priority (harmonize with graph PR before either merges)
6. E4/E8/E9/E12 editorial passes
