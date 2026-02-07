# KHR_audio_emitter + KHR_audio_graph: Comparison & Gap Analysis

## 1. Architecture Overview

### KHR_audio_emitter (Base)
A flat, 3-tier model:
```
audio[] → sources[] → emitters[] → (bound to nodes/scenes)
```
- **audio**: raw data location (uri/bufferView)
- **sources**: playback config referencing audio data
- **emitters**: spatial/global output referencing sources, bound to glTF nodes or scenes

### KHR_audio_graph (Extension)
A DAG-based processing pipeline:
```
audioData[] → graphs[].nodes[] → connections[] → emitter sinks
```
- **audioData**: raw data with encoding metadata
- **graphs**: contain nodes (source/processor/emitter), connections, and outputs
- Source nodes reference audioData or inline oscillator data
- Emitter nodes act as terminal sinks

### Key Architectural Tension
**audio_emitter** connects sources directly to emitters (no intermediate processing).
**audio_graph** places an arbitrary processing chain between source and sink.

The merge strategy must preserve audio_emitter's direct `source → emitter` path while allowing an optional graph to be inserted between them.

---

## 2. Audio Data Comparison

| Property | audio_emitter | audio_graph | Notes |
|---|---|---|---|
| **Array name** | `audio` | `audioData` | **Naming conflict** |
| **uri** | Yes | Yes | Same semantics |
| **bufferView** | Yes | Yes | Same semantics |
| **mimeType** | Yes (required if bufferView) | Yes (required if bufferView) | Same |
| **Supported MIME** | `audio/mpeg` only (base spec) | Not restricted | audio_emitter is more restrictive |
| **encodingProperties** | **Not supported** | Yes (bitsPerSample, duration, samples, sampleRate, channels) | **GAP**: audio_graph provides metadata needed for buffer-level processing |

### Findings

- **F1 — NAMING**: `audio` vs `audioData`. Since audio_emitter is the base, its `audio` naming stays. The graph extension would reference audio_emitter's `audio[]` array directly, no duplication needed.

- **F2 — ENCODING PROPERTIES GAP**: audio_emitter has no encoding metadata. For graph-based processing (channel splitting, sample-rate-dependent filters), this metadata is essential. **Proposal**: encoding properties could be:
  - (a) Added as an optional property to audio_emitter's audio objects (minor base spec change), OR
  - (b) Defined as an extension property on audio_emitter's audio objects via the graph extension

- **F3 — MIME TYPE RESTRICTION**: audio_emitter only mandates `audio/mpeg`. Audio graph processing (especially oscillators, real-time synthesis) may need `audio/wav`, `audio/ogg`, or raw PCM. The graph extension may need to expand supported MIME types.

---

## 3. Audio Source Comparison

| Property | audio_emitter | audio_graph (source node) | Notes |
|---|---|---|---|
| **gain** | `gain` (linear, default 1.0, range 0..+∞) | `gain` (linear, range [0..1]) | **CONFLICT: range differs** |
| **playback rate** | `playbackRate` | `playbackSpeed` | **NAMING conflict** |
| **loop** | `loop` (boolean) | `loop` (boolean) | Same |
| **autoplay** | `autoplay` | `autoPlay` | **NAMING conflict** (camelCase difference) |
| **audio data ref** | `audio` (index into `audio[]`) | `data.audioData` (index into `audioData[]`) | Different reference mechanism |
| **loopStart** | Not supported | Yes (ms) | **GAP** |
| **loopEnd** | Not supported | Yes (ms) | **GAP** |
| **offset** | Not supported | Yes (ms) | **GAP** |
| **when** | Not supported | Yes (seconds!) | **GAP** + unit inconsistency within audio_graph itself |
| **duration** | Not supported | Yes (ms) | **GAP** |
| **priority** | Not supported | Yes (0-256) | **GAP** |
| **state** | Not supported | Yes (paused/playing/stopped) | **GAP** |
| **channelInterpretation** | Not supported | Yes (speakers/discrete) | **GAP** |
| **oscillator data** | **Not supported** | Yes (type, frequency, pulseWidth) | **MAJOR GAP** |

### Findings

- **F4 — GAIN RANGE CONFLICT (FUNDAMENTAL)**: audio_emitter allows gain `(0, +∞)`, while audio_graph restricts to `[0, 1]`. This is a fundamental semantic difference. audio_emitter's approach is more flexible (allows amplification beyond original volume). Since audio_emitter is the base and harder to change, **the graph extension should adopt audio_emitter's gain range convention** `(0, +∞)`.

