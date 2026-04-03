# KHR_audio_graph

## Contributors

- Chintan Shah, Meta
- Alexey Medvedev, Meta

Copyright 2024 The Khronos Group Inc.
See [Appendix](#appendix-full-khronos-copyright-statement) for full Khronos Copyright Statement.

## Status

Draft

## Dependencies

Written against the glTF 2.0 spec.

- **Required**: `KHR_audio_emitter`
- **Optional**: `KHR_audio_environment` (for listener/room properties)

## Overview

This extension adds graph-based audio processing to `KHR_audio_emitter`. It allows inserting an arbitrary processing chain between audio sources and emitters, enabling mixing, filtering, effects, and procedural audio generation within glTF scenes.

Without this extension, `KHR_audio_emitter` provides a direct `source → emitter` path with no intermediate processing. `KHR_audio_graph` preserves that simple model as the default while enabling complex audio routing when needed.

### Motivation

Modern audio engines (Web Audio API, FMOD, Wwise, game engines) use graph-based routing to connect sources through processing chains to outputs. This extension brings that capability to glTF, enabling:

- Audio mixing and volume automation
- Filtering and equalization
- Waveshaping/distortion effects
- Channel routing (split, merge, up/down-mix)
- Procedural audio via oscillators
- Complex multi-source mixing to single or multiple emitters

### Design Principles

- Extends `KHR_audio_emitter` — does not replace it
- Graphs are declared at document level with explicit bindings to audio_emitter sources and emitters
- Follows audio_emitter conventions: seconds, radians, gain `(0, +∞)`, naming
- Processing nodes are the building blocks; connections form a DAG
- Oscillators are graph-only source nodes (do not use audio_emitter's source/audio model)

---

## Extension Declaration

Add `KHR_audio_graph` to `extensionsUsed`. The extension payload is declared under `extensions` at the document root.

```json
{
    "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_graph"],
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [...],
            "sources": [...],
            "emitters": [...]
        },
        "KHR_audio_graph": {
            "graphs": [...]
        }
    }
}
```

---

## Conventions

- **Time**: All time values are in **seconds** (matching glTF and Web Audio API conventions).
- **Angles**: All angles are in **radians** (matching glTF conventions).
- **Gain**: Linear, unitless, range `(0, +∞)` (matching `KHR_audio_emitter`).
- **Frequency**: Hertz (Hz).
- **Naming**: Follows `KHR_audio_emitter` conventions (`playbackRate`, `autoplay`, `positional`, `shapeType`).
- **Connections**: Edges reference node array indices with optional `input`/`output` port indices. Omitted port indices refer to the default (port 0).
- **Graph validity**: Graphs are directed acyclic graphs (DAGs). Cycles are not permitted.

---

## 1. Graphs

Graphs are the top-level construct of this extension. Each graph defines a set of nodes, connections between them, and bindings to `KHR_audio_emitter` sources and emitters.

### 1.1 Graph Object

| Property | Type | Description | Required |
|---|---|---|---|
| **name** | `string` | Human-readable name for debugging. | No |
| **nodes** | `node[]` | Array of audio processing nodes. | Yes |
| **connections** | `connection[]` | Array of edges between nodes. | Yes |
| **inputs** | `graphInput[]` | Bindings from `KHR_audio_emitter` sources to graph nodes. | No |
| **outputs** | `graphOutput[]` | Bindings from graph nodes to `KHR_audio_emitter` emitters. | No |

A valid graph must have at least one sink. A sink is either:
- A node whose output is listed in `outputs[]` (bound to an audio_emitter emitter), or
- A graph with no `outputs[]` where the final nodes are implicitly routed to the global audio destination.

### 1.2 Graph Inputs (Source Bindings)

Graph inputs bind `KHR_audio_emitter` sources to graph entry points.

| Property | Type | Description | Required |
|---|---|---|---|
| **source** | `integer` | Index into `KHR_audio_emitter.sources[]`. | Yes |
| **node** | `integer` | Index of the target node in this graph's `nodes[]`. | Yes |
| **input** | `integer` | Input port index on the target node (default: 0). | No |

```json
{
    "inputs": [
        { "source": 0, "node": 0 },
        { "source": 1, "node": 0, "input": 1 }
    ]
}
```

### 1.3 Graph Outputs (Emitter Bindings)

Graph outputs bind graph node outputs to `KHR_audio_emitter` emitters.

| Property | Type | Description | Required |
|---|---|---|---|
| **node** | `integer` | Index of the source node in this graph's `nodes[]`. | Yes |
| **output** | `integer` | Output port index on the source node (default: 0). | No |
| **emitter** | `integer` | Index into `KHR_audio_emitter.emitters[]`. | Yes |

```json
{
    "outputs": [
        { "node": 3, "emitter": 0 },
        { "node": 5, "emitter": 1 }
    ]
}
```

### 1.4 Connections

Connections are edges within the graph between nodes.

| Property | Type | Description | Required |
|---|---|---|---|
| **from** | `endpoint` | Upstream node endpoint. | Yes |
| **to** | `endpoint` | Downstream node endpoint. | Yes |

**Endpoint:**

| Property | Type | Description | Required |
|---|---|---|---|
| **node** | `integer` | Index into graph `nodes[]`. | Yes |
| **output** / **input** | `integer` | Port index (default: 0). | No |

```json
{
    "connections": [
        { "from": { "node": 0 }, "to": { "node": 1 } },
        { "from": { "node": 1, "output": 0 }, "to": { "node": 2, "input": 0 } }
    ]
}
```

### 1.5 Complete Graph Example

Two audio_emitter sources → gain mix → lowpass filter → audio_emitter emitter:

```json
{
    "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_graph"],
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [
                { "uri": "music.mp3" },
                { "uri": "ambience.mp3" }
            ],
            "sources": [
                { "audio": 0, "autoplay": true, "loop": true, "gain": 1.0 },
                { "audio": 1, "autoplay": true, "loop": true, "gain": 0.5 }
            ],
            "emitters": [
                { "type": "global", "gain": 1.0, "sources": [] }
            ]
        },
        "KHR_audio_graph": {
            "graphs": [
                {
                    "name": "music-mix",
                    "nodes": [
                        { "kind": "audiomixer", "params": {} },
                        { "kind": "gain", "params": { "gain": 0.8 } },
                        { "kind": "lowpass", "params": { "frequency": 8000 } }
                    ],
                    "connections": [
                        { "from": { "node": 0 }, "to": { "node": 1 } },
                        { "from": { "node": 1 }, "to": { "node": 2 } }
                    ],
                    "inputs": [
                        { "source": 0, "node": 0 },
                        { "source": 1, "node": 0 }
                    ],
                    "outputs": [
                        { "node": 2, "emitter": 0 }
                    ]
                }
            ]
        }
    }
}
```

---

## 2. Node Kinds

Each node in a graph is defined by a `kind` discriminator and a `params` object.

| Property | Type | Description | Required |
|---|---|---|---|
| **kind** | `string` | Node type identifier (see sections below). | Yes |
| **params** | `object` | Parameters specific to this node kind. | Yes |
| **label** | `string` | Human-readable label for debugging. | No |
| **bypass** | `boolean` | If `true`, node is skipped; audio passes through unprocessed (processing nodes only). Default: `false`. | No |

### Node Kind Summary

| Kind | Category | I/O | Section |
|---|---|---|---|
| `oscillator` | Source | 0→1 | 3.1 |
| `gain` | Processing | 1→1 | 4.1 |
| `delay` | Processing | 1→1 | 4.2 |
| `waveshaper` | Processing | 1→1 | 4.3 |
| `lowpass` | Filter | 1→1 | 5.1 |
| `highpass` | Filter | 1→1 | 5.2 |
| `bandpass` | Filter | 1→1 | 5.3 |
| `lowshelf` | Filter | 1→1 | 5.4 |
| `highshelf` | Filter | 1→1 | 5.5 |
| `peaking` | Filter | 1→1 | 5.6 |
| `notch` | Filter | 1→1 | 5.7 |
| `allpass` | Filter | 1→1 | 5.8 |
| `splitter` | Channel | 1→N | 6.1 |
| `channelmerger` | Channel | N→1 | 6.2 |
| `channelmixer` | Channel | 1→1 | 6.3 |
| `audiomixer` | Channel | N→1 | 6.4 |

---

## 3. Source Nodes

Source nodes generate audio. They have no inputs and one output.

### 3.1 Oscillator Node (0 input / 1 output)

Generates a periodic waveform. This is a graph-only source — it does not reference `KHR_audio_emitter` audio data or sources.

| Property | Type | Description | Required |
|---|---|---|---|
| **type** | `string` | Waveform type: `sine`, `square`, `triangle`, `sawtooth`, `custom`. | Yes |
| **frequency** | `number` | Frequency in Hz (0–20000). Default: 440. | No |
| **detune** | `number` | Detuning in cents. Default: 0. | No |
| **pulseWidth** | `number` | Pulse width for `square` waveform (0–1). 0.5 = pure square. Default: 0.5. | No |

**Web Audio Mapping**: Maps to `OscillatorNode`. If `type = "square"` and `pulseWidth != 0.5`, a `PeriodicWave` can approximate static PWM.

#### Example: Oscillator → gain → emitter

```json
{
    "name": "test-tone",
    "nodes": [
        { "kind": "oscillator", "params": { "type": "sine", "frequency": 440 } },
        { "kind": "gain", "params": { "gain": 0.3 } }
    ],
    "connections": [
        { "from": { "node": 0 }, "to": { "node": 1 } }
    ],
    "outputs": [
        { "node": 1, "emitter": 0 }
    ]
}
```

---

## 4. Processing Nodes

Processing nodes transform audio. Unless noted, they have 1 input and 1 output. All processing nodes support the `bypass` property.

### 4.1 Gain Node (1 input / 1 output)

Applies volume scaling.

| Property | Type | Description | Required |
|---|---|---|---|
| **gain** | `number` | Linear gain multiplier. Range: `(0, +∞)`. Default: 1.0. | No |
| **interpolation** | `string` | Transition curve when gain changes: `linear`, `custom`. Default: `linear`. | No |
| **duration** | `number` | Interpolation duration in seconds when gain changes. Default: 0. | No |

**Web Audio Mapping**: Maps to `GainNode`. `linear` interpolation → `linearRampToValueAtTime`. `custom` → `setTargetAtTime`.

### 4.2 Delay Node (1 input / 1 output)

Delays audio by a specified time.

| Property | Type | Description | Required |
|---|---|---|---|
| **delayTime** | `number` | Delay time in seconds. Default: 0. | No |
| **maxDelayTime** | `number` | Maximum delay time in seconds (for buffer allocation). Default: 1.0. | No |

**Web Audio Mapping**: Maps to `DelayNode`.

### 4.3 Waveshaper Node (1 input / 1 output)

Applies waveshaping distortion.

| Property | Type | Description | Required |
|---|---|---|---|
| **amount** | `number` | Distortion intensity in `[0, 1]`. Implementations may map to a shaping curve (e.g., tanh). | No |
| **oversample** | `string` | Oversampling factor: `none`, `2x`, `4x`. Default: `none`. | No |
| **curve** | `number[]` | Explicit shaping curve as normalized array in `[-1, 1]`. Overrides `amount` if provided. | No |

**Web Audio Mapping**: Maps to `WaveShaperNode`. `curve` maps directly to `WaveShaperNode.curve`. `oversample` maps to `WaveShaperNode.oversample`.

---

## 5. Filter Nodes

Filter nodes implement second-order (biquad) filters. All have 1 input and 1 output. All support `bypass`.

**Common Web Audio Mapping**: All filter nodes map to `BiquadFilterNode` with the corresponding `type` property. `qualityFactor` maps to `Q`.

### 5.1 Lowpass Filter (1 input / 1 output)

Passes frequencies below cutoff; attenuates above. 12dB/octave rolloff.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Cutoff frequency in Hz. | Yes |
| **qualityFactor** | `number` | Resonance at cutoff. Higher = more peaked. Default: 1.0. | No |

### 5.2 Highpass Filter (1 input / 1 output)

Passes frequencies above cutoff; attenuates below. 12dB/octave rolloff.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Cutoff frequency in Hz. | Yes |
| **qualityFactor** | `number` | Resonance at cutoff. Default: 1.0. | No |

### 5.3 Bandpass Filter (1 input / 1 output)

Passes a range of frequencies; attenuates outside.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Center frequency in Hz. | Yes |
| **qualityFactor** | `number` | Band width. Higher = narrower. Default: 1.0. | No |

### 5.4 Lowshelf Filter (1 input / 1 output)

Boosts or attenuates frequencies below a threshold.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Upper limit frequency in Hz for the shelf. | Yes |
| **gain** | `number` | Boost/attenuation in dB. Negative values attenuate. Default: 0. | No |

### 5.5 Highshelf Filter (1 input / 1 output)

Boosts or attenuates frequencies above a threshold.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Lower limit frequency in Hz for the shelf. | Yes |
| **gain** | `number` | Boost/attenuation in dB. Default: 0. | No |

### 5.6 Peaking Filter (1 input / 1 output)

Boosts or attenuates a range of frequencies (parametric EQ band).

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Center frequency in Hz. | Yes |
| **qualityFactor** | `number` | Band width. Higher = narrower. Default: 1.0. | No |
| **gain** | `number` | Boost/attenuation in dB. Default: 0. | No |

### 5.7 Notch Filter (1 input / 1 output)

Rejects (attenuates) a narrow band of frequencies (band-stop).

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Center frequency in Hz. | Yes |
| **qualityFactor** | `number` | Band width. Higher = narrower. Default: 1.0. | No |

### 5.8 Allpass Filter (1 input / 1 output)

Passes all frequencies but shifts phase relationships.

| Property | Type | Description | Required |
|---|---|---|---|
| **frequency** | `number` | Center frequency of phase transition in Hz. | Yes |
| **qualityFactor** | `number` | Phase transition sharpness. Higher = sharper. Default: 1.0. | No |

---

## 6. Channel Routing Nodes

### 6.1 Channel Splitter (1 input / N outputs)

Splits a multichannel input into individual mono channel outputs. The number of outputs equals the number of channels in the input.

| Property | Type | Description | Required |
|---|---|---|---|
| **channelInterpretation** | `string` | `speakers` or `discrete`. Default: `discrete`. | No |

**Web Audio Mapping**: Maps to `ChannelSplitterNode`.

Connection example: splitting stereo into L/R:
```json
{
    "from": { "node": 0, "output": 0 },
    "to": { "node": 2, "input": 0 }
},
{
    "from": { "node": 0, "output": 1 },
    "to": { "node": 3, "input": 0 }
}
```

### 6.2 Channel Merger (N inputs / 1 output)

Merges multiple mono inputs into a single multichannel output. Each input should be a single-channel stream.

| Property | Type | Description | Required |
|---|---|---|---|
| **channelInterpretation** | `string` | `speakers` or `discrete`. Default: `discrete`. | No |

**Web Audio Mapping**: Maps to `ChannelMergerNode`.

### 6.3 Channel Mixer (1 input / 1 output)

Up-mixes or down-mixes channel count. Follows [Web Audio mixing rules](https://webaudio.github.io/web-audio-api/#mixing-rules).

| Property | Type | Description | Required |
|---|---|---|---|
| **outputChannels** | `integer` | Desired output channel count. | Yes |
| **channelInterpretation** | `string` | `speakers` or `discrete`. Default: `speakers`. | No |

**Web Audio Mapping**: Realized via `channelCount`, `channelCountMode: "explicit"`, and `channelInterpretation` on a `GainNode`.

### 6.4 Audio Mixer (N inputs / 1 output)

Sums multiple audio inputs with the same channel count into a single output.

| Property | Type | Description | Required |
|---|---|---|---|
| **channelInterpretation** | `string` | `speakers` or `discrete`. Default: `speakers`. | No |

**Web Audio Mapping**: Realized by summing connections into a `GainNode` (Web Audio natively sums multiple connections to the same input).

---

## 7. Encoding Properties Extension

When `KHR_audio_graph` is present, audio data objects in `KHR_audio_emitter.audio[]` MAY include encoding metadata via an extension property. This metadata is needed for graph processing that depends on channel count, sample rate, or duration.

```json
{
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [
                {
                    "uri": "music.mp3",
                    "extensions": {
                        "KHR_audio_graph": {
                            "encoding": {
                                "sampleRate": 44100,
                                "channels": 2,
                                "bitsPerSample": 16,
                                "duration": 180.5,
                                "samples": 7960050
                            }
                        }
                    }
                }
            ]
        }
    }
}
```

### Encoding Properties

| Property | Type | Description | Required |
|---|---|---|---|
| **sampleRate** | `number` | Audio sampling rate in Hz. | Yes |
| **channels** | `integer` | Number of audio channels. | Yes |
| **bitsPerSample** | `integer` | Bits per audio sample. | No |
| **duration** | `number` | Audio duration in seconds. | No |
| **samples** | `integer` | Total number of samples. | No |

---

## 8. Source Property Extensions

When `KHR_audio_graph` is present, source objects in `KHR_audio_emitter.sources[]` MAY include extended playback properties:

```json
{
    "extensions": {
        "KHR_audio_emitter": {
            "sources": [
                {
                    "audio": 0,
                    "gain": 1.0,
                    "loop": true,
                    "autoplay": true,
                    "extensions": {
                        "KHR_audio_graph": {
                            "loopStart": 0.5,
                            "loopEnd": 120.0,
                            "offset": 0.0,
                            "when": 0.0,
                            "duration": 120.0,
                            "priority": 128,
                            "state": "playing",
                            "channelInterpretation": "speakers"
                        }
                    }
                }
            ]
        }
    }
}
```

### Extended Source Properties

| Property | Type | Description | Required |
|---|---|---|---|
| **loopStart** | `number` | Loop start position in seconds. | No |
| **loopEnd** | `number` | Loop end position in seconds. | No |
| **offset** | `number` | Playback start offset in seconds. Clamped to `[0, duration]`. | No |
| **when** | `number` | Scheduled start time in seconds. | No |
| **duration** | `number` | Playback duration in seconds. | No |
| **priority** | `integer` | Source priority (0 = highest, 256 = lowest). Default: 128. | No |
| **state** | `string` | Playback state: `playing`, `paused`, `stopped`. | No |
| **channelInterpretation** | `string` | `speakers` or `discrete`. | No |

---

## 9. Graph Rules (Normative)

1. Graphs are directed acyclic graphs (DAGs). **Cycles are not permitted.**
2. Each graph input binds exactly one `KHR_audio_emitter` source to one graph node input port.
3. Each graph output binds exactly one graph node output port to one `KHR_audio_emitter` emitter.
4. A single `KHR_audio_emitter` source may be bound to multiple graph inputs (fan-out from source).
5. A single `KHR_audio_emitter` emitter may be the target of multiple graph outputs (fan-in to emitter; signals are summed).
6. Multiple graphs may reference the same sources and/or emitters.
7. Fan-out is allowed: a node output may connect to multiple downstream node inputs.
8. Fan-in is allowed: a node input may receive connections from multiple upstream node outputs (signals are summed), subject to node arity constraints.
9. Splitter output count equals the channel count of its input.
10. Channel merger input count equals the channel count of its output.
11. Audio mixer inputs must have the same channel count.
12. If an `KHR_audio_emitter` emitter is bound as a graph output, its `sources` array in the base extension is ignored for that emitter — the graph provides the audio signal instead.

---

## 10. Bypass Behavior

Processing and filter nodes support an optional `bypass` boolean property.

- When `bypass` is `true`, the node is skipped. Audio passes through unprocessed from input to output.
- When `bypass` is `false` or not specified, the node processes audio normally.

Implementation options:
1. **Build-time routing**: Rewire connections around the bypassed node.
2. **Runtime passthrough**: Route dry signal directly to output, skipping processing.

Exact behavior is implementation-defined.

---

## 11. Web Audio API Mapping (Informative)

| Node Kind | Web Audio API | Notes |
|---|---|---|
| `oscillator` | `OscillatorNode` | PWM via `PeriodicWave` for `square` with non-0.5 pulseWidth |
| `gain` | `GainNode` | `interpolation: linear` → `linearRampToValueAtTime` |
| `delay` | `DelayNode` | |
| `waveshaper` | `WaveShaperNode` | `curve` maps directly; `amount` is implementation-defined mapping |
| `lowpass` | `BiquadFilterNode` (`type: "lowpass"`) | `qualityFactor` → `Q` |
| `highpass` | `BiquadFilterNode` (`type: "highpass"`) | `qualityFactor` → `Q` |
| `bandpass` | `BiquadFilterNode` (`type: "bandpass"`) | `qualityFactor` → `Q` |
| `lowshelf` | `BiquadFilterNode` (`type: "lowshelf"`) | `gain` in dB |
| `highshelf` | `BiquadFilterNode` (`type: "highshelf"`) | `gain` in dB |
| `peaking` | `BiquadFilterNode` (`type: "peaking"`) | `qualityFactor` → `Q`, `gain` in dB |
| `notch` | `BiquadFilterNode` (`type: "notch"`) | `qualityFactor` → `Q` |
| `allpass` | `BiquadFilterNode` (`type: "allpass"`) | `qualityFactor` → `Q` |
| `splitter` | `ChannelSplitterNode` | |
| `channelmerger` | `ChannelMergerNode` | |
| `channelmixer` | `GainNode` with explicit channel config | |
| `audiomixer` | Connection summing into `GainNode` | |

### Informative Mixing Behavior

When multiple graph outputs target the same `KHR_audio_emitter` emitter, an implementation can realize that emitter as a shared input bus and connect each upstream graph output to that bus. In Web Audio terms, this is naturally expressed by connecting multiple upstream nodes to the same `GainNode` or emitter input node, which sums those signals by default before emitter gain and spatialization are applied.

This informative mapping makes the normative fan-in rule practical for implementations and aligns multi-graph emitter routing with normal Web Audio connection semantics.

---

## 12. glTF Object Model

The following JSON pointers are defined for mutable properties, for use with `KHR_animation_pointer` and `KHR_interactivity`.

### Graph Node Parameters

| JSON Pointer | Object Model Type | Description |
|---|---|---|
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/params/gain` | `float` | Gain node volume |
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/params/delayTime` | `float` | Delay time (seconds) |
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/params/frequency` | `float` | Filter/oscillator frequency (Hz) |
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/params/qualityFactor` | `float` | Filter Q factor |
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/params/detune` | `float` | Oscillator detune (cents) |
| `/extensions/KHR_audio_graph/graphs/{}/nodes/{}/bypass` | `bool` | Node bypass state |

### Extended Source Properties

| JSON Pointer | Object Model Type |
|---|---|
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/loopStart` | `float` |
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/loopEnd` | `float` |
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/offset` | `float` |
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/when` | `float` |
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/priority` | `int` |
| `/extensions/KHR_audio_emitter/sources/{}/extensions/KHR_audio_graph/state` | `string` |

### Read-Only Properties

| JSON Pointer | Object Model Type |
|---|---|
| `/extensions/KHR_audio_graph/graphs.length` | `int` |
| `/extensions/KHR_audio_graph/graphs/{}/nodes.length` | `int` |
| `/extensions/KHR_audio_graph/graphs/{}/connections.length` | `int` |

---

## 13. JSON Schema Reference

Schema files (in `schema/` directory):

- `glTF.KHR_audio_graph.schema.json` — Document-level extension
- `graph.schema.json` — Graph object
- `node.schema.json` — Graph node (with kind discriminator)
- `connection.schema.json` — Connection edge
- `encoding.schema.json` — Encoding properties (audio data extension)
- `source.KHR_audio_graph.schema.json` — Extended source properties
- `oscillator.schema.json` — Oscillator parameters
- Node parameter schemas: `gain.schema.json`, `delay.schema.json`, `filter.schema.json`, `waveshaper.schema.json`, `splitter.schema.json`, `channelmerger.schema.json`, `channelmixer.schema.json`, `audiomixer.schema.json`

---

## Appendix: Full Khronos Copyright Statement

Copyright 2013-2017 The Khronos Group Inc.

Some parts of this Specification are purely informative and do not define requirements
necessary for compliance and so are outside the Scope of this Specification. These
parts of the Specification are marked as being non-normative, or identified as
**Implementation Notes**.

Where this Specification includes normative references to external documents, only the
specifically identified sections and functionality of those external documents are in
Scope. Requirements defined by external documents not created by Khronos may contain
contributions from non-members of Khronos not covered by the Khronos Intellectual
Property Rights Policy.

This specification is protected by copyright laws and contains material proprietary
to Khronos. Except as described by these terms, it or any components
may not be reproduced, republished, distributed, transmitted, displayed, broadcast
or otherwise exploited in any manner without the express prior written permission
of Khronos.
