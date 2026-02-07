# KHR_audio_environment

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
- **Optional**: `KHR_audio_graph` (for graph-integrated environmental processing)

## Overview

This extension adds listener, spatialization model, and environmental audio properties to `KHR_audio_emitter`. It addresses aspects of spatial audio that `KHR_audio_emitter` does not cover: where the listener is, how spatialization is computed, and how the acoustic environment affects sound.

Without this extension, `KHR_audio_emitter` relies on implementations to implicitly determine the listener (typically the active camera) and spatialization algorithm. `KHR_audio_environment` makes these explicit and adds room acoustics.

### Motivation

Spatial audio quality depends on three factors beyond source/emitter placement:

1. **Listener**: Where is the listener? What are its properties? (head size, HRTF profile)
2. **Spatialization model**: How is 3D positioning computed? (HRTF, equal-power, custom)
3. **Environment**: What does the room sound like? (reverb, reflections, absorption)

Modern XR and game engines provide all three. This extension brings them to glTF.

### Design Principles

- Extends `KHR_audio_emitter` — does not replace it
- Follows audio_emitter conventions: seconds, radians, gain `(0, +∞)`
- Listener is explicit, not implicit
- Spatialization model is selectable
- Environmental properties are flexible: supports both simple parametric reverb and IR-based processing
- Can work with or without `KHR_audio_graph`

---

## Extension Declaration

```json
{
    "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_environment"],
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [...],
            "sources": [...],
            "emitters": [...]
        },
        "KHR_audio_environment": {
            "listeners": [...],
            "environments": [...]
        }
    }
}
```

---

## 1. Listener

The listener represents the point in the scene from which spatial audio is heard. Without this extension, the listener is implicitly the active camera. This extension makes the listener explicit.

### 1.1 Listener Object

Listeners are defined at the document level in the `listeners[]` array.

| Property | Type | Description | Required |
|---|---|---|---|
| **name** | `string` | Human-readable name. | No |
| **spatializationModel** | `string` | Spatialization algorithm. See 1.3. | No |
| **hrtf** | `object` | Custom HRTF configuration. See 1.4. | No |
| **interauralDistance** | `number` | Distance between ears in meters. Default: 0.17. | No |
| **extensions** | `object` | Extension-specific objects. | No |
| **extras** | `any` | Application-specific data. | No |

### 1.2 Listener Binding

Listeners are bound to glTF nodes (typically cameras) via a node extension:

```json
{
    "nodes": [
        {
            "name": "MainCamera",
            "camera": 0,
            "extensions": {
                "KHR_audio_environment": {
                    "listener": 0
                }
            }
        }
    ]
}
```

A scene may have **at most one active listener** at runtime. If multiple nodes reference listeners, the implementation should select the listener attached to the active camera, or the first listener encountered.

The listener inherits the **position and orientation** from the glTF node it is attached to. These properties are not duplicated on the listener object.

| Property | Type | Description | Required |
|---|---|---|---|
| **listener** | `integer` | Index into `KHR_audio_environment.listeners[]`. | Yes |

### 1.3 Spatialization Model

The `spatializationModel` property defines how positional audio emitters are spatialized relative to the listener.

| Value | Description |
|---|---|
| `equalpower` | Equal-power panning. Simple and efficient. Produces a stereo image. **Default.** |
| `HRTF` | Head-Related Transfer Function. Renders binaural audio using measured or modeled impulse responses from human subjects. Higher quality spatial positioning than equal-power. |
| `custom` | User-defined spatialization. The algorithm is defined by the `hrtf` property or via application-specific extensions. |

If not specified, the default is `equalpower`.

```json
{
    "listeners": [
        {
            "name": "Player Listener",
            "spatializationModel": "HRTF"
        }
    ]
}
```

### 1.4 Custom HRTF Configuration

When `spatializationModel` is `HRTF` or `custom`, optional HRTF configuration may be provided.

| Property | Type | Description | Required |
|---|---|---|---|
| **audio** | `integer` | Index into `KHR_audio_emitter.audio[]` referencing an HRTF impulse response dataset. | No |
| **profile** | `string` | Named HRTF profile hint: `generic`, `small`, `medium`, `large`. Implementations may use this to select a built-in HRTF dataset. Default: `generic`. | No |

```json
{
    "listeners": [
        {
            "name": "Custom HRTF Listener",
            "spatializationModel": "HRTF",
            "hrtf": {
                "profile": "medium"
            }
        }
    ]
}
```

For IR-based custom HRTF:

```json
{
    "listeners": [
        {
            "name": "IR HRTF Listener",
            "spatializationModel": "custom",
            "hrtf": {
                "audio": 5
            }
        }
    ]
}
```

---

## 2. Environment

Environments define the acoustic characteristics of a space — how sound reflects, reverberates, and is absorbed. An environment applies to all audio emitters within its scope.

### 2.1 Environment Object

Environments are defined at the document level in the `environments[]` array.

| Property | Type | Description | Required |
|---|---|---|---|
| **name** | `string` | Human-readable name. | No |
| **reverb** | `object` | Reverb properties. See 2.3. | No |
| **extensions** | `object` | Extension-specific objects. | No |
| **extras** | `any` | Application-specific data. | No |

