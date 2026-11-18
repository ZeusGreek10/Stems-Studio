# Stems Studio

Native macOS app (Apple Silicon first) that separates a song into stems, transcribes them to MIDI, recreates parts with sample-based instruments, then mixes and exports.

## Read first

- `docs/Stems_Studio_Claude_Code_Specification.txt` is the single running spec. New features are added to this file, not to separate spec files.
- `docs/renderings/` holds the user-approved screens: `StemsStudio-analyze.png`, `StemsStudio-recreate.png`, `StemsStudio-mix.png`.

## Rules

- **Build status: not started.** Do not begin building until the user says to.
- Before coding, produce the section 11 deliverables: architecture plan, dependencies with licence checks, model evaluation matrix, milestones and test plan.
- Two languages (approved): Swift for the interface, playback, mixing, instrument hosting and export; Python for AI analysis only, as a separate background process.
- Match the approved renderings as closely as possible. If a built screen does not look like its rendering, redesign and rebuild it to match. If a technical constraint prevents a match, stop and ask before deviating.
- The song, waveforms, notes, levels and preset names in the renderings are sample data. Never hard-code them, and never fabricate AI output, notes, waveforms, progress or benchmarks in the app.
- All analysis runs locally on the Mac. No song upload.
- Verify licences for models, weights, sample libraries and plugins before bundling. Do not use Apple's Logic Pro instruments or samples.
- Workflow: build the requested functions, test them, then ask the user what to do next. Do not build DMGs or releases unless the user asks.
