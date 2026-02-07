# AudioGraphJS: Layered Extension Refactoring — Coding Agent Prompt

## Mission

Modify the AudioGraphJS project at `/Users/alexeymedvedev/Desktop/sources/audiograph2/AudioGraphJS` to support a new layered glTF audio extension architecture. The project currently implements `KHR_audio_graph` as a standalone extension. It must be refactored so that:

1. **KHR_audio_emitter** is the base layer (audio data, sources, emitters)
2. **KHR_audio_graph** extends audio_emitter with graph-based processing between sources and emitters
3. **KHR_audio_environment** extends audio_emitter with listener, reverb, and spatialization model

## Reference Documents

Read these files FIRST — they contain the full specifications:

- **Spec analysis & decisions**: `/Users/alexeymedvedev/Desktop/sources/audiograph2/analysis/spec_comparison_and_gap_analysis.md`
- **audio_emitter base spec changes**: `/Users/alexeymedvedev/Desktop/sources/audiograph2/specs/KHR_audio_emitter_changes.md`
- **audio_graph extension spec**: `/Users/alexeymedvedev/Desktop/sources/audiograph2/specs/KHR_audio_graph.md`
- **audio_environment extension spec**: `/Users/alexeymedvedev/Desktop/sources/audiograph2/specs/KHR_audio_environment.md`
- **Implementation plan**: `/Users/alexeymedvedev/.claude/plans/luminous-tumbling-panda.md`

## Key Design Decisions (Already Made)

1. **Oscillators** are graph-only source nodes (no audio_emitter source needed)
2. **Graphs** are declared at document level with explicit `inputs[]` (binding audio_emitter sources) and `outputs[]` (binding audio_emitter emitters)
3. **Reverb** belongs to the environment/listener layer, not the processing graph
4. **Spatialization model** (HRTF/equalpower/custom) belongs to the environment/listener layer
5. **Multiple emitters per node**: audio_emitter uses `emitters` (array) not `emitter` (single)
6. **Units**: seconds for time, radians for angles (matching audio_emitter/glTF conventions)
7. **Naming**: follows audio_emitter conventions — `playbackRate` (not playbackSpeed), `autoplay` (not autoPlay), `positional` (not spatial), `shapeType` (not shape), gain range `(0, +∞)`
8. **When a graph output binds to an emitter**, that emitter's `sources[]` array in audio_emitter is ignored — the graph provides the signal
9. **Backward compatibility**: existing runtime GraphSpec JSON files and old KHR format must still work

## Current Codebase Architecture

```
AudioGraphJS/
├── src/
│   ├── index.ts                    # Public API exports
│   ├── types.ts                    # Core types: NodeKind, GraphSpec, BuiltGraph, etc.
│   ├── runtime/
│   │   ├── buildGraph.ts           # Sync graph builder (GraphSpec → Web Audio nodes)
│   │   ├── buildGraphAsync.ts      # Async builder (loads URIs)
│   │   ├── emitters.ts             # applyEmitterInstances() — expands emitter buses to PannerNode+GainNode per instance
│   │   ├── lint.ts                 # lintGraph() — DAG, sink, arity validation
│   │   ├── preprocess.ts           # Build-time bypass rewiring
│   │   ├── wrapBypass.ts           # Dry/wet bypass wrapper
│   │   ├── bypassControl.ts       # Runtime bypass toggle
│   │   └── trace.ts               # Memory trace logger
│   ├── nodes/                      # 14 Web Audio node factories
│   │   ├── emitter.ts              # Creates GainNode bus (global) or PannerNode+GainNode (spatial)
│   │   ├── audioBufferSource.ts    # AudioBufferSourceNode with playback params
│   │   ├── oscillator.ts           # OscillatorNode with PWM support
│   │   ├── gain.ts                 # GainNode with interpolation
│   │   ├── biquadFilter.ts         # BiquadFilterNode (8 filter types)
│   │   ├── delay.ts                # DelayNode
│   │   ├── convolver.ts            # ConvolverNode (IR reverb)
│   │   ├── waveShaper.ts           # WaveShaperNode (distortion)
│   │   ├── panner.ts               # PannerNode (3D positioning)
│   │   ├── stereoPanner.ts         # StereoPannerNode
│   │   ├── channelSplitter.ts      # ChannelSplitterNode
│   │   ├── channelMerger.ts        # ChannelMergerNode
│   │   ├── channelMixer.ts         # GainNode with channel config
│   │   ├── audioMixer.ts           # GainNode summing junction
│   │   └── util.ts                 # applyChannelOptions helper
│   ├── serialization/
│   │   ├── gltf-emitters.ts        # extractEmitterBindings() — parses glTF node extensions
│   │   └── debugDump.ts            # Debug output
│   └── assets/
│       └── loadAudioBuffer.ts      # URI-based audio loading (browser + Node.js)
├── examples/
│   ├── run-graph.mjs               # Main CLI runner with mapKHRToRuntime() converter
│   ├── graphs/                     # 31 runtime GraphSpec JSON files
│   ├── graphs-khr/                 # 31 KHR container JSON files
│   ├── compare-khr-runtime.mjs     # Parity comparator
│   └── browser/                    # Browser demo
├── tools/spec-validate/            # Schema + lint validators
├── package.json                    # TypeScript, vitest, web-audio-engine, standardized-audio-context
└── tsconfig.json                   # ES2020, strict, declaration
```