### 2.2 Environment Binding

Environments are bound to **scenes** (global environment) or to **nodes** (localized environment zones):

#### Scene-level (global environment):

```json
{
    "scenes": [
        {
            "name": "Cathedral",
            "extensions": {
                "KHR_audio_environment": {
                    "environment": 0
                }
            }
        }
    ]
}
```

#### Node-level (localized zone):

```json
{
    "nodes": [
        {
            "name": "TunnelZone",
            "extensions": {
                "KHR_audio_environment": {
                    "environment": 1
                }
            }
        }
    ]
}
```

When a node-level environment is defined, it applies to emitters within that node's subtree (or within a defined spatial region, implementation-defined). Scene-level environments provide the fallback/default.

### 2.3 Reverb Properties

Reverb simulates the acoustic characteristics of an enclosed space.

| Property | Type | Description | Required |
|---|---|---|---|
| **type** | `string` | Reverb type: `parametric`, `impulseResponse`. Default: `parametric`. | No |
| **mix** | `number` | Wet/dry blend. 0 = fully dry, 1 = fully wet. Default: 0.5. | No |
| **roomSize** | `number` | Approximate room size, wall-to-wall distance in meters. Default: 10.0. | No |
| **reflectivity** | `number` | How much audio reflects at each surface bounce. Range `[0, 1]`. Higher = harder surfaces. Default: 0.5. | No |
| **reflectivityHigh** | `number` | Reflectivity for high frequencies. Range `[0, 1]`. Default: same as `reflectivity`. | No |
| **reflectivityLow** | `number` | Reflectivity for low frequencies. Range `[0, 1]`. Default: same as `reflectivity`. | No |
| **earlyReflections** | `integer` | Number of early reflections (0–32). Default: 8. | No |
| **earlyReflectionsGain** | `number` | Gain for early reflections. Range `(0, +∞)`. Default: 1.0. | No |
| **diffusionGain** | `number` | Gain for late diffuse reverb. Range `(0, +∞)`. Default: 1.0. | No |
| **reflectionDelay** | `number` | Initial reflection delay in seconds. Default: 0.02. | No |
| **reverbDelay** | `number` | Late reverb onset delay relative to initial reflection, in seconds. Default: 0.04. | No |
| **decayTime** | `number` | Time for reverb to decay by 60dB (RT60) in seconds. Default: 1.5. | No |
| **audio** | `integer` | Index into `KHR_audio_emitter.audio[]` for impulse response data. Required when `type` is `impulseResponse`. | No |

#### Parametric Reverb Example

```json
{
    "environments": [
        {
            "name": "Large Cathedral",
            "reverb": {
                "type": "parametric",
                "mix": 0.6,
                "roomSize": 40.0,
                "reflectivity": 0.8,
                "reflectivityHigh": 0.6,
                "reflectivityLow": 0.9,
                "earlyReflections": 16,
                "earlyReflectionsGain": 1.2,
                "diffusionGain": 0.8,
                "reflectionDelay": 0.03,
                "reverbDelay": 0.06,
                "decayTime": 4.0
            }
        }
    ]
}
```

#### Impulse Response Reverb Example

```json
{
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [
                { "uri": "music.mp3" },
                { "uri": "cathedral_ir.wav" }
            ],
            "sources": [...],
            "emitters": [...]
        },
        "KHR_audio_environment": {
            "environments": [
                {
                    "name": "Cathedral IR",
                    "reverb": {
                        "type": "impulseResponse",
                        "mix": 0.5,
                        "audio": 1
                    }
                }
            ]
        }
    }
}
```

**Web Audio Mapping**: Parametric reverb is implementation-defined (may use algorithmic reverb or generate an IR). Impulse response reverb maps to `ConvolverNode` with the referenced audio data as the impulse response buffer. Wet/dry mix is realized by parallel dry and wet signal paths with respective gains.

---

## 3. Custom Distance Models

`KHR_audio_emitter` defines three distance models: `linear`, `inverse`, `exponential`. This extension adds the ability to define custom distance attenuation.

Custom distance models are declared as an extension on `KHR_audio_emitter` emitter positional properties:

```json
{
    "extensions": {
        "KHR_audio_emitter": {
            "emitters": [
                {
                    "type": "positional",
                    "gain": 1.0,
                    "sources": [0],
                    "positional": {
                        "distanceModel": "custom",
                        "refDistance": 1.0,
                        "maxDistance": 50.0,
                        "rolloffFactor": 1.0,
                        "extensions": {
                            "KHR_audio_environment": {
                                "distanceCurve": [1.0, 0.8, 0.5, 0.3, 0.1, 0.0]
                            }
                        }
                    }
                }
            ]
        }
    }
}
```

### Custom Distance Properties

| Property | Type | Description | Required |
|---|---|---|---|
| **distanceCurve** | `number[]` | Array of gain values sampled uniformly from `refDistance` to `maxDistance`. Linear interpolation between samples. Values in range `[0, 1]`. | Yes |

