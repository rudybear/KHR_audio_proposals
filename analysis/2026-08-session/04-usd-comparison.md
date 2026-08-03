# USD Comparison: glTF Audio Proposals vs the USD Ecosystem

Three distinct baselines matter: core USD (the standardized schema), NVIDIA Omniverse (what a real-time USD runtime added), and Apple RealityKit/visionOS + PHASE (what the largest USDZ consumer added). AOUSD has **no audio working group** and audio is absent from its core-spec roadmap.

## 1. Core USD: UsdMediaSpatialAudio — glTF is already ahead

The entire standardized schema is 7 attributes: `filePath`, `auralMode` (spatial/nonSpatial), `playbackMode` (5 modes), `startTime`, `endTime`, `mediaOffset`, `gain`. Positioning is purely transform-based. No cones, no distance/attenuation model, no listener, no reverb, no mixing, no priority. Only `gain` is time-samplable.

| Capability | USD core | KHR_audio_emitter |
|---|---|---|
| Positional vs ambient | auralMode | emitter type global/positional |
| Distance attenuation | — (runtime-defined) | 3 models + ref/max/rolloff |
| Directivity | — | cone (inner/outer/outerGain) |
| Playback region | startTime/endTime/mediaOffset | — base; offset/duration/loop pts via graph ext |
| Timeline-synced start | startTime in stage timeCodes (layer offsets compose) | — (autoplay only; `when` in graph ext) |
| Loop modes | 5 playbackModes incl. loopFromStage | loop bool |
| Animatable properties | gain only | gain, cone, distances via Object Model |

**Two genuine USD-core advantages to absorb:**
- **U1 — Timeline anchoring.** `startTime`/`endTime` are *stage timeCodes*, composing with layer offsets, so audio aligns with animation deterministically. glTF has `autoplay` (load-time) and the graph ext's `when` (seconds from context start — start of what, exactly? undefined). Recommendation: define `when`/playback scheduling relative to a named glTF animation's timeline, or at least define the clock normatively. This is the USD feature most missed in review.
- **U2 — `loopFromStage`-style behavior** (loop phase derived from scene time, so scrubbing stays in sync) — worth a note in the interactivity discussion.

## 2. Omniverse additions = the "real runtime" delta

What NVIDIA had to add to make USD audio usable is nearly a checklist of the gaps already identified in [02](02-gap-analysis-audio-emitter.md)/[05](05-KHR_audio_environment-proposals.md):

| Omniverse feature | glTF status |
|---|---|
| Attenuation range + type (inverse/linear/linearSquare) | ✅ emitter (richer: rolloff factor) |
| Cone angles/volumes | ✅ emitter |
| **Cone low-pass filter** (off-axis tone change) | ❌ — candidate for environment (P8) |
| **Doppler enable + scale + limit, speed of sound** | ❌ — proposed for environment (P5) |
| **Distance delay** (propagation latency) | ❌ — tier-3 future work |
| Interaural delay | ~ environment draft has interauralDistance |
| **Loop count (finite)** | ❌ — loop is boolean; cheap add |
| Media offset start/end | ✅ via graph ext (placement debate, G8) |
| Time scale (rate) | ✅ playbackRate |
| **Priority + Concurrent Voices (1–4096)** | ❌ base; partial in graph ext (E6) |
| **Listener prim + listener directivity cones** | ❌ — listener proposed in environment; directivity tier-3 |
| Speaker layouts to 9.1.6 | ❌ — out of scope (runtime), document |

## 3. Apple: RealityKit / Reality Composer Pro / PHASE

Apple ships **three source archetypes** (Spatial with HRTF + directivity beam; Ambient = fixed-direction multichannel bed, no reverb; Channel = head-locked, no spatialization). glTF mapping: positional emitter ≈ Spatial; global emitter ≈ Channel; **Ambient (audio-skybox bed) has no glTF equivalent** — candidate P9 in the environment proposals (ambisonic/multichannel bed with orientation but no position).

Apple's per-source **directLevel/reverbLevel split** (dry path vs send into environment reverb) is exactly the per-emitter reverb-send model recommended for KHR_audio_environment (P3) — stronger precedent than a single global wet/dry `mix`.

PHASE (and visionOS ReverbMeshResource) adds geometry-aware acoustics: occluder shapes with material presets (cardboard/brick/concrete/glass/wood), transmission, ray-traced reverb from scanned rooms, reverb presets, and Wwise-like sound-event graphs (sampler/random/switch/blend containers). These are the tier-3 items — real, but correctly out of scope for the first environment spec; they justify reserving acoustic-material and preset hooks now.

Caveat: RCP stores this as proprietary `RealityKitComponent` prims, *not* standardized USD — which is the point below.

## 4. Strategic read (messaging)

1. **Core USD standardizes less than KHR_audio_emitter already does.** Both NVIDIA and Apple had to invent proprietary superstructure. There is no standardized, portable interactive-audio scene description shipping today in either ecosystem.
2. **glTF can be the delivery-format leader here** — consistent with the Khronos↔AOUSD liaison framing (USD = authoring/composition, glTF = delivery/runtime). A finished KHR audio suite becomes the natural interchange target that USD authoring tools export *to*, and a template if AOUSD ever charters audio.
3. **The convergence set is striking**: Omniverse, Apple, X3D, and MPEG-I independently added the same missing pieces — listener, doppler, priority/voices, reverb sends, acoustic materials. That convergence *is* the requirements list for KHR_audio_environment, and citing it preempts "why this feature set?" review questions.

## 5. Items to fold back into the glTF feedback

- Finite loop count (`loopCount` int, 0 = infinite) — trivial, matches Omniverse + engine baseline. → emitter or playback ext.
- Define the playback clock for `when`/scheduling; consider animation-timeline anchoring (U1). → graph/emitter joint issue.
- Ambient bed source archetype (Apple Ambient / Meta skybox) with ambisonics metadata (order, AmbiX/FuMa, SN3D/N3D) — reserve, don't spec yet. → environment future-work section.
- Per-source direct/reverb levels rather than a single environment-wide mix. → environment (P3).
