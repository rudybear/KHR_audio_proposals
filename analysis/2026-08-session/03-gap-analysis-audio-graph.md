# Gap Analysis: KHR_audio_graph (PR #2572)

Benchmarks: W3C Web Audio API 1.0 node set + AudioParam model; X3D 4.0 Sound component (the existing ISO declarativization of Web Audio); engine mixing architecture (buses, sends, ducking). Severity: **H**/**M**/**L** as in [02](02-gap-analysis-audio-emitter.md).

## What is already right (messaging ammunition)

- Document-level `graphs[]` with explicit `inputs[]`/`outputs[]` bindings is *cleaner than X3D's* destination-rooted `children` fan-in encoding: connections are first-class, fan-in/fan-out are symmetric, and graphs are reusable across sources/emitters. This is a genuine improvement over the ISO precedent, worth saying out loud.
- Node roster maps 1:1 to Web Audio informatively (filters → BiquadFilterNode etc.), so every construct has a browser-native realization *and* an X3D/ISO analog — the "expressible as an audio standard" story holds.
- Units harmonized with the emitter layer (seconds/Hz/linear gain; dB only for shelf/peaking filter gain, matching Web Audio) — the old Meta-draft ms/[0,1] inconsistencies are resolved.
- Layering discipline: reverb, spatialization, and listener deliberately pushed to the environment layer; scene attachment stays in the base. Rule 12 (graph-bound emitter ignores its `sources`) is a clean override semantic.
- Object Model pointers on node params (gain, frequency, delayTime, Q, detune, bypass) give k-rate animation via KHR_animation_pointer/KHR_interactivity — X3D has no declarative equivalent short of ROUTEs.

## Gaps

