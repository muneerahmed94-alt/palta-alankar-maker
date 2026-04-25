# Palta Alankar Maker

A web-based tool for Indian classical music practice. Enter the first line of a palta (alankar) and the app generates all ascending and descending lines automatically, with playback using synthesized Piano or Harmonium sounds. Includes a MIDI keyboard interface for playing along.

## Features

### Swara Selector (Piano Keyboard UI)
- One-octave piano keyboard layout at the top of the input card
- 7 white keys = Shuddh (natural) swaras: Sa Re Ga ma Pa Dha Ni
- 5 black keys = Komal/Teevra variants: re, ga, Ma (Teevra), dha, ni
- Click any non-fixed key to toggle between Shuddh and Komal/Teevra
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
- On "Generate Palta", the input text field auto-corrects to match the selected swara variants (e.g., typing `SRG` with Komal Ga selected becomes `S R g`)

### Palta Generation
- Enter swaras in any format: full names (`Sa Re Ga Ma`), short (`S R G M`), or continuous (`SRGM`)
- Supports three octaves: lower (`.Sa`), middle (`Sa`), upper (`Sa'`)
- Octave markers work as prefix or suffix: `.N`, `N.`, `'S`, `S'` all valid
- **Nearest-octave auto-resolution**: When no octave marker is given, each note after the first is placed in the octave nearest to the previous note (e.g., `SNSN` becomes `S .N S .N` — N below Sa, not 6 steps above)
- Continuous strings support embedded octave markers: `S.NS.NSRGM` parses correctly
- Generates 7 ascending (aarohi) and 7 descending (avarohi) lines
- Add or remove lines with +/- buttons on the last line
- Copy generated palta as plain text
- 8 common preset patterns included

### Sound Engine (Web Audio API, no samples)
- **Grand Piano** — Additive synthesis with 10 harmonics, inharmonicity modeling, hammer noise burst, soundboard resonance
- **Upright Piano** — Shorter decay, mid-frequency EQ boost for "boxy" character
- **Electric Piano** — FM synthesis (Rhodes-style) with tremolo LFO
- **Harmonium** — Sawtooth oscillators through lowpass + peaking EQ filters, bellows tremolo LFO
  - 6 reed configurations: Bass-Male, Bass-Male-Female, Bass-Male-Male, Male-Male-Female-Bass, Bass-Male-Female-Female, Bass-Bass-Male-Female
  - Doubled reeds use slight detuning (±4–6 cents) for natural chorus/beating effect

### Playback
- Adjustable tempo (60–480 BPM)
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
Organized into 7 sections:
1. **Base / Layout** — body, container, header, cards
2. **Input Card** — form elements, preset chip buttons, options row
3. **Swara Selector** — piano keyboard layout with white/black keys, selection states
4. **Output Card** — palta line display, direction headers, add/delete buttons
5. **Playback Controls** — key/sound/beat selectors, tempo slider, play/stop buttons
6. **MIDI Card** — connection UI, sound options, octave/transpose adjustment buttons
7. **Responsive** — mobile breakpoint at 600px (piano keys scale to percentage widths)

### JavaScript (inline `<script>`)
Six subsystems, each clearly marked with section headers:

#### 1. Swara Parser
Converts user input text into numeric note values (0–20 across three octaves). Handles full names, short names, continuous strings, and octave markers.

**Key features:**
- `parseSwara()` — Parses a single token. Returns `{ value, isShort, explicitOctave }`. The `explicitOctave` flag tracks whether the user provided `.` or `'` markers.
- `trySplitContinuous()` — Tokenizes continuous strings like `S.NS.NSRGM` into individual swara tokens with attached octave markers.
- `nearestOctave()` — For notes without explicit octave markers, finds the octave placement (lower/middle/upper) that is closest to the previous note. This is critical for patterns like `SNSN` where N should be below Sa, not above.
- `parseSequence()` — Two-phase parsing: (1) tokenize all swaras, (2) apply nearest-octave resolution for unmarked notes. First note always defaults to middle octave if unmarked.

**Display name system:**
- `SWARAS_FULL` / `SWARAS_SHORT` — Immutable constants for input parsing (case-insensitive).
- `SWARAS_FULL_DISPLAY` / `SWARAS_SHORT_DISPLAY` — Mutable arrays for output rendering. Updated by `updateSwaraDisplayName()` when swara variants change on the piano selector.
- `numToSwara()` — Reads from the `_DISPLAY` arrays, so rendered output always reflects the current Komal/Teevra selections.

#### 2. Palta Generator
Takes parsed notes and generates all ascending/descending lines by computing interval offsets from the first note and transposing up/down one step at a time.

#### 3. Renderer
Builds DOM elements for each palta line. Each swara `<span>` gets `data-section`, `data-line`, `data-swar` attributes used by the playback engine for visual highlighting.

#### 4. Audio Engine
All synthesis uses the Web Audio API (no audio samples). Key design decisions:

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

**Swara Selector integration:**
- `updateSwaraDisplayName()` — Updates display name arrays when a variant is toggled. Handles the special Ma convention (Shuddh=lowercase, Teevra=uppercase).
- Piano key click handler — Toggles `selected` CSS class, updates `SWARA_SEMITONES[]` and display names, then calls `reRenderPalta()` if a palta is displayed.
- `doGenerate()` — After parsing, auto-corrects the input text field to reflect current swara variant display names.

#### 6. MIDI Keyboard
Uses `navigator.requestMIDIAccess()` to connect to external keyboards. Each note-on creates synth oscillators wrapped in a per-note `masterGain` node. On note-off, the masterGain is faded to zero (simulating a damper). The Map `midiActiveNotes` tracks all sounding notes for cleanup.

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

The Swara Selector piano UI maps these to a real piano octave layout:
- **White keys (7):** Shuddh swaras at semitones 0, 2, 4, 5, 7, 9, 11
- **Black keys (5):** Variants at semitones 1, 3, 6, 8, 10

Selecting a variant updates `SWARA_SEMITONES[idx]` which is read by `noteNumToFreq()` for both palta playback and MIDI keyboard.

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
Add a `<button class="preset-btn" data-pattern="Sa Ga Re Ma">Sa Ga Re Ma</button>` inside `.preset-chips`. The event listener is already wired up via `querySelectorAll('.preset-btn')`.

### Adding a completely new instrument
1. Create a synth function: `createMyInstrumentNote(audioCtx, dest, freq, startTime, duration, ...params)` that returns an array of started oscillators/nodes.
2. Add the instrument to the sound select dropdowns.
3. Update `scheduleSegment()` in the playback engine to call your function.
4. Update `playMidiNote()` in the MIDI section to call your function.
5. Add any instrument-specific UI options with appropriate show/hide CSS classes.

### Adding a new swara variant
The current system supports Shuddh + one variant per swara. To add more variants (e.g., both Komal and Ati-Komal):
1. Extend `SWARA_VARIANTS[]` entries to include additional variant semitone values.
2. Add more black keys to the piano HTML (may need CSS adjustments).
3. Update the click handler to cycle through variants or use a different selection mechanism.
4. Update `updateSwaraDisplayName()` with display names for the new variants.

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