- **F5 — NAMING: playbackRate vs playbackSpeed**: audio_emitter uses `playbackRate`, which aligns with Web Audio API's `AudioBufferSourceNode.playbackRate`. The graph extension should adopt `playbackRate` to match the base spec.

- **F6 — NAMING: autoplay vs autoPlay**: audio_emitter uses lowercase `autoplay`, which matches the HTML `<audio>` element convention. The graph extension should adopt `autoplay`.

- **F7 — MISSING SOURCE PROPERTIES**: audio_emitter sources lack `loopStart`, `loopEnd`, `offset`, `when`, `duration`, `priority`, `state`, `channelInterpretation`. These are NOT needed to change in the base spec — they can be expressed as:
  - (a) Extension properties on audio_emitter sources (via `extensions.KHR_audio_graph`), OR
  - (b) Overrides within the graph node that references the source

- **F8 — OSCILLATOR SUPPORT (MAJOR GAP)**: audio_emitter has no concept of procedural audio generation. Oscillators need to be introduced as a new source type. Options:
  - (a) Add oscillator as a new audio_emitter source type (requires base spec change)
  - (b) Define oscillator as a graph-only source node that can connect to audio_emitter emitters
  - (c) Define a separate small extension `KHR_audio_source_oscillator` that extends audio_emitter sources

  Since audio_emitter sources always reference an `audio` data index, oscillators fundamentally break this model. **This likely requires either a base spec change or a separate extension.**

- **F9 — UNIT INCONSISTENCY IN audio_graph**: The `when` property is documented as "seconds" while all other timing is in milliseconds. This needs to be resolved. Recommendation: align everything to seconds to match Web Audio API, OR to milliseconds for internal consistency.

---

## 4. Emitter Comparison

| Property | audio_emitter | audio_graph (emitter node) | Notes |
|---|---|---|---|
| **Type field** | `type` | `emitterType` | **NAMING conflict** |
| **Type values** | `global`, `positional` | `global`, `spatial` | **VALUE conflict** |
| **gain** | `gain` (linear, 0..+∞, default 1.0) | `gain` (linear, [0, 1]) | **RANGE conflict** (same as F4) |
| **Sources** | `sources` (array of source indices) | N/A (graph connections) | Different source binding model |
| **Spatial sub-object** | `positional` | `spatialProperties` | **NAMING conflict** |
| **channelInterpretation** | Not supported | Yes | **GAP** |

### Findings

- **F10 — TYPE NAMING: "positional" vs "spatial"**: audio_emitter uses `positional`, audio_graph uses `spatial`. Since audio_emitter is the base, `positional` stays. However, `spatial` is arguably more accurate (it includes directionality, not just position). **Decision needed: keep `positional` or propose rename?**

- **F11 — SOURCE BINDING MODEL (FUNDAMENTAL)**: In audio_emitter, emitters directly reference sources via an index array (`"sources": [0, 1]`). In audio_graph, emitters are graph sinks connected via explicit edge connections. The graph extension needs to define how a graph's output connects to an audio_emitter emitter:
  - (a) The graph replaces the `sources` array — emitter's input comes from the graph instead
  - (b) The graph is inserted between existing sources and emitters — sources feed into the graph, graph output feeds the emitter
  - (c) The graph is declared on the emitter via an extension property

- **F12 — GAIN RANGE**: Same issue as F4. Graph extension should adopt audio_emitter's `(0, +∞)` range.

---

## 5. Spatial/Positional Properties Comparison

| Property | audio_emitter (positional) | audio_graph (spatialProperties.attenuation) | Notes |
|---|---|---|---|
| **Nesting** | Flat under `positional` | Nested: `spatialProperties.attenuation` | Structural difference |
| **distanceModel** | `linear`, `inverse`, `exponential` | `linear`, `inverse`, `exponential`, `custom` | audio_graph adds `custom` |
| **maxDistance** | Yes (default 0.0 = unlimited) | Yes | Same semantics |
| **refDistance** | Yes (default 1.0) | Yes | Same semantics |
| **rolloffFactor** | Yes (default 1.0) | Yes | Same semantics |
| **Shape type** | `shapeType` (`omnidirectional`, `cone`) | `shape` (`cone`, `omnidirectional`, `custom`) | **NAMING** + audio_graph adds `custom` |
| **coneInnerAngle** | Yes (**radians**, default τ) | Yes (**degrees?**) | **UNIT CONFLICT** |
| **coneOuterAngle** | Yes (**radians**, default τ) | Yes (**degrees?**) | **UNIT CONFLICT** |
| **coneOuterGain** | Yes (linear, default 0.0) | Yes | Same |
| **spatializationModel** | **Not supported** | Yes (equal power, HRTF, custom) | **MAJOR GAP** |

