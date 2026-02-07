# KHR_audio_emitter — Proposed Changes

## Status

Proposed amendment to Draft spec.

## Summary

This document describes the minimal required changes to the `KHR_audio_emitter` extension to support the `KHR_audio_graph` and `KHR_audio_environment` extension layers.

---

## Change 1: Multiple Emitters Per Node

### Current Spec (Node Binding)

Currently, a glTF node may reference only a **single** audio emitter:

```json
{
    "nodes": [
        {
            "name": "Duck",
            "translation": [1.0, 2.0, 3.0],
            "extensions": {
                "KHR_audio_emitter": {
                    "emitter": 0
                }
            }
        }
    ]
}
```

### Proposed Change

Change `emitter` (single integer index) to `emitters` (array of integer indices) on node-level bindings:

```json
{
    "nodes": [
        {
            "name": "Duck",
            "translation": [1.0, 2.0, 3.0],
            "extensions": {
                "KHR_audio_emitter": {
                    "emitters": [0]
                }
            }
        }
    ]
}
```

Multiple emitters example:

```json
{
    "nodes": [
        {
            "name": "Character",
            "translation": [0.0, 0.0, 0.0],
            "extensions": {
                "KHR_audio_emitter": {
                    "emitters": [0, 1, 2]
                }
            }
        }
    ]
}
```

### Rationale

- Enables complex audio scenarios: a character node with voice, footsteps, and armor sounds as separate emitters.
- Aligns node-level binding with scene-level binding, which already uses an array (`"emitters": [1, 3]`).
- Required by `KHR_audio_graph` where different emitters may be driven by different processing graphs.
- Avoids the need for artificial child nodes purely to host additional emitters.

### Schema Change

**Before (node extension):**
```json
{
    "type": "object",
    "properties": {
        "emitter": {
            "type": "integer",
            "minimum": 0,
            "description": "The index of the emitter referenced by this node."
        }
    },
    "required": ["emitter"]
}
```

**After (node extension):**
```json
{
    "type": "object",
    "properties": {
        "emitters": {
            "type": "array",
            "items": {
                "type": "integer",
                "minimum": 0
            },
            "minItems": 1,
            "uniqueItems": true,
            "description": "The indices of the emitters referenced by this node."
        }
    },
    "required": ["emitters"]
}
```

### glTF Object Model Update

Additional JSON pointer:

| JSON Pointer | Object Model Type |
|---|---|
| `/nodes/{}/extensions/KHR_audio_emitter/emitters` | `int[]` |
| `/nodes/{}/extensions/KHR_audio_emitter/emitters.length` | `int` (read-only) |

The existing pointer for scene-level emitters remains unchanged.

### Migration

- Existing files with `"emitter": N` should be migrated to `"emitters": [N]`.
- Implementations SHOULD accept the legacy `"emitter"` form during a transition period and treat it as equivalent to `"emitters": [N]`.

---

## No Other Base Spec Changes Required

All other gaps between `KHR_audio_emitter` and the audio graph/environment extensions are addressed entirely within the extension layers:

- Processing nodes → `KHR_audio_graph`
- Oscillator sources → `KHR_audio_graph` (graph-only)
- Encoding properties → `KHR_audio_graph` extension on audio data
- Listener → `KHR_audio_environment`
- Spatialization model → `KHR_audio_environment`
- Reverb/room properties → `KHR_audio_environment`
- Custom distance models → `KHR_audio_environment`