### How the current pipeline works:

```
KHR_audio_graph JSON ──→ mapKHRToRuntime() ──→ runtime GraphSpec ──→ buildGraph() ──→ Web Audio API
                         (in run-graph.mjs)     (internal model)      (in runtime/)
```

Key points about the runtime model:
- `GraphSpec` = `{ nodes: GraphNodeSpec[], connections: GraphConnectionSpec[], outputs?: NodeId[] }`
- `GraphNodeSpec` = `{ id: string, kind: NodeKind, params?: {} }`
- NodeId is **string-based** in runtime, **index-based** in KHR
- Emitter nodes are GainNode "buses" in the runtime; `applyEmitterInstances()` later adds per-instance PannerNode + GainNode chains connected to `context.destination`
- Connections use `_outputs` map (from node) → `_inputs` map (to node) to support bypass wrappers

### Current KHR → Runtime mapping (in run-graph.mjs):
- `source` (with oscillator data) → `oscillator`
- `source` (with audioData ref) → `audio-buffer-source`
- `lowpass/highpass/bandpass/...` → `biquad-filter` with `type` param
- `reverb` → `convolver`
- `waveshaper` → `wave-shaper`
- `splitter` → `channel-splitter`
- `channelmerger` → `channel-merger`
- `channelmixer` → `channel-mixer`
- `audiomixer` → `audio-mixer`
- Times: ms → seconds (divide by 1000)
- `qualityFactor` → `Q`
- `playbackSpeed` → `playbackRate`
- Oscillator type: numeric → string

## What to Implement

### Phase 1: Type Definitions (`src/types.ts`)

Add interfaces for all three extension layers. Keep all existing types unchanged. Add:

- `AudioEmitterAudioData`, `AudioEmitterSource`, `AudioEmitterPositional`, `AudioEmitter`, `KHRAudioEmitterExtension`
- `GraphInput`, `GraphOutput`, `KHRGraphNodeSpec`, `KHRGraphConnection`, `KHRGraph`, `KHRAudioGraphExtension`
- `Listener`, `ReverbProperties`, `Environment`, `KHRAudioEnvironmentExtension`
- `GltfDocument` (unified glTF document type)

### Phase 2: Serialization Layer (`src/serialization/parse-layered.ts`) — NEW FILE

Create `parseLayeredExtensions(gltf)` that converts the layered glTF format to the existing runtime GraphSpec:

**Logic:**
1. Extract `KHR_audio_emitter` extension
2. If `KHR_audio_graph` is present:
   - For each graph, for each `inputs[]` entry: create an `audio-buffer-source` runtime node from the referenced audio_emitter source + audio data (resolve URI, set gain/playbackRate/loop/autoplay from source)
   - Map graph processing nodes to runtime nodes using the same kind mapping as existing code
   - Oscillator nodes are self-contained (no audio_emitter source binding)
   - For each `outputs[]` entry: create an `emitter` runtime node from the referenced audio_emitter emitter (map `type`→`emitterType`, `positional`→`spatialProperties` with nesting conversion)
   - Convert index-based connections to string-ID-based
   - Use `label` for node ID if present, else generate `"node_N"` IDs
3. If only `KHR_audio_emitter` (no graph): create simple source→emitter runtime specs
4. Extract emitter bindings from glTF nodes (look for `KHR_audio_emitter.emitters` array)
5. Extract listener from `KHR_audio_environment` on nodes
6. Extract environment from `KHR_audio_environment` on scenes