### Findings

- **F13 — ANGLE UNITS (POTENTIAL FUNDAMENTAL CONFLICT)**: audio_emitter explicitly uses **radians** (matching glTF conventions). audio_graph's cone angle units are not explicitly stated but the spec mentions "degrees" in the description. **The graph extension MUST use radians** to match the base spec and glTF conventions. This is a required change to audio_graph.

- **F14 — SPATIALIZATION MODEL (MAJOR GAP)**: audio_emitter has no `spatializationModel` property — it doesn't let you choose between equal-power panning, HRTF, or custom. This is critical for XR/spatial audio. Options:
  - (a) Add `spatializationModel` to audio_emitter's positional properties (base spec change)
  - (b) Add it via the graph extension's emitter extension properties
  - Since this is fundamental to how spatial audio is rendered, **(a) would be preferable but harder**.

- **F15 — CUSTOM DISTANCE MODEL**: audio_graph supports `custom` distance model; audio_emitter does not. This can be cleanly added via the graph extension.

- **F16 — STRUCTURAL NESTING**: audio_graph uses deeper nesting (`spatialProperties.attenuation`), audio_emitter uses flat `positional`. The graph extension should respect audio_emitter's existing structure.

---

## 6. Processing Nodes (audio_graph only — no equivalent in audio_emitter)

These exist only in audio_graph and have no counterpart in audio_emitter:

### 6.1 Basic Processing
| Node | I/O | Purpose |
|---|---|---|
| **gain** | 1→1 | Volume control with optional interpolation |
| **delay** | 1→1 | Time delay |
| **waveshaper** | 1→1 | Distortion/waveshaping |

### 6.2 Filters
| Node | I/O | Purpose |
|---|---|---|
| **lowpass** | 1→1 | Low-pass filter |
| **highpass** | 1→1 | High-pass filter |
| **bandpass** | 1→1 | Band-pass filter |
| **lowshelf** | 1→1 | Low-shelf EQ |
| **highshelf** | 1→1 | High-shelf EQ |
| **peaking** | 1→1 | Parametric EQ band |
| **notch** | 1→1 | Band-reject filter |
| **allpass** | 1→1 | Phase shifting |

### 6.3 Channel Operations
| Node | I/O | Purpose |
|---|---|---|
| **splitter** | 1→N | Split multichannel to mono channels |
| **channelmerger** | N→1 | Merge mono channels to multichannel |
| **channelmixer** | 1→1 | Up/down-mix channel count |
| **audiomixer** | N→1 | Sum multiple same-channel-count inputs |

### 6.4 Environmental / Sink
| Node | I/O | Purpose |
|---|---|---|
| **reverb** | 1→1 | Reverberation (room simulation) |
| **emitter** | 1→0 | Spatial/global output sink |

### Findings

- **F17 — PROCESSING NODE PLACEMENT**: All processing nodes are purely additive to audio_emitter. They form the core value of the graph extension. These should be defined entirely within the graph extension spec and don't require any audio_emitter changes.

- **F18 — REVERB: PROCESSING vs ENVIRONMENTAL**: The reverb node in audio_graph is defined as a 1→1 processing node, but its parameters (roomSize, minDistance, maxDistance, reflectivity) describe room/environment properties. Per the user's architecture vision, reverb belongs in the "environmental/listener" category, not the processing chain. **Decision needed**:
  - (a) Keep reverb as a processing node (insert anywhere in chain)
  - (b) Move reverb to environmental/listener extension
  - (c) Support both — simple reverb in chain + room-level reverb as environmental

- **F19 — BYPASS PROPERTY**: audio_graph defines `bypass` on processing nodes. This is purely a graph extension concern and requires no base spec changes.

---

## 7. Listener (audio_graph only)