### G1. No DynamicsCompressor node — **H**
Both Web Audio and X3D 4.0 have it (identical params: threshold/knee/ratio/attack/release). It is the workhorse for mastering, limiting, and (with a sidechain) ducking — B-tier engine table stakes. Its absence is the single most visible hole in the roster vs both benchmark standards.
**Recommendation:** add `compressor` kind with the Web Audio/X3D param set (threshold −24 dB, knee 30, ratio 12, attack 0.003 s, release 0.25 s). Sidechain input can wait (Web Audio can't do it natively either — parity preserved).

### G2. `custom` oscillator type has no data model — **H** (spec bug)
The enum value exists but no PeriodicWave payload is defined. Web Audio: PeriodicWave (real/imag Fourier coefficients); X3D: PeriodicWave node with optionsReal/optionsImag.
**Recommendation:** either add `periodicWave: {real: [...], imag: [...]}` to oscillator params or drop `custom` from the enum until defined. Same for `custom` gain-interpolation curve (undefined payload) and waveshaper `amount`→curve mapping (implementation-defined — give a normative reference curve or make `curve` the only mechanism).

### G3. No cycles = no feedback delay — **M/H**
DAG-only forbids the classic feedback-echo topology. Web Audio *allows* cycles when the loop contains a DelayNode (which enforces ≥ one render quantum of latency); X3D inherits that.
**Recommendation (design decision to make explicitly):** either adopt the Web Audio rule ("cycles are permitted iff every cycle contains at least one `delay` node; delayTime is clamped to ≥ one processing block") or keep DAG-only and state the rationale + that feedback is future work. Silence here will read as an oversight; the Web Audio rule is cheap to adopt and validators can check it.

### G4. No audio-rate parameter modulation — **M** (document, likely defer)
Web Audio's connect-to-AudioParam (LFO → gain, envelope followers, sidechains) is its deepest expressive feature. X3D also chose *not* to declarativize it — good precedent for deferring. But the spec should say so: "node outputs carry audio only; parameter-input connections are reserved for a future extension." Consider reserving a `to: {node, param}` connection form in the schema design now so it can be added compatibly.

### G5. No envelopes / scheduled automation — **M** (document, defer)
Only gain `interpolation` + `duration` smoothing exists; no ADSR, no setTargetAtTime-style curves. Animation-pointer covers k-rate control from the scene side; sample-accurate envelopes are the remaining gap vs Web Audio. Acceptable for v1 — state it, and note the gain node's smoothing model is the sanctioned mechanism.

### G6. Missing lesser nodes vs Web Audio/X3D — **L** (triage list)
- `constant` source (Web Audio ConstantSourceNode): only useful with G4 — defer together.
- `stereopanner`: expressible via splitter/gain/merger; defer.
- `iirfilter`: niche, coefficients not animatable even in Web Audio; defer.
- `analyser`: runtime introspection, not asset semantics; correctly excluded — say so.
- `channelselector` (X3D addition, 1 channel out of N): cheap and genuinely useful for multichannel assets; splitter technically covers it. Optional.
- noise source (engines): defer; oscillator+waveshaper can't fake it, but no standard precedent (neither Web Audio nor X3D has one).
- `convolver` as *generic* node: reverb moved to environment (right call), but convolution has non-reverb uses (speaker/cabinet IRs, HRTF-ish effects). Defer, but note that the environment layer's IR-reverb machinery could later be generalized.

### G7. Emitter-level graph insert ambiguity — **M**
`inputs[]` bind *sources* and `outputs[]` bind *emitters* — so processing happens pre-spatialization, and there is no post-spatialization/master-bus insert point (where engines put master compressors/EQ). Global emitters bound as graph outputs partially cover this (their signal is post-"emission").
**Recommendation:** state the model plainly: "graphs process source signals before emitter gain/spatialization; master-bus processing is the domain of KHR_audio_environment." And make sure the environment draft actually reserves a master/listener-bus insert (see [05](05-KHR_audio_environment-proposals.md), P6).

### G8. Extended-source-property placement and duplication — **M**
`loopStart/loopEnd/offset/when/duration/playbackRate/priority/state` live in the graph extension's source payload, but none of them are graph-specific — they're playback semantics (see emitter gap E4/E6/E1). `playbackRate` now exists in *both* the base source and the graph extension. `state` as a mutable Object Model string is novel and under-specified (allowed transitions? what does writing "playing" do at t=5s?).
**Recommendation:** move loop/offset/when/duration and priority down into KHR_audio_emitter (or a small KHR_audio_playback companion); define `state` transitions normatively or replace with interactivity verbs. Resolve the playbackRate duplication before either PR merges.

### G9. Encoding metadata is graph-layer but universally useful — **L**
`sampleRate`/`channels` etc. on `audio[]` entries lives in the graph extension, yet a base-only implementation also benefits (preallocation, validation). Consider moving to the base as optional metadata. Also: it duplicates what the decoded file already knows — state the precedence rule (file wins? metadata must match?). Currently unstated; a validator can't flag mismatches without it.

### G10. Channel model is thinner than Web Audio's — **L** (document)
No per-node channelCount/channelCountMode; interpretation enums only on the 4 channel nodes. Probably right for a declarative format — but add one normative paragraph: implicit fan-in summing uses Web Audio mixing rules with `speakers` interpretation unless a channel node says otherwise. (Rule 8 says summing happens; it doesn't say how channels mix.)

### G11. Spec-hygiene items — **L**
- Gain range README `[0,+inf)` vs proposal `(0,+∞)` vs schema `minimum: 0.0` — align (inclusive 0 is correct; 0 = mute).
- `connections` required-but-may-be-empty vs README "required" wording.
- PR README dropped "Optional: KHR_audio_environment" dependency line present in the proposal repo — re-add once the environment PR exists.
- Bundled KHR_audio_emitter copy is stale vs #2137 (singular `emitter` in node schema/prose vs plural `emitters`) — drop the copy, depend on #2137.
- 13 graph rules: renumber/regroup as (topology, channels, binding) for reviewability.

## Feedback priority order for PR #2572

1. G2 custom-oscillator payload (spec bug — fix or drop enum value)
2. G1 compressor node (parity with both benchmark standards)
3. G8 extended-source-props placement + `state` semantics + playbackRate duplication (must be settled jointly with #2137)
4. G3 feedback-cycle decision (adopt Web Audio delay-cycle rule or justify DAG)
5. G11 sync bundled emitter with #2137 / drop the copy
6. G4/G5/G7/G10 "document the decision" paragraphs — cheap, and they preempt WG review questions
