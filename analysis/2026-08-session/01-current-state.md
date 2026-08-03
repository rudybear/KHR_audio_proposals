# Current State of glTF Audio Proposals

*Compiled 2026-08-02 from PR/issue snapshots. Companion docs: [02-gap-analysis-audio-emitter.md](02-gap-analysis-audio-emitter.md), [03-gap-analysis-audio-graph.md](03-gap-analysis-audio-graph.md), [04-usd-comparison.md](04-usd-comparison.md), [05-KHR_audio_environment-proposals.md](05-KHR_audio_environment-proposals.md), [06-structured-feedback.md](06-structured-feedback.md).*

## The pieces on the board

| Artifact | Where | Status | Content |
|---|---|---|---|
| **KHR_audio_emitter** | [PR #2137](https://github.com/KhronosGroup/glTF/pull/2137) (omigroup/glTF @ `KHR_audio`) | Open, Draft; last update 2026-04-16 | `audio[]` / `sources[]` / `emitters[]`; global + positional emitters; scene & node attachment (both plural `emitters` arrays as of Apr 2026); MP3-only base codec; Object Model pointers |
| **KHR_audio_graph** | [PR #2572](https://github.com/KhronosGroup/glTF/pull/2572) (facebook/glTF @ `codex/update-khr-audio-graph-only`) | Open, Draft; created 2026-04-22 | Document-level `graphs[]`, 16 node kinds, DAG-only, inputs bind emitter sources / outputs bind emitters; encoding metadata + extended source props layered onto KHR_audio_emitter |
| **KHR audio framework design doc** | [PR #2421](https://github.com/KhronosGroup/glTF/pull/2421) (Meta: Chintan Shah, Alexey Medvedev) | Open design discussion (2024) | Predecessor of the graph spec: source/oscillator, emitter/listener sinks, gain/delay/pitch-shift/channel ops/filter/reverb nodes; superseded in detail by #2572 |
| **MSFT_audio_emitter** | [PR #1400](https://github.com/KhronosGroup/glTF/pull/1400) | Historical vendor ext | Animation-triggered clips, randomized clips — features deliberately dropped from KHR_audio_emitter |
| **Layered architecture proposal** | [Issue #2561](https://github.com/KhronosGroup/glTF/issues/2561) (2026-02-07) | Open | Three layers: emitter (base) → graph (processing) → environment (listener/acoustics) |
| **Proposal repo** | [rudybear/KHR_audio_proposals](https://github.com/rudybear/KHR_audio_proposals) | Drafts | Includes **KHR_audio_environment draft** (listeners + environments/reverb + custom distance curve), gap analysis F1–F21 |
| **Reference impl** | [rudybear/AudioGraphJS](https://github.com/rudybear/AudioGraphJS) @ `feature/layered-extensions` | Working | Web Audio/TypeScript runtime, 4 examples, validator, 6 test suites |

## Layer responsibilities (as proposed in #2561)

1. **KHR_audio_emitter** — *what plays and where*: audio data, sources (gain, loop, autoplay, playbackRate), emitters (global/positional; cone + 3 distance models), scene/node attachment.
2. **KHR_audio_graph** — *how it's processed*: DAG of oscillator/gain/delay/waveshaper/8 biquad-style filters/splitter/merger/channelmixer/audiomixer; sources in via `inputs[]`, emitters out via `outputs[]`. Requires KHR_audio_emitter. Explicitly excludes reverb, spatialization, listener.
3. **KHR_audio_environment** — *how it's heard*: listener (equalpower/HRTF/custom, interaural distance), environments (parametric or IR reverb, scene-global or node zones), custom distance curves, per-emitter spatialization override. Draft only, not yet a PR.

## Precedent alignment (the standards story)

The same node/semantics lineage runs through three tiers:

- **W3C Web Audio API 1.0** (REC, 2021) — imperative browser API; PannerNode distance/cone model is what KHR_audio_emitter adopts verbatim (radians instead of degrees).
- **X3D 4.0 / ISO-IEC 19775-1:2023 Sound component** — the proof that Web Audio semantics can be *declarativized into an ISO standard*: 22 nodes mapping 1:1 to Web Audio, plus SpatialSound, AcousticProperties (on materials), ListenerPointSource, Doppler. X3D conformance levels map almost exactly onto the three glTF layers (Level 1 ≈ emitter, Level 2 ≈ graph, Level 3 ≈ environment).
- **MPEG-I Immersive Audio (ISO/IEC 23090-4)** — the 6DoF benchmark: source directivity/extent, acoustic environments, parametric reverb, occlusion/diffraction, Doppler, HOA.

## Process/hygiene blockers (independent of design)

- **PR #2572 CLA**: 3 of 6 committers unsigned at snapshot (Chintan Shah, utuere, rudybear) — practical merge blocker.
- **PR #2572 internal inconsistency**: bundles a copy of KHR_audio_emitter whose `node.KHR_audio_emitter.schema.json` and "Using Audio Emitters" prose still use singular `emitter`, while one example and the changes doc use plural `emitters`. Upstream #2137 switched to plural on 2026-04-16 — the copies must be re-synced or (better) the emitter copy dropped from #2572 in favor of a stated dependency on #2137.
- **No technical review yet on #2572** (zero review comments); Norbert Nopper asked for contributor attribution and roadmap-spreadsheet alignment.
- **#2137's existential question**: OMI is waiting on Khronos to say whether the graph framework supersedes or layers on top of their PR. Issue #2561's layered answer resolves this — that resolution should be stated *in both PRs*.