| Aspect | audio_emitter | audio_graph |
|---|---|---|
| **Listener defined?** | **No** | Yes (TODO/incomplete) |
| **Attachment** | N/A | Camera node |
| **Properties** | N/A | Not yet specified |

### Findings

- **F20 — LISTENER (MAJOR GAP)**: audio_emitter has **no listener concept at all**. It relies on implementations to determine the listener position (presumably the active camera). audio_graph marks listener as TODO but states it should attach to a camera. This is a significant gap:
  - For audio_emitter: the implicit listener assumption works for simple cases
  - For graph-based spatial audio: explicit listener properties are needed (position, orientation, HRTF profiles, room characteristics)
  - **This belongs in the environmental/listener extension** as part of the second layer

---

## 8. Node/Scene Binding Comparison

| Aspect | audio_emitter | audio_graph |
|---|---|---|
| **Node binding** | `node.extensions.KHR_audio_emitter.emitter` (single index) | `node.extensions.KHR_audio_graph.emitter` or `.emitters` (scalar or array) |
| **Scene binding** | `scene.extensions.KHR_audio_emitter.emitters` (array of indices) | Not specified (emitters are graph-internal) |
| **Multiple emitters per node** | **No** (single emitter only) | Yes (array form) |

### Findings

- **F21 — SINGLE vs MULTIPLE EMITTERS PER NODE (FUNDAMENTAL)**: audio_emitter restricts nodes to a **single emitter**. audio_graph allows multiple. If the graph extension needs multiple emitters per node (e.g., a character with voice + footstep emitters processed through different chains), this is a **base spec constraint**.
  - The graph extension could work around this by using child nodes
  - Or this could be proposed as a base spec change: `emitter` → `emitters` (array)

- **F22 — SCENE-LEVEL EMITTERS**: audio_emitter supports scene-level global emitters. audio_graph doesn't address this. The graph extension should define how graph-processed audio can target scene-level emitters.

---

## 9. glTF Object Model / Animation

| Aspect | audio_emitter | audio_graph |
|---|---|---|
| **JSON Pointers** | Fully defined for emitter and source properties | Deferred (TODO) |
| **KHR_animation_pointer** | Supported | Not yet specified |

### Findings

- **F23 — ANIMATION SUPPORT**: audio_emitter has a complete set of animatable properties via JSON pointers. The graph extension will need to define its own JSON pointers for graph node parameters (e.g., filter frequency sweeps, gain automation). This is purely additive.

---

## 10. Fundamental Differences Requiring audio_emitter Changes

These are the items that **cannot** be cleanly solved purely as an extension layer and may require changes to the audio_emitter base spec:

| ID | Issue | Severity | Description |
|---|---|---|---|
| **F4** | Gain range | Medium | audio_graph [0,1] vs audio_emitter (0,+∞). **Resolution: graph adopts audio_emitter range.** No base change needed. |
| **F8** | Oscillator sources | **High** | audio_emitter sources always reference audio data. Oscillators have no data. Need new source type or extension mechanism. |
| **F13** | Angle units | Medium | audio_graph must adopt radians. **Resolution: graph changes to radians.** No base change needed. |
| **F14** | Spatialization model | **High** | No way to specify HRTF vs equal-power in audio_emitter. Could be extension property on emitter, but ideally is a base property. |
| **F20** | Listener | **High** | No listener in audio_emitter at all. Needed for proper spatial audio. Entirely new concept. |
| **F21** | Multiple emitters per node | Medium | audio_emitter limits to one. Workaround exists (child nodes). |
| **F2** | Encoding properties | Medium | Needed for processing. Can be extension property on audio data. |

### Summary: Minimum Required audio_emitter Changes (Proposed)

If we want to minimize base spec changes, here's what's **unavoidable** vs **avoidable**:

**Can be handled purely via extension:**
- All processing nodes (F17)
- Encoding properties on audio data (F2 — via extension property)
- Additional source properties like loopStart/loopEnd (F7)
- Listener (F20 — as separate extension)
- Custom distance models (F15)
- Bypass (F19)

**Strongly recommended base spec changes:**
- F14: Add optional `spatializationModel` to positional emitter properties (small, backward-compatible addition)

**Requires design decision:**
- F8: Oscillator sources — either extend the source model in the base spec, or design a graph-only oscillator node that bypasses the source→emitter model entirely
- F21: Multiple emitters per node — live with child-node workaround or change `emitter` to `emitters`

---

## 11. Proposed Extension Architecture

