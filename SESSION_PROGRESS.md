# Session Progress: glTF Audio Extension Layered Architecture

## Date: 2026-02-07

---

## 1. Specifications Analyzed

### KHR_audio_emitter (OMI Group — base layer)
- **Source**: https://github.com/omigroup/glTF/blob/5079aeef2c59d44a138a9e33b1767224ff13d449/extensions/2.0/Khronos/KHR_audio_emitter/README.md
- **Status**: Draft
- **Model**: Flat 3-tier: `audio[] → sources[] → emitters[] → nodes/scenes`
- **Features**: Audio data (uri/bufferView, audio/mpeg), sources (gain, playbackRate, loop, autoplay), emitters (global/positional), positional properties (distance models, cone directionality), glTF Object Model JSON pointers
- **Conventions**: Seconds for time, radians for angles, gain (0, +∞)

### KHR_audio_graph (Meta/Facebook — processing layer)
- **Source**: https://github.com/facebook/glTF/blob/main/extensions/2.0/Khronos/KHR_audio_graph/README.md
- **Status**: Draft
- **Model**: DAG-based: `audioData[] → graphs[].nodes[] → connections[] → emitter sinks`
- **Features**: 16+ node kinds (source, gain, delay, 8 filter types, reverb, waveshaper, channel ops, emitter), oscillator sources, encoding properties, bypass, Web Audio API mapping
- **Conventions**: Milliseconds for time (inconsistent), degrees for angles (unclear), gain [0,1]

---

## 2. Gap Analysis Completed

Full analysis document: `analysis/spec_comparison_and_gap_analysis.md`

### 23 Findings identified, including:
- **F4**: Gain range conflict — [0,1] vs (0,+∞) → resolved: adopt audio_emitter's (0,+∞)
- **F5/F6**: Naming conflicts — playbackRate vs playbackSpeed, autoplay vs autoPlay → resolved: adopt audio_emitter naming
- **F8**: Oscillator support gap — audio_emitter has no oscillators → resolved: graph-only nodes
- **F10**: Type naming — "positional" vs "spatial" → resolved: keep "positional"
- **F11**: Source binding model — direct reference vs graph connections → resolved: document-level graphs with explicit bindings
- **F13**: Angle units — radians vs degrees → resolved: use radians
- **F14**: Spatialization model gap → resolved: environment layer
- **F20**: No listener in audio_emitter → resolved: environment layer
- **F21**: Single vs multiple emitters per node → resolved: base spec change to array

---

## 3. Design Decisions (All 7 Questions Resolved)

| # | Question | Decision |
|---|---|---|
| 1 | Oscillator source model | Graph-only node |
| 2 | Graph insertion point | Document-level with explicit input/output bindings (option C) |
| 3 | Reverb placement | Environmental/listener layer (generalized) |
| 4 | Spatialization model | Environmental/listener layer |
| 5 | Multiple emitters per node | Base spec change: `emitter` → `emitters` (array) |
| 6 | Unit convention | Seconds + radians (audio_emitter conventions) |
| 7 | Naming alignment | Full adoption of audio_emitter naming |

---

## 4. Specs Drafted

### `specs/KHR_audio_emitter_changes.md`
- Single change: node-level `emitter` (integer) → `emitters` (integer array)
- Schema change, migration notes, Object Model update

### `specs/KHR_audio_graph.md` — Full Layer 1 spec
- Extension of KHR_audio_emitter (required dependency)
- Document-level `graphs[]` with `nodes[]`, `connections[]`, `inputs[]`, `outputs[]`
- **Graph inputs**: bind audio_emitter sources to graph entry nodes
- **Graph outputs**: bind graph exit nodes to audio_emitter emitters
- **16 node kinds**: oscillator (source), gain, delay, waveshaper, 8 filters, 4 channel routing nodes
- **Rule**: When emitter is graph output, its `sources[]` in audio_emitter is ignored
- Encoding properties as extension on audio_emitter audio data
- Extended source properties as extension on audio_emitter sources
- Bypass, Web Audio mapping, JSON pointers for animation
- All conventions: seconds, radians, gain (0,+∞), audio_emitter naming

### `specs/KHR_audio_environment.md` — Full Layer 2 spec
- Extension of KHR_audio_emitter (required), optionally KHR_audio_graph
- **Listener**: bound to camera/node, spatializationModel (equalpower/HRTF/custom), HRTF config, interauralDistance
- **Environment**: bound to scene (global) or node (zone), reverb (parametric or impulse response)
- **Custom distance models**: distanceCurve array on emitter positional extension
- **Per-emitter spatialization override**: via extension on positional properties
- Parametric reverb: mix, roomSize, reflectivity, decayTime, earlyReflections, etc.
- IR reverb: references audio_emitter audio data
- JSON pointers for animation