**Important mapping for emitter properties (audio_emitter → runtime):**
```
audio_emitter                    →  runtime emitter node params
──────────────────────────────────────────────────────────
type: "positional"               →  emitterType: "spatial"
type: "global"                   →  emitterType: "global"
gain                             →  gain
positional.distanceModel         →  spatialProperties.attenuation.distanceModel
positional.refDistance            →  spatialProperties.attenuation.refDistance
positional.maxDistance            →  spatialProperties.attenuation.maxDistance
positional.rolloffFactor          →  spatialProperties.attenuation.rolloffFactor
positional.shapeType             →  (used for cone setup)
positional.coneInnerAngle        →  spatialProperties.attenuation.coneInnerAngle
positional.coneOuterAngle        →  spatialProperties.attenuation.coneOuterAngle
positional.coneOuterGain         →  spatialProperties.attenuation.coneOuterGain
```

### Phase 3: Update `src/serialization/gltf-emitters.ts`

- Update `extractEmitterBindings()` to look for **both** `KHR_audio_emitter` and `KHR_audio_graph` extensions
- Support `emitters` (array) form per the spec change

### Phase 4: Runtime Additions

**`src/runtime/emitters.ts`** — Add `applyEmitterInstancesFromExtension()`:
- Reads emitter config from `AudioEmitter` objects (not from GraphSpec nodes)
- Maps audio_emitter's flat `positional` structure to PannerNode config
- Uses listener's `spatializationModel` as default panning model if available

**`src/runtime/environment.ts`** — NEW FILE:
- `applyEnvironment(context, environment, builtGraph, trace?)`
- For `impulseResponse` type: creates ConvolverNode with referenced audio data
- For `parametric` type: creates a simple reverb approximation
- Handles wet/dry mix via parallel gain paths
- Inserts between final graph output and destination

**`src/runtime/listener.ts`** — NEW FILE:
- `applyListener(context, listener, transform?, trace?)`
- Configures `context.listener` position/orientation from glTF node transform
- Returns the spatializationModel for use by emitter instance expansion

