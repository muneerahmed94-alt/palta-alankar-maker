# Palta Alankar Maker

A web-based tool for Indian classical music practice. Enter the first line of a palta (alankar) and the app generates all ascending and descending lines automatically, with playback using synthesized Piano or Harmonium sounds. Includes a MIDI keyboard interface for playing along.

**Live app:** https://muneerahmed94-alt.github.io/palta-alankar-maker/

Developed by Muneer Ahmed Shaik.

## Features

### Swara Selector (Chromatic Piano Keyboard UI)
- Full chromatic 12-key piano layout (7 white + 5 black) at the top of the input card
- **Western note labels are always fixed** (C, C#, D, D#, E, F, F#, G, G#, A, A#, B)
- **Swara labels move** based on the selected root key (Sa =). When Sa = C, "Sa" is on the C key; when Sa = D, "Sa" moves to the D key, etc.
- Click any non-fixed key to toggle between Shuddh and Komal/Teevra variants
- Sa and Pa are fixed (no variants in Indian classical music)
- Selection dynamically updates frequency mapping (`SWARA_SEMITONES[]`) and display names
- When a palta is displayed, changing a swara variant instantly re-renders with updated pitches and notation

### Display Notation Convention (Pt Ramanuj Dasgupta style)
The app follows the standard Indian classical notation convention:
- **Shuddh swaras**: uppercase — S R G P D N
- **Shuddh Ma**: lowercase **m** (because Teevra Ma takes the uppercase)
- **Komal swaras**: lowercase — r g d n
- **Teevra Ma**: uppercase **M**
- Example: `SRGmPDNS` = all Shuddh; `SRGMPDNS` = with Teevra Ma
- On "Generate", the input text field auto-corrects to match the selected swara variants (e.g., typing `SRG` with Komal Ga selected becomes `S R g`)

### Pattern Presets
Three categories of preset patterns:

**Common patterns** — 8 standard palta starting sequences:
- Sa Re Ga Ma, Sa Re Ga Re, Sa Re Sa Re Ga Re, Sa Ga Re Ma Ga Pa, Sa Re Ga Ma Pa, Sa Re Ga Ma Ga Re, Sa Re Ga Re Ga Ma, Sa Ma Ga Re

**Special patterns** — Fixed-base patterns where Sa is paired with each ascending swara:
- SR, SG, Sm, SP, SD, SN, SS' (Sa paired with each note going up)
- RS, GS, mS, PS, DS, NS, S'S (each note paired back to Sa)
- Default tempo: 60 BPM (vs 102 BPM for regular paltas)
- Aarohi walks up, avarohi mirrors back down from upper Sa

**My patterns** — Custom saved patterns:
- Sa Ga Sa Re, Sa Re Ga ma Sa Ga Re ma, Sa Re Re Sa Re Ga Re Sa, Sa .Ni Re .Ni Re Ga ma Ga

### Palta Generation
- Enter swaras in any format: full names (`Sa Re Ga Ma`), short (`S R G M`), or continuous (`SRGM`)
- Supports three octaves: lower (`.Sa`), middle (`Sa`), upper (`Sa'`)
- Octave markers work as prefix or suffix: `.N`, `N.`, `'S`, `S'` all valid
- **Nearest-octave auto-resolution**: When no octave marker is given, each note after the first is placed in the octave nearest to the previous note (e.g., `SNSN` becomes `S .N S .N` — N below Sa, not 6 steps above)
- Continuous strings support embedded octave markers: `S.NS.NSRGM` parses correctly
- Generates 7 ascending (aarohi) and 7 descending (avarohi) lines
- **Include Avarohi** checkbox in output card — live toggle to show/hide descending section
- Add or remove lines with +/- buttons on the last line
- Copy generated palta as plain text

### Sound Engine (Web Audio API, no samples)
- **Grand Piano** — Additive synthesis with 10 harmonics, inharmonicity modeling, hammer noise burst, soundboard resonance
- **Upright Piano** — Shorter decay, mid-frequency EQ boost for "boxy" character
- **Electric Piano** — FM synthesis (Rhodes-style) with tremolo LFO
- **Harmonium** — Sawtooth oscillators through lowpass + peaking EQ filters, bellows tremolo LFO
  - 6 reed configurations: Bass-Male, Bass-Male-Female, Bass-Male-Male, Male-Male-Female-Bass, Bass-Male-Female-Female, Bass-Bass-Male-Female
  - Doubled reeds use slight detuning (±4–6 cents) for natural chorus/beating effect
- Sound options are located in the Output Card (after "Generated Palta" header)

### Playback
- Adjustable tempo (60–480 BPM, default 102 for regular, 60 for special patterns)
- Beat counter (2, 3, 4, 6, 8) with optional skip
- Play aarohi only, avarohi only, or both
- Repeat mode for continuous practice
- Visual note highlighting during playback
- Left coupler (adds lower octave) for both Piano and Harmonium
- Selectable root key (Sa = C through B)

### MIDI Keyboard
- Connect any MIDI keyboard via Web MIDI API
- Same sound engine as palta playback (Piano / Harmonium with all options)
- Velocity sensitivity
- Sustain pedal support (MIDI CC 64)
- Sustain checkbox (Piano only) — notes ring out naturally without damping
- Coupler: None / Left / Right
- Octave shift: ±3 octaves
- Transpose: ±12 semitones
- Register-dependent sustain (bass notes ring longer than treble)
- Register-dependent damper release time
- Auto-detects MIDI devices with hot-plug support

## Project Structure

```
palta-alankar-maker/
  index.html    — Entire app (HTML + CSS + JS in a single file, no build step)
  README.md     — This file
```

## Architecture

The app is a single `index.html` file with no external dependencies (except Google Fonts). Everything runs client-side in the browser.

### CSS (inline `<style>`)
Organized into 8 sections:
1. **Base / Layout** — body, container, header, cards
2. **Input Card** — form elements, preset chip buttons
3. **Swara Selector** — chromatic piano keyboard with white/black keys, swara labels, note labels, selection states, fixed key styling
4. **Output Card** — palta line display, direction headers, add/delete buttons
5. **Playback Controls** — key/sound/beat selectors, tempo slider, play/stop buttons
6. **MIDI Card** — connection UI, sound options, octave/transpose adjustment buttons
7. **Footer** — developer credit
8. **Responsive** — mobile breakpoint at 600px (piano keys scale to percentage widths)

### JavaScript (inline `<script>`)
Six subsystems + event listeners:

#### 1. Swara Parser
Converts user input text into numeric note values (0–20 across three octaves). Handles full names, short names, continuous strings, and octave markers.

**Key features:**
- `parseSwara()` — Parses a single token. Returns `{ value, isShort, explicitOctave }`. The `explicitOctave` flag tracks whether the user provided `.` or `'` markers. Supports markers as prefix OR suffix.
- `trySplitContinuous()` — Tokenizes continuous strings like `S.NS.NSRGM` into individual swara tokens with attached octave markers.
- `nearestOctave()` — For notes without explicit octave markers, finds the octave placement (lower/middle/upper) that is closest to the previous note.
- `parseSequence()` — Two-phase parsing: (1) tokenize all swaras, (2) apply nearest-octave resolution for unmarked notes. First note always defaults to middle octave if unmarked.

**Display name system:**
- `SWARAS_FULL` / `SWARAS_SHORT` — Immutable constants for input parsing (case-insensitive).
- `SWARAS_FULL_DISPLAY` / `SWARAS_SHORT_DISPLAY` — Mutable arrays for output rendering. Updated by `updateSwaraDisplayName()` when swara variants change on the piano selector.
- `numToSwara()` — Reads from the `_DISPLAY` arrays, so rendered output always reflects the current Komal/Teevra selections.

#### 2. Palta Generator
Two generators:

**Standard** (`generatePalta()`) — Takes parsed notes and generates all ascending/descending lines by computing interval offsets from the first note and transposing up/down one step at a time.

**Special** (`generateSpecialPalta()`) — For SR/SG/.../SS' and RS/GS/.../S'S patterns. Each line pairs a fixed base note (Sa) with a different scale degree, walking up for aarohi and back down for avarohi.

Both generators produce the same `{ ascending, descending, landingTop, landingBottom }` structure, so rendering, playback, and copy work identically.

#### 3. Renderer
Builds DOM elements for each palta line. Each swara `<span>` gets `data-section`, `data-line`, `data-swar` attributes used by the playback engine for visual highlighting.

#### 4. Audio Engine
All synthesis uses the Web Audio API (no audio samples).

**Frequency mapping:**
- `SWARA_SEMITONES[]` — **Mutable** array mapping swara index (0–6) to semitone offset from Sa. Updated by the Swara Selector when Komal/Teevra variants are toggled.
- `SWARA_VARIANTS[]` — Configuration defining Shuddh and variant semitone values for each swara. Sa (index 0) and Pa (index 4) are `null` (fixed, no variants).
- `noteNumToFreq()` — Reads `SWARA_SEMITONES[swaraIdx]` to compute Hz. Automatically reflects current variant selections.

**Synthesis:**
- **Piano inharmonicity**: Partials are stretched using `freq * n * sqrt(1 + B*n²)` where B increases with pitch, modeling real piano string physics.
- **Harmonium reeds**: Sawtooth wave provides natural 1/n harmonic rolloff. Lowpass filter at `reed_freq * 6` tames harshness. Peaking EQ at `reed_freq * 2.5` adds nasal character.
- **Metronome**: Two sine oscillators (880 Hz + 1760 Hz) with instant attack and fast decay for a wood-block click sound.

#### 5. Playback Engine
Schedules notes on the Web Audio API timeline for sample-accurate timing. Uses `setTimeout` for visual highlighting (which doesn't need sample accuracy). Supports seamless aarohi→avarohi chaining and repeat loops.

**Key functions:**
- `doGenerate()` — Parses input, generates palta, renders, sets tempo to 102 BPM. Auto-corrects input text to reflect swara variants.
- `doGenerateSpecial(type)` — Generates special patterns (sa-x or x-sa), sets tempo to 60 BPM.
- `reRenderPalta()` — Re-generates and re-renders the current palta (handles both standard and special via `currentSpecialType` flag). Called when swara variants, include avarohi, or line counts change.
- `updateSwaraDisplayName()` — Updates display name arrays when a variant is toggled. Handles the special Ma convention (Shuddh=lowercase, Teevra=uppercase).
- `setTempo(bpm)` — Sets both the slider and input to the given BPM value.

#### 6. MIDI Keyboard
Uses `navigator.requestMIDIAccess()` to connect to external keyboards. Each note-on creates synth oscillators wrapped in a per-note `masterGain` node. On note-off, the masterGain is faded to zero (simulating a damper). The Map `midiActiveNotes` tracks all sounding notes for cleanup.

#### Event Listeners
- **Preset buttons** (common patterns) — Set input text and call `doGenerate()`.
- **Special preset buttons** — Call `doGenerateSpecial()` with the pattern type.
- **Swara Selector piano** — `updatePianoSwaraLabels()` places swara labels on correct chromatic keys based on Sa=. Click handler toggles variants and re-renders.
- **Key selector** (Sa=) — Updates piano swara labels and re-renders if a palta is displayed.
- **Include Avarohi** — Live toggle that re-generates the palta to show/hide descending section.
- **Sound/instrument options** — Show/hide piano-only and harmonium-only controls.

## Swara Numbering System

Each swara maps to a unique integer for easy transposition via addition:

| Octave | Sa | Re | Ga | Ma | Pa | Dha | Ni |
|--------|----|----|----|----|----|----|-----|
| Lower (mandra)  | 0  | 1  | 2  | 3  | 4  | 5  | 6  |
| Middle (madhya)  | 7  | 8  | 9  | 10 | 11 | 12 | 13 |
| Upper (taar)     | 14 | 15 | 16 | 17 | 18 | 19 | 20 |

Transposing a pattern up by one swara = adding 1 to every note value.

## Swara Variant System

Five of the seven swaras have Komal (flat) or Teevra (sharp) variants:

| Swara | Shuddh (semitones) | Variant (semitones) | Type |
|-------|-------------------|-------------------|------|
| Sa | 0 | — | Fixed |
| Re | 2 | 1 | Komal |
| Ga | 4 | 3 | Komal |
| Ma | 5 | 6 | Teevra |
| Pa | 7 | — | Fixed |
| Dha | 9 | 8 | Komal |
| Ni | 11 | 10 | Komal |

The Swara Selector piano UI is a chromatic 12-key layout:
- **White keys (7):** Fixed Western notes C, D, E, F, G, A, B
- **Black keys (5):** Fixed Western notes C#, D#, F#, G#, A#
- Swara labels are placed dynamically based on Sa = key selection
- Selecting a variant updates `SWARA_SEMITONES[idx]` which is read by `noteNumToFreq()` for both palta playback and MIDI keyboard

## Frequency Mapping

Swaras use the major scale (shuddh swaras):
- Sa=unison, Re=+2, Ga=+4, Ma=+5, Pa=+7, Dha=+9, Ni=+11 semitones
- Base frequency is set by the key selector (Sa=C → 261.63 Hz, Sa=A → 440 Hz, etc.)
- Formula: `freq = baseFreq * 2^(semitones/12)`
- When Komal/Teevra variants are selected, `SWARA_SEMITONES` is updated and the formula uses the new values automatically.

## How to Extend

### Adding a new piano type
1. Create a function `createMyPianoNote(audioCtx, dest, freq, startTime, duration)` following the pattern of `createGrandPianoNote`.
2. Add it to the `createPianoNote` dispatcher (the `if/else` chain checking `pianoType`).
3. Add a `<option>` to both the palta sound select (`#piano-type-select`) and the MIDI sound select (`#midi-piano-type-select`).

### Adding a new harmonium reed configuration
1. Add a new `else if (reedType === 'your-type')` block inside `createHarmoniumNote`, pushing reeds with the desired `{ freq, amp, detune }` values.
2. Add a `<option>` to both reed selects (`#reed-select` and `#midi-reed-select`).

### Adding a new preset pattern
Add a `<button class="preset-btn" data-pattern="Sa Ga Re Ma">Sa Ga Re Ma</button>` inside the appropriate `.preset-chips` div. The event listener is already wired up via `querySelectorAll('.preset-btn:not(.special-preset-btn)')`.

### Adding a new special pattern type
1. Add a new `else if` branch inside `generateSpecialPalta()` for the new type.
2. Add a `<button class="preset-btn special-preset-btn" data-special="your-type">Label</button>` in the special patterns `.preset-chips` div.
3. The click handler is already wired via `querySelectorAll('.special-preset-btn')`.

### Adding a completely new instrument
1. Create a synth function: `createMyInstrumentNote(audioCtx, dest, freq, startTime, duration, ...params)` that returns an array of started oscillators/nodes.
2. Add the instrument to the sound select dropdowns.
3. Update `scheduleSegment()` in the playback engine to call your function.
4. Update `playMidiNote()` in the MIDI section to call your function.
5. Add any instrument-specific UI options with appropriate show/hide CSS classes.

### Adding a new swara variant
The current system supports Shuddh + one variant per swara. To add more variants (e.g., both Komal and Ati-Komal):
1. Extend `SWARA_VARIANTS[]` entries to include additional variant semitone values.
2. Update the piano click handler to cycle through variants or use a different selection mechanism.
3. Update `updateSwaraDisplayName()` with display names for the new variants.
4. Update `PIANO_SWARA_LABELS[]` with labels for the new variants.

### Modifying sustain behavior (MIDI)
Piano sustain duration is calculated in `playMidiNote()`:
```javascript
const pianoDur = Math.max(0.6, 3.6 * Math.pow(0.98, adjustedNote - 21));
```
- `3.6` controls the bass sustain time (~3.2s at MIDI note 21)
- `0.98` controls how fast sustain decreases with pitch
- `0.6` is the minimum sustain for the highest notes

Damper release time:
```javascript
const pianoReleaseTime = Math.max(0.05, 0.2 - adjustedNote * 0.0013);
```

## Browser Requirements

- **Web Audio API** — all modern browsers
- **Web MIDI API** — Chrome, Edge, Opera (not supported in Firefox/Safari without flags)
- **Google Fonts** — Poppins font loaded from CDN (falls back to system sans-serif)

## Running Locally

No build step required. Just open `index.html` in a browser:

```bash
open index.html
# or
python -m http.server 8000   # then visit http://localhost:8000
```

For MIDI keyboard support, you must serve over HTTPS or localhost (Web MIDI API security requirement).

## License

This project is open source.
