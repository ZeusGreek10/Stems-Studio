# Stems Studio: Pre-Coding Plan

Date: 2026-11-18 | Covers spec v1.3, section 11 deliverables (a) to (d) | Build status: not started

**How to read the licence and status marks**

- **Checked**: confirmed against a source on 2026-11-18 (listed at the end).
- **To confirm**: from general knowledge, not re-checked. Claude Code must confirm against the project's own licence file before adding the dependency.
- **To measure**: no number exists yet. Nothing in this plan is a benchmark result. All performance and accuracy figures come from Phase 0, run on the Mac Studio M2 Max.

---

## 0. Decisions needed from the user before Phase 0

These change what gets built, so they come first.

1. **Own use, or share/sell?** No six-stem separation model has clean commercial terms (section C). For private use on your own Mac the risk is low. If the app will be given or sold to others, the separation model must be downloaded by each user under its own terms, or a commercially licensed model must be found. *Recommended assumption: own use first, models never bundled inside the app.*
2. **Apple Developer account?** Signing and notarizing needs a paid Apple Developer ID. Without it the app still runs on your Mac, but macOS shows an "unidentified developer" warning on first open. *Recommended: skip until the app is shared.*
3. **Test songs.** Phase 0 needs 5 to 10 worship recordings you have the right to process, ideally 1 or 2 with original multitracks for ground truth.
4. **Minimum macOS.** *Proposed: macOS 14 on Apple Silicon only.* (MLX's minimum version is to confirm.)

---

## A. Architecture and implementation plan

### A1. Processes

| Process | Language | Does | Never does |
|---|---|---|---|
| App | Swift (SwiftUI) | Interface, project store, playback, mixing, instrument hosting, rendering, export, job orchestration | ML inference |
| Analysis worker | Python | Separation, beat/tempo, key/chords/sections, drum and pitch transcription | Touch the project database, play audio |

The app launches the worker as a child process. One job runs at a time by default (memory). A worker crash or cancel cannot affect playback because it is a separate process.

### A2. App to worker protocol (MVP)

- Transport: newline-delimited JSON over the worker's stdin/stdout. stderr is captured as a log.
- Every message carries `protocol_version`, `job_id`, `type`.
- App to worker: `run` (job kind, input paths, output directory, model id, parameters), `cancel`.
- Worker to app: `progress` (real fraction and stage name, only when measured), `result` (artifact paths, model id and version, diagnostics), `error` (code, message).
- Audio never travels through the pipe. The worker reads WAV files and writes WAV/JSON/MIDI into a temporary job directory. The app validates the result, then moves it into the project atomically. A killed job leaves only a temp directory to delete.
- Schemas live in `schemas/` (spec section 7) and are validated on both sides.
- The app decodes MP3/M4A/AIFF/FLAC to WAV itself (AVFoundation) before any job. The worker only ever sees WAV, so no ffmpeg is needed.

### A3. Provider abstraction

Each analysis capability is an interface with swappable implementations, selected by `models/manifest.json`:

- `SeparationProvider`: input WAV to six aligned stems, same length and sample rate as the input.
- `BeatProvider`: beats, downbeats, tempo map.
- `HarmonyProvider`: key, chords, sections (all marked preliminary).
- `DrumTranscriber`: onsets with class, velocity estimate, confidence.
- `PitchTranscriber`: notes with onset, offset, velocity estimate, optional bend, confidence.

The manifest records per model: id, version, source URL, file hashes, licence, licence status, size. The UI reads capabilities from the manifest, so swapping a model needs no interface change.

### A4. Time and synchronisation

- One timeline origin: sample 0 of the decoded original at the project sample rate. All positions are 64-bit sample counts.
- Stems must match the original's length exactly. The app checks this on import of a result and rejects misaligned output.
- Tempo map: list of (sample position, BPM, meter) from the beat grid. Beats are derived from samples, never the other way round, so there is no accumulated drift.
- MIDI events are stored in samples. Beat positions are computed for display and for MIDI file export.

### A5. Audio engine (Swift, AVAudioEngine)

Per track signal path:

```
original stem player ─┐
                      ├─ blend ─ EQ ─ compressor ─ fader/pan ─┬─ master ─ limiter ─ output
recreated player ─────┘                                       └─ sends ─ reverb/delay buses ─ master
```

- Original / Recreated / Blend is the blend stage, available in Recreate and Mix.
- Effects use Apple's built-in Audio Units (EQ, dynamics, reverb, delay, peak limiter). No third-party effect code in the MVP.
- Meters come from taps, read by the UI on a timer. Nothing on the audio thread allocates, locks or logs.
- Analyze and Recreate use the same engine with effects bypassed and no mixer shown.
- Export uses the same graph in offline (manual rendering) mode, so the mixdown matches what was heard. Every stem export renders from sample 0 for the full project length.

### A6. Instruments and recreation

- Instrument format: SFZ sample libraries, played by the sfizz engine (C/C++ library) wrapped as an in-process instrument node.
- Live preview: the instrument node is fed scheduled MIDI from the MIDI lane.
- Render: the same instrument rendered offline to `Renders/recreated-<stem>.wav`, then shown in the waveform lane. Render cache key = MIDI hash + instrument id + preset + parameters. Changing a preset re-renders that stem only and never re-runs separation.
- Third-party Audio Unit instruments come later (spec phase 3+) behind the same `InstrumentHost` interface.

### A7. Project store

- Package `Song.stemsstudio/` as laid out in spec section 6.
- `project.sqlite` (via GRDB): tracks, regions, MIDI events, tempo events, jobs, instrument assignments, effect chains, export presets, schema version with migrations.
- Large files (audio, renders, cache) stay as files, referenced by relative path and content hash.
- Analysis cache key = input hash + model id + model version + parameters.
- Undo/redo: every edit is a command with an inverse, registered with the system undo manager and written to SQLite in one transaction.
- Missing plug-in or sample library on reopen: the project opens, the track falls back to its last render or the original stem, and a clear notice is shown.

### A8. Packaging

- The Python worker ships inside the app as a self-contained runtime (python-build-standalone) under `Contents/Resources/python`, with dependencies pre-installed. The user installs nothing.
- Models are not inside the app. A model manager downloads them on request, shows size and licence, verifies hashes, and stores them in Application Support.
- Builds run on GitHub Actions macOS runners. DMGs are built only when the user asks (CLAUDE.md).

---

## B. Dependencies and licence checks

### B1. Swift side

| Dependency | Purpose | Licence | Status |
|---|---|---|---|
| SwiftUI, AVFAudio/AVAudioEngine, AudioToolbox, Accelerate | UI, playback, built-in effects, MIDI file I/O, waveform peaks | Apple system frameworks | No bundling issue |
| GRDB.swift 7.x | SQLite access and migrations | MIT | Checked. Needs Swift 6 / Xcode 16 or later |
| sfizz | SFZ instrument playback | BSD-2-Clause | Checked. Build with its default dr_libs audio backend (MIT-0) rather than libsndfile (LGPL-2.1) |

No other Swift packages are planned for the MVP. Anything added later goes through the same check.

### B2. Python side

| Dependency | Purpose | Licence | Status |
|---|---|---|---|
| python-build-standalone runtime | Bundled Python | To confirm | Widely used for embedding; every binary inside must be signed individually |
| NumPy | Arrays | BSD-3-Clause | To confirm |
| soundfile (libsndfile) | WAV read/write | BSD-3-Clause (LGPL-2.1 library, dynamically linked) | To confirm |
| librosa | Onsets, chroma, baseline rhythm and harmony features | ISC | To confirm |
| PyTorch | Model runtime (if a PyTorch model wins) | BSD-3-Clause | To confirm. Large; bundled size to measure in Phase 0 |
| MLX | Model runtime on Apple Silicon (if an MLX port wins) | MIT | To confirm |
| ONNX Runtime | Lightweight model runtime (Basic Pitch, Beat This!) | MIT | To confirm |
| basic-pitch | Bass/melodic notes | Apache-2.0, code and weights | Checked. Ships CoreML and ONNX versions of its model, so TensorFlow should be avoidable (to confirm for the chosen Python version) |
| beat_this | Beats and downbeats | MIT, code and weights | Checked |
| mido | MIDI file writing in the worker | MIT | To confirm |

Aim: pick **one** heavy runtime (PyTorch **or** MLX), not both, to keep the app small.

### B3. Sample libraries (candidates, to audition in Phase 3)

| Library | Instrument | Licence | Status |
|---|---|---|---|
| Virtuosity Drums | Acoustic kit with hi-hat steps, flams, rolls | CC0 | Checked |
| Karoryfer Growlybass, Black And Blue Basses, Fashionbass | Electric basses | CC0-1.0 | Checked |

Excluded: Apple Logic Pro / GarageBand instruments and samples (licence forbids repackaging; CLAUDE.md rule).

---

## C. Model evaluation matrix and feasibility risks

### C1. Stem separation

| Candidate | Stems | Weights licence | Runs on Apple Silicon? | Phase 0 result |
|---|---|---|---|---|
| **HTDemucs-6s** (Meta) | vocals, drums, bass, guitar, piano, other | **Disputed.** Code is MIT. Some hosts label the weights MIT; another finds no explicit grant for the weights. A public question to the author about this is unanswered | Yes. MLX port exists; its page reports about 0.6 s per 15 s of audio on an M5 Max (not our machine). Piano quality is described as limited | To measure |
| **BS-RoFormer-SW** | vocals, drums, bass, guitar, piano, other | **Unclear, likely non-commercial.** Original listing said "unknown"; one host labels it CC-BY-NC-SA-4.0; original uploader's account was deleted, file now on a mirror (about 700 MB) | **Unproven.** The main inference package is CUDA/CPU only and has removed Apple GPU support. Needs an MLX or ONNX/Core ML route, or CPU | To measure |
| MelBand-RoFormer specialists | Single stems (Enhanced mode) | Varies per model | To verify per model | Later phase |

Columns to fill for each candidate in Phase 0: wall-clock time and peak memory on the M2 Max for a 4 to 5 minute song; listening notes per stem (clarity, bleed, artefacts); SDR where multitracks exist; **drum and bass transcription accuracy downstream** (the deciding measure); stability over 10 runs; stem length equals input length.

Decision rule (proposed): choose the model with the best downstream drum and bass transcription that runs under an agreed time and memory limit on the M2 Max. HTDemucs-6s is the fallback because it is known to run on Apple Silicon.

### C2. Rhythm and harmony

| Task | Candidate | Licence | Notes |
|---|---|---|---|
| Beats, downbeats, tempo | **Beat This!** | MIT (code and weights), checked | Primary. Tempo map built from its beat times |
| Beats (fallback) | librosa beat tracker | ISC, to confirm | No model download |
| Key, chords, sections | Feature-based baseline (chroma templates, self-similarity) | Own code | Preliminary only, always user-correctable. A dedicated model is a later upgrade |

### C3. Transcription

| Task | Candidate | Licence | Notes |
|---|---|---|---|
| Bass notes | **Basic Pitch** + monophonic clean-up (one note at a time, octave-error fix, minimum length) | Apache-2.0, checked | Velocity and pitch bend are estimates |
| Drums (kick, snare, hi-hat) | **Baseline:** onset detection + per-onset multi-label classifier on frequency bands | Own code | No licence risk; proves the pipeline end to end |
| Drums (upgrade) | **ADTOF** (five classes: kick, snare, hi-hat, toms, cymbals) | **Not yet verified** | Research papers describe it as accurate for five classes. Licence, framework and Mac support must be checked before use |
| Guitar, piano | Basic Pitch, labelled experimental | Apache-2.0 | Not an MVP requirement |

Confidence shown in the UI is the model's own estimate and is labelled as such until measured against labelled fixtures.

### C4. Feasibility risks

| # | Risk | Likelihood | Effect | Mitigation |
|---|---|---|---|---|
| 1 | No six-stem model has clean commercial terms | High | Blocks sharing or selling as-is | Never bundle weights; user-initiated download with licence shown; decision 1 above; provider abstraction allows a licensed model later |
| 2 | Best separation model does not run well on Apple Silicon | Medium | Slow or unusable primary model | Phase 0 tests MLX, ONNX/Core ML and CPU routes; HTDemucs-6s MLX as fallback |
| 3 | Drum transcription accuracy too low on real worship mixes (ghost notes, cymbal wash, bleed) | Medium | Recreated drums sound wrong | Benchmark early; uncertain-note marking; fast correction tools; ADTOF upgrade |
| 4 | Recreated audio does not sound realistic enough with free sample libraries | Medium | Core promise weakened | Audition libraries in Phase 3; velocity layers and round robin required; third-party AU support next |
| 5 | Hosting sfizz inside AVAudioEngine and rendering offline, sample-accurate | Medium | Blocks Recreate | Phase 0 spike: render a known MIDI file offline and compare against expected onsets |
| 6 | Signing and notarizing a bundled Python runtime | Medium | Blocks distribution, not own use | Sign every binary individually; keep Python under Resources; test on a clean Mac |
| 7 | App size and memory with PyTorch plus models | Medium | Large download, memory pressure | One runtime only; models downloaded on demand; one job at a time |
| 8 | Variable-tempo and live recordings break the beat grid | Medium | MIDI drifts against audio | Sample-based positions; tempo map from actual beats; manual beat-grid correction |
| 9 | Phase 0 must run on the Mac Studio, which has no Python or Xcode | Certain | Benchmarks cannot be run by hand | GitHub Actions builds a self-contained benchmark tool; the user runs it and sends back the report file |
| 10 | Upstream model hosting disappears (already happened once for BS-RoFormer-SW) | Medium | Downloads break | Pin hashes in the manifest; support importing a model file manually |

---

## D. Milestones and test plan

### D1. Milestones

Each phase ends with the listed proof. Per CLAUDE.md, Claude Code stops after each phase, reports, and asks what to do next.

| Phase | Goal | Done when |
|---|---|---|
| **0. Feasibility** | Command-line pipeline and benchmarks, no UI | Benchmark tool runs on the M2 Max; matrix in C1 filled with measured numbers; separation model chosen; sfizz offline-render spike passes; licences in section B confirmed |
| **1. App shell** | SwiftUI shell matching the three renderings (empty states), import, original waveform, shared transport, timeline, save/reopen | Import a song, play it, zoom and scroll, save, reopen. Screens compared against renderings |
| **2. Analyze** | Worker integration, six stems, beats/tempo/sections, drum and bass MIDI, note review | Spec MVP items 1 to 4. Cancel and crash recovery proven |
| **3. Recreate** | Instruments, paired MIDI/audio lanes, piano roll, render/re-render, Original/Recreated/Blend | Spec MVP items 5 to 7 |
| **4. Mix and export** | Mixer, EQ, compressor, sends, master, export of mix, stems, MIDI | Spec MVP items 8 to 10. All acceptance tests in D2 pass |

The spec's week ranges are kept as rough planning only. Real pace depends on Phase 0 results.

### D2. Test plan

**Automated, every change (GitHub Actions)**

| Area | Test |
|---|---|
| Project store | Save/reopen round trip returns identical state; schema migration from every older version; undo then redo returns identical state |
| Protocol | Requests and results validate against `schemas/`; unknown protocol version is rejected cleanly |
| Worker | pytest on tiny synthetic fixtures: stems same length as input; outputs deterministic for fixed settings |
| Cache | Changing an instrument preset invalidates only that stem's render; changing a MIDI note invalidates only that stem's render; neither re-runs separation |
| Sync | A click track through separation, transcription and render: every MIDI onset and rendered click within a set tolerance of the source, at the start and at the end of a long file; repeated with a tempo change |
| Export | All stems and the mixdown have the same start and the same length; with effects off and faders at unity, the stems sum to the mixdown (null test) |
| Robustness | Kill the worker mid-job: project opens intact, job marked failed, temp files removed. Cancel mid-job: same. Missing sample library or plug-in: project opens with a notice |
| Honesty rules | No sample data in production code paths; empty MIDI lanes stay empty; progress only reports measured values |
| UI | Snapshot images of the three tabs compared against `docs/renderings/` for layout |

**On the Mac Studio (benchmark tool, Phase 0 and before each release)**

| Area | Test |
|---|---|
| Separation | Time, peak memory, stability on the user's test songs |
| Transcription accuracy | Drum onsets and bass notes scored against labelled fixtures (hit within a set time window; precision, recall, F-measure per drum class and for bass notes) |
| Playback under load | Play a full project while a separation job runs: no audio dropouts reported by the engine |

**Labelled fixtures without licence problems**: render known MIDI drum and bass parts through the bundled instruments, mix them with other material, and use the original MIDI as ground truth. Add real recordings with multitracks from the user where available.

**Manual listening (user)**: A/B original versus recreated on each test song; notes recorded in the benchmark report.

---

## Sources checked (2026-11-18)

- HTDemucs weights licence question, unanswered: https://huggingface.co/adefossez/HTDemucs/discussions/1
- HTDemucs and BS-RoFormer-SW weight terms as listed by one host: https://huggingface.co/Ryan5453/unblend
- HTDemucs-6s MLX port (speed figure, MIT label): https://huggingface.co/schism-audio/htdemucs-6s-mlx
- HTDemucs-6s piano quality note: https://huggingface.co/puar-playground/htdemucs
- BS-RoFormer-SW inference package (MIT code, no Apple GPU support, mirror host, size): https://github.com/openmirlab/bs-roformer-infer
- BS-RoFormer-SW original listing ("License: unknown"): https://huggingface.co/jarredou/BS-ROFO-SW-Fixed/commit/f99fc5c2e4bc201413cfc1ca5767cf62ddb1a9a4
- Basic Pitch (Apache-2.0, bundled model formats): https://github.com/spotify/basic-pitch and https://huggingface.co/schism-audio/basic-pitch-mlx
- Beat This! (MIT code and weights): https://huggingface.co/musetric/beat-this-onnx and https://huggingface.co/cstr/beat-this-GGUF
- ADTOF five drum classes: https://webspace.eecs.qmul.ac.uk/s.e.dixon/pub/2024/482_lbd.pdf
- sfizz (BSD-2-Clause, backends): https://github.com/sfztools/sfizz and https://sfzformat.com/software/players
- GRDB.swift (MIT, requirements): https://swiftpackageregistry.com/groue/GRDB.swift and https://dev.co/databases/open-source/grdb-swift
- Virtuosity Drums (CC0): https://www.kvraudio.com/product/virtuosity-drums-by-versilian-studios/details
- Karoryfer basses (CC0-1.0): https://sfzinstruments.github.io/basses
- Bundled Python layout and signing: https://pyappdist.readthedocs.io/en/latest/platforms/macos-app.html
- Apple Logic Pro sample content terms: https://support.apple.com/en-sg/101908