Based on the analysis, here's a two-layer architecture proposal:

### Layer 1: `KHR_audio_graph` (extends KHR_audio_emitter)
- Defines processing nodes (gain, delay, filters, waveshaper, channel ops)
- Defines graph structure (nodes, connections)
- Defines oscillator source node (graph-only, doesn't go through audio_emitter sources)
- Defines how graphs connect audio_emitter sources to audio_emitter emitters
- Extension on document-level `KHR_audio_emitter` object
- Extension on individual sources/emitters for graph binding

### Layer 2: `KHR_audio_environment` (extends KHR_audio_emitter, optionally KHR_audio_graph)
- Defines listener node (attached to camera/node)
- Defines room/environment properties
- Defines reverb as environmental effect
- Defines HRTF/spatialization model selection
- Defines custom distance model functions

---

## 12. Design Decisions (Resolved)

| # | Question | Decision | Impact |
|---|---|---|---|
| 1 | **Oscillator source model** | **Graph-only node.** Oscillators exist only in the graph extension as source nodes. No changes to audio_emitter's source model. | No base spec change |
| 2 | **Graph insertion point** | **(c) Document-level declaration** with explicit input/output bindings to audio_emitter sources and emitters. Graphs are declared at the top level, not on individual sources or emitters. | No base spec change |
| 3 | **Reverb placement** | **Environmental/listener layer.** Reverb is generalized as part of the listener/environment extension alongside HRTF mode selection. Flexible to support either reverb processing or arbitrary HRTF modes. | Reverb removed from graph processing nodes → moved to Layer 2 |
| 4 | **Spatialization model** | **Environmental/listener layer.** Same as #3 — HRTF, equal-power, and custom spatialization live in the environment extension, not as a base spec change. | No base spec change |
| 5 | **Multiple emitters per node** | **Base spec change: allow multiple emitters per node.** Change `emitter` (single) → `emitters` (array) on node bindings. | **Base spec change required** |
| 6 | **Unit convention** | **Adopt audio_emitter conventions.** Timing in seconds, angles in radians, matching glTF and Web Audio API. audio_graph's milliseconds convention is dropped. | Graph extension changes units |
| 7 | **Naming alignment** | **Full adoption of audio_emitter naming.** `playbackRate`, `autoplay`, `positional`, `shapeType`, gain range `(0, +∞)`, radians, seconds. | Graph extension adopts all conventions |

---

## 13. Confirmed Required audio_emitter Base Spec Changes

Only **one** base spec change is confirmed as required:

| Change | Current | Proposed | Rationale |
|---|---|---|---|
| **Multiple emitters per node** | `node.extensions.KHR_audio_emitter.emitter` (single index) | `node.extensions.KHR_audio_emitter.emitters` (array of indices) | Enables complex audio scenarios (e.g., character with voice + footsteps). Aligns with scene-level binding which already uses an array. |

All other gaps are addressed via the two extension layers without modifying the base spec.

---

## 14. Final Two-Layer Architecture

### Layer 1: `KHR_audio_graph` (extends KHR_audio_emitter)
- **Dependency**: Requires `KHR_audio_emitter`
- **Scope**: Processing pipeline between sources and emitters
- **Declares at document level**: graphs with nodes, connections, input/output bindings
- **Processing nodes**: gain, delay, filters (lowpass, highpass, bandpass, lowshelf, highshelf, peaking, notch, allpass), waveshaper, channel splitter/merger/mixer, audio mixer
- **Graph-only sources**: oscillator nodes (sine, square, triangle, saw, custom)
- **Encoding properties**: added as extension on audio_emitter's audio data objects
- **Conventions**: seconds, radians, gain (0, +∞), audio_emitter naming
- **Animation**: defines JSON pointers for graph node parameters (filter freq, gain automation, etc.)

### Layer 2: `KHR_audio_environment` (extends KHR_audio_emitter)
- **Dependency**: Requires `KHR_audio_emitter`, optionally `KHR_audio_graph`
- **Scope**: Listener, room acoustics, spatialization model
- **Listener**: explicit listener node attached to camera/node with properties
- **Spatialization model**: HRTF, equal-power, custom — selectable per listener or per scene
- **Reverb/Room**: generalized environmental audio processing (room simulation, IR-based reverb, HRTF profiles)
- **Custom distance models**: extend audio_emitter's linear/inverse/exponential
