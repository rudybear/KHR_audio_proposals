# Structured Feedback for Extension Authors

Consolidated, actionable items from [02](02-gap-analysis-audio-emitter.md)/[03](03-gap-analysis-audio-graph.md)/[04](04-usd-comparison.md). Each item is phrased so it can be pasted as a GitHub review comment or issue. IDs reference the detail docs.

## For KHR_audio_emitter (PR #2137)

**Blocking (should resolve before ratification):**
1. **[E3] Define channel behavior for positional emitters.** Suggested text: "A positional emitter MUST down-mix its input to mono per the Web Audio API mixing rules prior to spatialization. A global emitter presents source channels as authored." (Closes the 2022 mono/stereo thread.)
2. **[E1] State the playback-control story.** Either add start/stop semantics to the Object Model table (a writable `playing`/`state`) or normatively reference a KHR_interactivity audio-node companion. Cross-link the answer from the README so implementors stop asking.
3. **[E2] Add one gapless-loop-capable codec.** MP3 cannot loop seamlessly (acknowledged in-thread). Propose Opus (RFC 6716/7845) as a second base mimeType, or bless OMI_audio_opus normatively.

**Should-fix:**
4. **[E10] Adopt the KHR_lights_punctual scale rule** ("node scale does not affect emitter gain or distances; distances are world-space") and close issue #2162 with it.
5. **[E6] Add `priority`** to source or emitter; harmonize scale with the graph PR (currently 0–256 there, [0,1] in X3D) — one scale, one polarity, defined ordering, implementation-defined culling.
6. **[E9] Fix the linear-model/maxDistance=0 conflict**: require explicit `maxDistance > refDistance` when `distanceModel == "linear"`.
7. **[E8] One sentence defining the default listener** (active camera/viewer pose) with a pointer to KHR_audio_environment for overrides.
8. **[U-OV] `loopCount`** (int, 0 = infinite) — matches Omniverse/engines, trivial.

**Editorial:**
9. **[E4/E5]** Document why loopStart/loopEnd/detune are layered out of the base (or lift loop points in).
10. **[E12]** Close the naming thread (Web Audio names vs lights_punctual style) with a recorded rationale.

## For KHR_audio_graph (PR #2572)

**Blocking:**
1. **[G2] `custom` oscillator has no data model.** Add `periodicWave {real[], imag[]}` (X3D/Web Audio PeriodicWave) or remove `custom` from the enum. Same treatment for `custom` gain-interpolation and the waveshaper `amount`→curve mapping.
2. **[G8] Resolve extended-source-property placement.** loopStart/loopEnd/offset/when/duration/priority are playback semantics, not graph semantics — move to the base (or a KHR_audio_playback micro-extension); fix the `playbackRate` duplication with #2137; define `state` transitions or replace with interactivity verbs. Must be settled *jointly* with #2137.
3. **[G11] De-duplicate the bundled KHR_audio_emitter copy** (stale singular `emitter` schema vs #2137's plural `emitters`); depend on #2137 instead of vendoring it.

**Should-fix:**
4. **[G1] Add a `compressor` node** (Web Audio/X3D DynamicsCompressor params). Only roster gap vs *both* benchmark standards.
5. **[G3] Decide feedback cycles.** Recommend Web Audio's rule: cycles permitted iff each contains ≥1 `delay` node (min one block of latency); otherwise keep DAG and document why.
6. **[G10] One paragraph on implicit mixing**: fan-in summing follows Web Audio mixing rules, `speakers` interpretation unless a channel node overrides.
7. **[U1] Define the playback clock.** `when` is seconds from — load? scene start? Recommend anchoring schedulable playback to a glTF animation timeline (USD's startTime-in-stage-time is the precedent that composes with animation).

**Editorial:**
8. **[G4/G5/G7]** Add "decision recorded" paragraphs: no audio-rate param connections (reserve `to:{node,param}` syntax), no envelopes (gain smoothing is the mechanism), graphs are pre-spatialization (master-bus inserts live in KHR_audio_environment).
9. **[G11]** Gain-range wording (README `[0,+inf)` vs proposal `(0,+∞)` vs schema `min 0`), `connections` required-but-empty, restore the "Optional: KHR_audio_environment" line when that PR exists.
10. **CLA signatures** (3 of 6 outstanding) and contributor attribution per Norbert's comment.

## Joint / working-group items

1. **Publish the layer map in both PRs**: emitter = X3D Level 1 analog, graph = Level 2, environment = Level 3. Resolves the "#2421 graph vs #2137 emitter — competing or layered?" ambiguity that has stalled OMI.
2. **One units-and-conventions appendix shared by all three specs** (seconds, Hz, radians, linear gain except filter-shelf dB, -Z forward, priority scale).
3. **Coordinate with KHR_interactivity** on audio verbs/events (play/stop/onEnded) — repeatedly requested, unowned.
4. **Validator rules** now: emitter type vs positional-object presence, linear-model maxDistance, graph acyclicity (or delay-cycle rule), port-index bounds, oneOf uri/bufferView.