---

## 5. Code Project Analyzed

### AudioGraphJS (https://github.com/rudybear/AudioGraphJS)
- Cloned to `/Users/alexeymedvedev/Desktop/sources/audiograph2/AudioGraphJS`
- TypeScript, ES2020, Node 18+, Web Audio API via standardized-audio-context + web-audio-engine
- 14 node types fully implemented
- Graph builder (sync + async), bypass system, emitter instance expansion
- 31 runtime + 31 KHR example graphs with parity validation
- Full CI with trace comparison and WAV checksum verification

---

## 6. Implementation Plan Created

### Plan file: `~/.claude/plans/luminous-tumbling-panda.md`
### Coding prompt: `CODING_AGENT_PROMPT.md`

### Strategy: Keep runtime model intact, add serialization layer

**Files to modify:**
- `src/types.ts` — Add layered extension interfaces
- `src/index.ts` — Add new exports
- `src/serialization/gltf-emitters.ts` — Support KHR_audio_emitter extension
- `src/runtime/emitters.ts` — Add extension-based variant
- `src/runtime/lint.ts` — Add layered validation
- `examples/run-graph.mjs` — Add layered format detection

**New files:**
- `src/serialization/parse-layered.ts` — Core layered → runtime conversion
- `src/runtime/environment.ts` — Environment/reverb application
- `src/runtime/listener.ts` — Listener configuration
- `examples/graphs-layered/*.json` — 4 example files
- `tools/spec-validate/validate-layered.mjs` — Layered format validator

**No changes to:**
- All 14 node factories in `src/nodes/`
- `buildGraph.ts` / `buildGraphAsync.ts`
- Bypass system
- Existing examples

---

## 7. Next Steps

- [ ] Start coding agent with `CODING_AGENT_PROMPT.md`
- [ ] Implement all phases (types → serialization → runtime → examples → validation)
- [ ] Build and test
- [ ] Review spec proposals if changes needed during implementation
- [ ] Iterate on specs based on implementation feedback

---

## Session 2: 2026-08-02 — Environment v2 (branch `feature/environment-v2`)

### Research pass completed
- Gap analyses of KHR_audio_emitter (#2137) and KHR_audio_graph (#2572) vs W3C Web Audio 1.0/1.1, X3D 4.0 (ISO/IEC 19775-1:2023) Sound component, USD ecosystem (core UsdMediaSpatialAudio, Omniverse, Apple RealityKit/PHASE), and engine baseline (Wwise/FMOD/Steam Audio/Project Acoustics/MPEG-I).
- Analysis docs live in the parent workspace: `../0{1..6}-*.md` (current state, emitter gaps E1–E12, graph gaps G1–G11, USD comparison, environment proposals P1–P14, structured feedback).

### Design decisions (user-confirmed)
1. Reverb vocabulary: abstract/generic parameter set (I3DL2-aligned, glTF units) + named presets; detailed models attach via `extensions` on reverb/environment.
2. Scope: Tier 1 (listener lifecycle, reverb+presets, per-emitter direct/reverb sends, normative zones) **plus** P5 Doppler, P6 listener-bus graph hook, P7 air absorption, P8 cone low-pass.
3. Zones: box + sphere with blendDistance/priority; mesh shapes via future extensions.
4. HRTF: keep audio[] + profile; SOFA (AES69) named in future work.

### Spec rewritten: `specs/KHR_audio_environment.md`
- Normative listener lifecycle (activeListener → active-camera binding → first binding → implicit viewer listener), listener gain, HRTF fallback rule.
- Reverb: preset + decayTime/decayHFRatio/reflectionsGain+Delay/reverbGain+Delay/diffusion/density/mix; IR mode with normalize; informative preset value table.
- Doppler per environment (enabled/scale/speedOfSound) + normative pitch formula + per-emitter opt-out.
- Emitter integration: directLevel/reverbLevel sends + forced environment; positional: spatialization override, distanceCurve, airAbsorption, coneOuterCutoff.
- Zones: shape (box/sphere), blendDistance, priority, normative listener-position selection rules.
- Listener-bus graph hook (§3.5) for master processing via KHR_audio_graph.
- Updated Object Model pointers; Future Work section (ambient beds, acoustic materials, rooms/portals, voice mgmt, SOFA, mesh zones, AR).

### Schemas added: `specs/schema/KHR_audio_environment/` (12 files, all validated)

### Next
- [ ] Prototype v2 features in AudioGraphJS branch `feature/environment-v2`
- [ ] Iterate spec from implementation feedback; then convert to glTF-repo PR layout
