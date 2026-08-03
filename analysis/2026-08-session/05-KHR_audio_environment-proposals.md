# KHR_audio_environment — Proposals for the Draft

Starting point: the existing draft in [rudybear/KHR_audio_proposals](https://github.com/rudybear/KHR_audio_proposals) — `listeners[]` (spatializationModel equalpower/HRTF/custom, hrtf {audio, profile}, interauralDistance) + `environments[]` (reverb parametric|impulseResponse, mix, 9 parametric params, decayTime) + custom `distanceCurve` + per-emitter `spatializationModel` override. Required: KHR_audio_emitter; Optional: KHR_audio_graph.

The proposals below are tiered so the spec can ship small and grow. Convergent precedent for each item: **X3D** (ISO 19775-1:2023), **WA** (Web Audio), **I3DL2/EAX** (legacy standard param sets), **OV** (Omniverse), **AP** (Apple RealityKit/PHASE), **SA/PA** (Steam Audio / Project Acoustics), **MPEG-I** (23090-4).

## Tier 1 — must be in the first PR

### P1. Listener: keep, add normative lifecycle rules
Draft is right (node-bound, camera-typical, inherits transform, one active). Add:
- **Default rule**: no listener declared → listener is the active camera/viewer pose with equalpower model. (Closes emitter gap E8; without this, base-only and environment files behave identically by default — key compat story.)
- **Multiple listeners**: exactly one active; define selection (first in scene? explicit `activeListener` scene property — recommend the latter). X3D ListenerPointSource (multiple simultaneous virtual mics) is deliberately out of scope — say so (tier 3, P13).
- `gain` on the listener = master volume (WA AudioListener lacks this; engines/OV have master gain; cheap and useful).
- HRTF selection: keep `profile` enum + optional IR dataset reference, but define fallback normatively (unsupported HRTF → equalpower, MUST NOT fail load). Consider referencing **SOFA** (AES69) as the IR container instead of raw audio[] entries — it's the interchange standard Steam Audio already accepts; raw audio entries can't carry per-direction IR sets.

### P2. Reverb: anchor the parametric set to I3DL2 and add presets
The draft's parametric set (roomSize, reflectivity/High/Low, earlyReflections count/gain, diffusionGain, reflectionDelay, reverbDelay, decayTime) is Meta-lineage and reasonable, but it is a *bespoke* set — reviewers will ask "why these knobs?". Two changes:
- **Map to / adopt I3DL2** (room, roomHF, decayTime, decayHFRatio, reflections+delay, reverb+delay, diffusion, density, HFReference): it's the one *standardized* parametric reverb vocabulary, survives today in Unity/FMOD/OpenAL-EFX, and gives every engine a documented mapping. At minimum publish an informative I3DL2 ↔ KHR mapping table. reflectivityHigh/Low ≈ decayHFRatio-style frequency dependence — converge on one mechanism.
- **Named presets**: `preset` enum (generic, room, bathroom, cave, hangar, hall, arena, underwater, …) with normative parametric equivalents in an appendix. Presets = tiny files + instant cross-engine recognizability (EAX/Unity/PHASE all ship them). Explicit params override the preset.
- Keep the IR (`impulseResponse` + audio index) mode — WA/X3D Convolver parity. Specify IR channel handling (1/2/4 channels, WA true-stereo convention) and `normalize` semantics (note X3D default FALSE vs WA true — pick WA's true and say so).

### P3. Per-emitter sends, not one global mix
Replace/augment environment-wide `mix` with per-emitter `directLevel` + `reverbLevel` (linear gains; defaults 1.0/1.0), as an emitter-level extension property. Precedent: AP SpatialAudioComponent directLevel/reverbLevel, Wwise aux sends, PHASE direct/ER/LR pipeline flags, MPEG-I direct-to-reverberant control. The global `mix` stays as the environment's return level. This is the single highest-leverage authoring feature: distant-but-dry vs close-but-wet is unachievable with a global mix.

### P4. Zone semantics must be normative
Draft says node-bound environments scope "subtree or spatial region, implementation-defined" — that's a portability hole (same file, different sound). Define:
- Scene-level environment = default/fallback.
- Node-bound environment = **spatial zone**: explicit `shape` (box | sphere, in node-local space, dimensions given — reuse the KHR_audio_emitter shape-extensibility pattern) + `blendDistance` (crossfade band at the boundary) + integer `priority` for nesting (inner room wins over building wins over scene).
- The *listener's* position selects the active environment(s) (engine convention: reverb follows listener; state it).
- Emitters MAY override with an explicit `environment` index (Wwise-style forced send) — optional, cheap.

## Tier 2 — strongly recommended, second revision at latest

### P5. Doppler lives here
`doppler: {enabled: false, scale: 1.0, speedOfSound: 343.0}` on the environment (global), optional per-emitter `dopplerEnabled` override. Precedent: X3D Level 3, OV (scale + limit), MPEG-I. WA removed it, so the spec must state the model (pitch shift from radial velocity of emitter vs listener, velocities from node transforms per rendered frame; implementations MAY smooth). This closes emitter gap E7 in the right layer.

### P6. Listener-bus processing hook
One optional `graph` index on the listener (or environment): a KHR_audio_graph applied post-mix, pre-output (master compressor/EQ — the reason G1 wants a compressor node). Defines the graph spec's missing "master insert" point (gap G7) with one property. Requires KHR_audio_graph; degrade = skip.

### P7. Distance-dependent filtering (air absorption)
Keep the draft's `distanceCurve` (custom attenuation — engines' #1 complaint about the 3 fixed models is solved by this, good). Add the tonal counterpart: `airAbsorption: {enabled, cutoffAtMaxDistance}` (simple one-pole low-pass whose cutoff ramps with distance), or a second curve. Precedent: Wwise attenuation LPF curve, SA air absorption, PH. Volume-only distance is the most audible "not a real engine" tell.

### P8. Off-axis low-pass on cones
`coneOuterFilter` (0–1 LPF strength outside the inner cone) on the positional-emitter extension. Precedent: OV cone LPF, Wwise, AP directivity ("quieter *and darker* at rear"). Small, high realism-per-byte.

### P9. Ambient beds (reserve now, spec later)
A third emitter archetype: orientation-locked, position-free multichannel/ambisonic bed (AP Ambient, Meta audio skybox). Requires ambisonics metadata (order, AmbiX/FuMa ordering, SN3D/N3D normalization) and a multichannel-capable codec (→ emitter gap E2, Opus). Recommend: name it in future-work + reserve `"ambient"` in the emitter `type` enum discussion now, spec in a revision once the codec question lands.

### P10. Voice management
`maxVoices` (int) on the environment/scene + normative priority ordering (harmonize the 0–256 graph scale with X3D's [0,1] — pick once, see emitter gap E6). Behavior when exceeded: implementation-defined (virtualize or cull), but *ordering* is normative. Precedent: OV Concurrent Voices, X3D priority, every engine.

## Tier 3 — name in "Future work," do not spec yet

- **P11. Acoustic materials** — `KHR_materials_acoustic` on glTF materials: frequency-banded (3-band) absorption/scattering/transmission. Precedent: X3D AcousticProperties (absorption/diffuse/specular/refraction), SA materials, PH material presets, MPEG-I. This is its own extension with its own WG review; the environment spec should only say materials-based acoustics composes on top.
- **P12. Rooms & portals / occlusion / baked acoustics** — Wwise Rooms & Portals, SA pathing, PA's baked five-tuple (occlusion/obstruction/wetness/decay/portal direction). Real engines' frontier; premature for a delivery format today.
- **P13. Multiple listeners / virtual microphones** — X3D ListenerPointSource (broadcast/recording use cases).
- **P14. AR room estimation** — visionOS-style automatic acoustics is runtime behavior, not authorable; one informative sentence that implementations MAY substitute estimated acoustics in AR passthrough contexts.

## Strawman object model (Tier 1 + P5/P7)

```jsonc
"extensions": {
  "KHR_audio_environment": {
    "listeners": [{
      "name": "main",
      "gain": 1.0,
      "spatializationModel": "HRTF",        // equalpower | HRTF | custom
      "hrtf": { "profile": "generic" },      // optional; or SOFA/audio ref
      "interauralDistance": 0.17
    }],
    "environments": [{
      "name": "cathedral",
      "reverb": {
        "type": "parametric",               // parametric | impulseResponse
        "preset": "hall",                   // optional; params override
        "decayTime": 3.2, "decayHFRatio": 0.6,
        "reflectionsGain": 0.8, "reflectionsDelay": 0.02,
        "reverbGain": 1.0, "reverbDelay": 0.04,
        "diffusion": 1.0, "density": 1.0,
        "mix": 0.5                          // return level
      },
      "doppler": { "enabled": true, "scale": 1.0, "speedOfSound": 343.0 }
    }]
  }
}
// scene.extensions.KHR_audio_environment: { "environment": 0, "activeListener": 0 }
// node.extensions.KHR_audio_environment (listener): { "listener": 0 }
// node.extensions.KHR_audio_environment (zone):
//   { "environment": 1, "shape": {"type": "box", "size": [10,4,12]}, "blendDistance": 1.0, "priority": 1 }
// emitter-level (extension on KHR_audio_emitter emitter):
//   { "directLevel": 1.0, "reverbLevel": 0.7, "spatializationModel": "HRTF",
//     "distanceCurve": [...], "airAbsorption": {"enabled": true}, "coneOuterFilter": 0.5 }
```

Object Model pointers (animatable): listener gain/interauralDistance; environment decayTime, mix, gains; emitter directLevel/reverbLevel, coneOuterFilter; doppler scale.

## Open design questions to settle before drafting (next session)

1. I3DL2-aligned params vs Meta-lineage params — adopt, map, or hybrid? (P2)
2. Zone shape set: box+sphere only, or also convex mesh reference? (P4)
3. Where do per-emitter environment props live — extension-on-emitter (current draft pattern) or environment-side emitter list? (P3/P7/P8)
4. SOFA reference for HRTF IRs vs audio[] entries? (P1)
5. Does `mix` survive as return level once per-emitter sends exist? (P3)
6. Reserve `parametricModel` extensibility enum so SA/PA-style baked data can slot in later without breaking? (P12)