When `distanceModel` is `custom` on the base emitter, the `distanceCurve` in this extension defines the attenuation. The curve is sampled uniformly:
- Index 0 corresponds to `refDistance`
- Last index corresponds to `maxDistance`
- Intermediate values are linearly interpolated

---

## 4. Emitter Spatialization Override

While the listener defines the default spatialization model, individual emitters may override it. This is useful when some emitters need HRTF while others use equal-power in the same scene.

```json
{
    "extensions": {
        "KHR_audio_emitter": {
            "emitters": [
                {
                    "type": "positional",
                    "gain": 1.0,
                    "sources": [0],
                    "positional": {
                        "distanceModel": "inverse",
                        "refDistance": 1.0,
                        "maxDistance": 100.0,
                        "rolloffFactor": 1.0,
                        "extensions": {
                            "KHR_audio_environment": {
                                "spatializationModel": "HRTF"
                            }
                        }
                    }
                }
            ]
        }
    }
}
```

| Property | Type | Description | Required |
|---|---|---|---|
| **spatializationModel** | `string` | Override: `equalpower`, `HRTF`, `custom`. If not set, uses the listener's model. | No |

---

## 5. Complete Example

A scene with a listener, environment, and spatially-processed audio:

```json
{
    "asset": { "version": "2.0" },
    "extensionsUsed": ["KHR_audio_emitter", "KHR_audio_environment"],
    "extensions": {
        "KHR_audio_emitter": {
            "audio": [
                { "uri": "footsteps.mp3" },
                { "uri": "cathedral_ir.wav" }
            ],
            "sources": [
                { "audio": 0, "autoplay": true, "loop": true, "gain": 1.0 }
            ],
            "emitters": [
                {
                    "type": "positional",
                    "gain": 0.8,
                    "sources": [0],
                    "positional": {
                        "shapeType": "omnidirectional",
                        "distanceModel": "inverse",
                        "refDistance": 1.0,
                        "maxDistance": 50.0,
                        "rolloffFactor": 1.0
                    }
                }
            ]
        },
        "KHR_audio_environment": {
            "listeners": [
                {
                    "name": "Player",
                    "spatializationModel": "HRTF",
                    "interauralDistance": 0.17
                }
            ],
            "environments": [
                {
                    "name": "Cathedral",
                    "reverb": {
                        "type": "parametric",
                        "mix": 0.4,
                        "roomSize": 30.0,
                        "reflectivity": 0.75,
                        "decayTime": 3.5,
                        "earlyReflections": 12
                    }
                }
            ]
        }
    },
    "scenes": [
        {
            "name": "Main",
            "nodes": [0, 1],
            "extensions": {
                "KHR_audio_environment": {
                    "environment": 0
                }
            }
        }
    ],
    "nodes": [
        {
            "name": "PlayerCamera",
            "camera": 0,
            "extensions": {
                "KHR_audio_environment": {
                    "listener": 0
                }
            }
        },
        {
            "name": "Character",
            "translation": [5.0, 0.0, 3.0],
            "extensions": {
                "KHR_audio_emitter": {
                    "emitters": [0]
                }
            }
        }
    ]
}
```

---

## 6. glTF Object Model

### Mutable Properties

| JSON Pointer | Object Model Type | Description |
|---|---|---|
| `/extensions/KHR_audio_environment/listeners/{}/spatializationModel` | `string` | Listener spatialization model |
| `/extensions/KHR_audio_environment/listeners/{}/interauralDistance` | `float` | Inter-aural distance |
| `/extensions/KHR_audio_environment/environments/{}/reverb/mix` | `float` | Reverb wet/dry mix |
| `/extensions/KHR_audio_environment/environments/{}/reverb/roomSize` | `float` | Room size |
| `/extensions/KHR_audio_environment/environments/{}/reverb/reflectivity` | `float` | Surface reflectivity |
| `/extensions/KHR_audio_environment/environments/{}/reverb/decayTime` | `float` | RT60 decay time |
| `/extensions/KHR_audio_environment/environments/{}/reverb/earlyReflectionsGain` | `float` | Early reflections gain |
| `/extensions/KHR_audio_environment/environments/{}/reverb/diffusionGain` | `float` | Late reverb gain |

### Read-Only Properties

| JSON Pointer | Object Model Type |
|---|---|
| `/extensions/KHR_audio_environment/listeners.length` | `int` |
| `/extensions/KHR_audio_environment/environments.length` | `int` |

---

## 7. JSON Schema Reference

Schema files (in `schema/` directory):

- `glTF.KHR_audio_environment.schema.json` — Document-level extension
- `listener.schema.json` — Listener object
- `hrtf.schema.json` — HRTF configuration
- `environment.schema.json` — Environment object
- `reverb.schema.json` — Reverb properties
- `distanceCurve.schema.json` — Custom distance attenuation curve
- `node.KHR_audio_environment.schema.json` — Node-level extension (listener binding)
- `scene.KHR_audio_environment.schema.json` — Scene-level extension (environment binding)
- `positional.KHR_audio_environment.schema.json` — Emitter positional extension (spatialization override, custom distance)

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