**`src/runtime/lint.ts`** — Add `lintLayeredGraph()`:
- Validates graph input/output bindings against audio_emitter arrays
- Ensures no `emitter` kind nodes in layered graphs (they're external)
- Existing DAG/arity checks still apply to processing nodes

### Phase 5: Update `src/index.ts`

Add exports: `parseLayeredExtensions`, `applyEnvironment`, `applyListener`

### Phase 6: Examples

Create `examples/graphs-layered/` with:

1. **`simple-emitter-only.json`** — KHR_audio_emitter only, no graph:
```json
{
  "extensionsUsed": ["KHR_audio_emitter"],
  "extensions": {
    "KHR_audio_emitter": {
      "audio": [{ "uri": "GENERATE_NOISE:1:0.5" }],
      "sources": [{ "audio": 0, "gain": 0.5, "autoplay": true }],
      "emitters": [{ "type": "global", "gain": 1.0, "sources": [0] }]
    }
  },
  "scenes": [{ "extensions": { "KHR_audio_emitter": { "emitters": [0] } } }]
}
```

2. **`graph-processing.json`** — audio_emitter + audio_graph with processing:
```json
{
  "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_graph"],
  "extensions": {
    "KHR_audio_emitter": {
      "audio": [{ "uri": "GENERATE_NOISE:1:0.6" }],
      "sources": [{ "audio": 0, "gain": 1.0, "autoplay": true }],
      "emitters": [{ "type": "global", "gain": 0.8, "sources": [] }]
    },
    "KHR_audio_graph": {
      "graphs": [{
        "name": "filtered",
        "nodes": [
          { "kind": "gain", "params": { "gain": 0.5 }, "label": "vol" },
          { "kind": "lowpass", "params": { "frequency": 2000, "qualityFactor": 1.0 }, "label": "lpf" }
        ],
        "connections": [{ "from": { "node": 0 }, "to": { "node": 1 } }],
        "inputs": [{ "source": 0, "node": 0 }],
        "outputs": [{ "node": 1, "emitter": 0 }]
      }]
    }
  }
}
```

3. **`oscillator-graph.json`** — Graph-only oscillator source:
```json
{
  "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_graph"],
  "extensions": {
    "KHR_audio_emitter": {
      "audio": [],
      "sources": [],
      "emitters": [{ "type": "global", "gain": 1.0, "sources": [] }]
    },
    "KHR_audio_graph": {
      "graphs": [{
        "name": "synth",
        "nodes": [
          { "kind": "oscillator", "params": { "type": "sine", "frequency": 440 }, "label": "osc" },
          { "kind": "gain", "params": { "gain": 0.3 }, "label": "vol" }
        ],
        "connections": [{ "from": { "node": 0 }, "to": { "node": 1 } }],
        "outputs": [{ "node": 1, "emitter": 0 }]
      }]
    }
  }
}
```

4. **`spatial-with-environment.json`** — Full stack with environment + listener

### Phase 7: Update `examples/run-graph.mjs`

Add layered format detection. Detection order:
1. Runtime GraphSpec (has `nodes[]` + `connections[]` at root)
2. Layered (has `extensions.KHR_audio_emitter`)
3. Legacy KHR (has `extensions.KHR_audio_graph` without `KHR_audio_emitter`)

Import and call `parseLayeredExtensions()` for layered format.

### Phase 8: Validator

Create `tools/spec-validate/validate-layered.mjs` for the new format.

## Constraints

- **Do NOT change** existing node implementations in `src/nodes/` unless absolutely necessary
- **Do NOT change** `buildGraph.ts` or `buildGraphAsync.ts` — the runtime builder stays the same
- **Do NOT break** existing runtime JSON or old KHR examples
- **All new code must be TypeScript** (strict mode, ES2020)
- **New spec uses seconds** (not milliseconds) — no ms→s conversion needed for layered format
- **New spec uses string oscillator types** (not numeric) — no type mapping needed

## Build & Test

```bash
cd /Users/alexeymedvedev/Desktop/sources/audiograph2/AudioGraphJS
npm install
npm run build        # TypeScript compile
npm test             # Unit tests
npm run example:compare-khr  # Existing parity check (must still pass)
node examples/run-graph.mjs examples/graphs-layered/oscillator-graph.json  # Test new format
```

## Implementation Order

1. `src/types.ts` — Add all new interfaces
2. `src/serialization/parse-layered.ts` — Core parsing logic
3. `src/serialization/gltf-emitters.ts` — Update for dual extension support
4. `src/runtime/listener.ts` — Listener support
5. `src/runtime/environment.ts` — Environment/reverb support
6. `src/runtime/emitters.ts` — Add extension-based variant
7. `src/runtime/lint.ts` — Add layered validation
8. `src/index.ts` — Add exports
9. `examples/graphs-layered/*.json` — Example files
10. `examples/run-graph.mjs` — Layered format detection
11. `tools/spec-validate/validate-layered.mjs` — Validator
12. Tests — Update/create test assets for all three extensions
13. Build + test + verify

## Phase 9: Test Assets

**IMPORTANT**: All extensions must have test coverage. Create or update test files to cover:

### New test files (vitest):

Create tests in a `tests/` or `src/__tests__/` directory (follow existing project conventions — check if tests already exist and where):

1. **`parse-layered.test.ts`** — Test `parseLayeredExtensions()`:
   - Test audio_emitter-only input (no graph) produces correct runtime spec
   - Test audio_emitter + audio_graph produces correct runtime spec with source→processing→emitter chain
   - Test graph with oscillator-only source (no audio_emitter source binding)
   - Test multiple graphs
   - Test multiple inputs/outputs per graph
   - Test that emitter's `sources[]` is ignored when graph output binds to it
   - Test encoding properties extension on audio data

2. **`gltf-emitters.test.ts`** — Test updated `extractEmitterBindings()`:
   - Test `KHR_audio_emitter` extension (new format with `emitters` array)
   - Test `KHR_audio_graph` extension (legacy format)
   - Test both present (KHR_audio_emitter takes precedence)
   - Test transform extraction (translation, rotation, scale)

3. **`listener.test.ts`** — Test `applyListener()`:
   - Test HRTF model
   - Test equalpower model
   - Test custom model with profile
   - Test position/orientation from transform

4. **`environment.test.ts`** — Test `applyEnvironment()`:
   - Test parametric reverb creates processing chain
   - Test impulse response reverb creates ConvolverNode
   - Test wet/dry mix

5. **`lint-layered.test.ts`** — Test `lintLayeredGraph()`:
   - Test valid graph passes
   - Test invalid source index errors
   - Test invalid emitter index errors
   - Test emitter kind nodes rejected in layered graph
   - Test DAG validation still works

6. **`emitters-extension.test.ts`** — Test `applyEmitterInstancesFromExtension()`:
   - Test global emitter creates gain → destination
   - Test positional emitter creates panner + gain → destination
   - Test spatial properties mapping (distance model, cone angles in radians)
   - Test listener spatializationModel applied as default panning model

### Update existing tests:

- Ensure existing tests in the project still pass unchanged
- If `compare-khr-runtime.mjs` acts as an integration test, ensure it still passes

### Integration test examples:

The example JSON files in `examples/graphs-layered/` serve as integration tests. Each should be runnable with `run-graph.mjs` and produce valid WAV + trace output. Consider adding expected trace baselines for these in `examples/expected/`.
