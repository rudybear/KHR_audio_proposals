# glTF Audio — Analysis, Gaps, and KHR_audio_environment Groundwork

Working notes compiled 2026-08-02. Purpose: (a) verify the proposed audio extensions meet modern audio-development needs while remaining expressible as a standard, (b) provide structured feedback to extension authors, (c) prepare to draft KHR_audio_environment.

## Documents

| Doc | Content |
|---|---|
| [01-current-state.md](01-current-state.md) | Inventory of PRs #2137/#2572/#2421/#1400, issue #2561, proposal repo, reference impl; process blockers |
| [02-gap-analysis-audio-emitter.md](02-gap-analysis-audio-emitter.md) | KHR_audio_emitter vs Web Audio / X3D 4.0 / engines — 12 gaps E1–E12, prioritized |
| [03-gap-analysis-audio-graph.md](03-gap-analysis-audio-graph.md) | KHR_audio_graph vs Web Audio / X3D 4.0 — 11 gaps G1–G11, prioritized |
| [04-usd-comparison.md](04-usd-comparison.md) | Core USD, Omniverse, Apple RealityKit/PHASE deltas; strategic read |
| [05-KHR_audio_environment-proposals.md](05-KHR_audio_environment-proposals.md) | Tiered proposals P1–P14 + strawman JSON + open questions — input for drafting session |
| [06-structured-feedback.md](06-structured-feedback.md) | Paste-ready feedback items per PR + joint WG items |

## Executive summary

**The architecture is sound and has the right precedents.** The three-layer split (emitter → graph → environment) mirrors X3D 4.0's ISO-standardized declarativization of Web Audio almost level-for-level, which is the strongest possible answer to "can this be expressed as an audio standard?" — ISO already did it once with the same semantics. The graph's explicit inputs/outputs binding model is genuinely cleaner than X3D's destination-rooted encoding.

**The base layers are close but have a handful of holes that will surface in WG review:**
- *Emitter* (top 3): no channel rule for positional emitters; no playback-control/interactivity story; MP3-only means no gapless loops — the primary use case.
- *Graph* (top 3): `custom` oscillator enum with no PeriodicWave payload (spec bug); no compressor node (present in both Web Audio and X3D); playback properties (loop points, `when`, priority, `state`) parked in the graph extension where they don't belong, duplicating/conflicting with the base.
- *Process*: PR #2572 vendors a stale copy of the emitter spec (singular vs plural `emitters`), CLA unsigned for 3 committers, zero technical review so far.

**Against USD, glTF is ahead, not behind.** Core UsdMediaSpatialAudio is 7 attributes with no attenuation, cones, listener, or reverb; NVIDIA and Apple each built proprietary superstructure to compensate. The one core-USD idea worth stealing is timeline-anchored playback (startTime in stage time). Strategically: no standardized interactive-audio scene description ships in the USD ecosystem today, AOUSD has no audio WG — glTF can define the interchange target.

**KHR_audio_environment requirements converge from four independent sources.** X3D Level 3, MPEG-I, Omniverse, and Apple all added the same set: listener, HRTF selection, parametric+IR reverb with zones, per-source reverb sends, doppler, priority/voices, acoustic materials. Tier-1 proposal: listener lifecycle rules, I3DL2-anchored parametric reverb + presets + IR mode, per-emitter direct/reverb levels (not one global mix), and normative zone semantics (shape + blend + priority). Doppler, air absorption, cone low-pass, ambient/ambisonic beds, and voice limits in tier 2; acoustic materials, rooms/portals, and baked acoustics named as future work only.

## Messaging pillars ("solidify our messaging")

1. **Standards lineage, not invention**: every construct maps 1:1 to W3C Web Audio (reference implementation exists — AudioGraphJS) and to ISO/IEC 19775-1:2023 (X3D 4.0), while staying implementable on FMOD/Wwise/PHASE/Steam Audio.
2. **Layered adoption**: a viewer can ship emitter-only and be conformant; graph and environment add capability without breaking base files (default-listener and degrade rules make this literal).
3. **glTF leads transmission-format audio**: richer than core USD today; the natural export target for USD/DCC authoring; complements — not competes with — the Khronos↔AOUSD split of authoring vs delivery.
4. **Modern-needs coverage with a bounded core**: the gap analyses show the deltas to engine baseline are enumerated and tiered, not unknown; frontier acoustics (portals, baked wave data) are explicitly future-layered, keeping the core standardizable.

## Next session (drafting KHR_audio_environment)

Start from [05](05-KHR_audio_environment-proposals.md): settle the 6 open design questions, then draft README + JSON schemas following the KHR_audio_graph PR structure (document-level arrays, node/scene extension points, Object Model table, degrade rules). Target: Tier 1 (P1–P4) normative, P5–P8 included if the WG appetite is there, P9–P14 in Future Work.
