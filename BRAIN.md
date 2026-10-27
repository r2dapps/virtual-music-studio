# Virtual Music Studio (MAHATI) — System Architecture Blueprint

This document acts as the engineering brain for MAHATI (`virtual-music-studio`). It captures the audio routing graphs, state machines, instrument voice lifecycles, and data contracts to allow rapid comprehension, debugging, and extension without having to trace through monolithic source files.

---

## 1. System Philosophy & Constraints

1. **Zero External Audio Dependencies:** All sound generation is synthesized procedurally in real-time using the native HTML5 Web Audio API (`AudioContext`). No static WAV/MP3 soundbanks are required for playback, ensuring zero asset download latency and full offline operation.
2. **Deterministic Polyphony & Holding Voice Lifecycle:** Notes are tracked via unique voice handles keyed by note strings (e.g., `'C4'`, `'F#3'`). Long-press holding voices sustain continuously until explicit `pointerup`, `pointercancel`, or `touchend` events fire, preventing audio pops through exponential/linear gain de-allocation.
3. **Multi-Touch Concurrency:** Pointer events are assigned independent tracking IDs across multi-touch screens, enabling two-handed piano playing, simultaneous guitar strumming, and drum stage accompaniment.
4. **Adaptive UI Layout:** Single-page workstation architecture that scales across desktop monitors, tablets, and mobile devices in landscape orientation, with standalone PWA detection and full touch-action isolation (`touch-action: none`).

---

## 2. Web Audio Master Signal Graph

All sound sources connect into a unified master signal chain before routing to the hardware destination:

```
[Voice Generators]
  ├── Grand Piano (Dual Osc + LFO Detune)
  ├── Physics Guitar (Pluck Impulse + String Resonance)
  ├── Bowed Strings (Continuous Bandpass Sawtooth)
  ├── Concert Harp (Resonant Pluck Generators)
  ├── Sub-Bass & Marimba (Sine Sub + Triangle Transient)
  ├── Drum Kit (Procedural Noise Burst + Low Oscillators)
  └── XY Pad Synth (Dual Detuned Saw + Dynamic Biquad)
         │
         ▼
    [Master Bus]
         │
         ├───► [Delay Send] ──► [Stereo Delay Node] ──► [Feedback Gain (0.35)] ──┐
         │                                                                       │
         ▼                                                                       │
    [Dynamics Compressor] ◄──────────────────────────────────────────────────────┘
      - Threshold: -18 dB
      - Knee: 14 dB
      - Ratio: 4.5:1
      - Attack: 0.003s
      - Release: 0.25s
         │
         ▼
    [Master Volume GainNode] (0.0 to 1.0)
         │
         ├───► [AnalyserNode] (fftSize: 256, smoothing: 0.8) ──► Canvas Visualizer
         │
         ▼
    [AudioContext.destination] (Hardware DAC)
```

---

## 3. Instrument Synthesis Engines

### 3.1 Polyphonic Grand Piano
- **Voice Architecture:** Dual detuned oscillators (`osc1` triangle/sine, `osc2` fine-tuned saw/triangle +3 to +6 cents) feeding into an acoustic Biquad low-pass filter (cutoff 2800 Hz) and gentle vibrato LFO.
- **Sustain Handling:**
  - `startSustainedPianoVoice(freq)`: Initializes an envelope holding gain at sustain level (0.42 to 0.70).
  - `stopSustainedPianoVoice(note)`: Reads the active voice from `activePianoVoices[note]`, schedules `setTargetAtTime(0.0001, now, 0.08)`, and calls `stop()` after 220ms release decay.
- **Key Range:** Polyphonic chromatic bed with configurable octave shift (`currentOctaveOffset`: -1 to +2).

### 3.2 Physics Guitar & Virtual Fretboard
- **Strings Configuration:** 6 strings tuned to standard EADGBE:
  - E2: 82.41 Hz
  - A2: 110.00 Hz
  - D3: 146.83 Hz
  - G3: 196.00 Hz
  - B3: 246.94 Hz
  - E4: 329.63 Hz
- **Fret Math:** For string base frequency $f_{0}$ and fret $n \in [0, 12]$:
  $$f = f_{0} \cdot 2^{n / 12}$$
- **Pluck Dynamic:** Excitation pulse with rapid decay and harmonic overtones, combined with canvas spring vibration physics (`y(t) = y_0 + A \cdot e^{-\lambda t} \sin(\omega t)`).

### 3.3 Procedural Drum Kit Stage
- **Kick:** Sine sweep starting from 145 Hz dropping to 38 Hz in 0.08s, paired with a sub-click envelope.
- **Snare:** Dual component: 180 Hz body oscillator plus high-pass filtered white noise buffer decay (0.18s).
- **Hi-Hats:** Band-pass filtered metallic noise bursts (closed: 0.04s, open: 0.32s).
- **Toms & Percussion:** Pitched decaying resonators (Hi-Tom, Mid-Tom, Low-Tom, Hand Clap, Rimshot).

### 3.4 Bowed Strings (Violin & Cello)
- **Tuning Bases:**
  - Violin: G3 (196.0 Hz), D4 (293.66 Hz), A4 (440.0 Hz), E5 (659.25 Hz)
  - Cello: C2 (65.41 Hz), G2 (98.0 Hz), D3 (146.83 Hz), A3 (220.0 Hz)
- **Bowing Interaction:** Pointer move over string canvas continuously modulates filter frequency, drive gain, and resonance based on finger velocity and position along the string length.

### 3.5 Concert Harp
- **Scale Arrays:**
  - Major, Natural Minor, Pentatonic, Blues, Dorian.
- **Pluck Resonance:** High-Q bell bandpass filter combined with soft exponential decay.

### 3.6 Sub-Bass & Marimba
- **Sub Engine:** Pure sine wave sub-octave fundamental + subtle overdrive wave-shaper.
- **Marimba Engine:** High-transient wooden mallet attack impulse with rapid fundamental decay.

### 3.7 XY Synthesizer Pad
- **Axis Mapping:**
  - Horizontal ($X$): Biquad filter cutoff frequency (80 Hz to 8000 Hz, logarithmic scale).
  - Vertical ($Y$): Resonance ($Q$-factor, 0.5 to 18.0) and oscillator detune spread.

---

## 4. Song Academy & Note Trainer Engine

### 4.1 State Machine
```
[Trainer Disabled (Default)]
       │
       ▼ (User clicks 'SONGS' / Toggle Bar)
[Trainer Open / Active]
       ├── Mode 1: Practice Mode (Active: Wait for matching note from any instrument)
       │       │
       │       ├── On Match: Advance step -> highlight next target
       │       └── Mismatch: Non-blocking, keep waiting
       │
       └── Mode 2: Auto-Play Mode (Sequenced timer plays active instrument voices automatically)
```

### 4.2 Guard Predicate
All target indicator renders across Piano, Guitar, Bowed Strings, and Harp check:
```javascript
function isSongTrainerActive() {
    const bar = document.getElementById('song-tutorial-bar');
    return Boolean(bar && !bar.classList.contains('hidden') && (tutorialPracticeActive || tutorialAutoPlayActive));
}
```
When `isSongTrainerActive()` evaluates to `false`, zero target notes or pulses appear on any instrument canvas or DOM element.

### 4.3 Song Repository Schema
Each song in `TUTORIAL_SONGS` conforms to:
```javascript
{
    title: String,
    bpm: Number,
    notes: Array<{
        note: String,    // e.g. 'C4', 'G4', 'F#4'
        dur: Number,     // Duration in musical beats
        lyric?: String   // Lyric or step label
    }>
}
```
Built-in repertoire:
1. `birthday` — Happy Birthday to You
2. `christmas` — Jingle Bells
3. `harrypotter` — Hedwig's Theme
4. `twinkle` — Twinkle Twinkle Little Star
5. `odetojoy` — Beethoven's Ode to Joy

---

## 5. Sequencer & Loop Engine

- **Lookahead Scheduling:** Web Audio clock-synchronized scheduler utilizing recursive `setTimeout` with small lookahead intervals to eliminate JavaScript main-thread timing jitter.
- **Matrix Configuration:** 16-step grid supporting polyphonic drum tracks and melodic patterns.
- **BPM & Swing:** Variable tempo control (60 to 180 BPM) bound dynamically to CSS custom variables (`--val`, `--fill-color`).

---

## 6. Progressive Web App (PWA) & Offline Lifecycle

- **Service Worker (`sw.js`):** Stale-while-revalidate caching model for app shell (`index.html`, `manifest.json`, local icons, and CDN bundles).
- **Standalone Banner Detection:** Checks `display-mode: standalone`, `display-mode: fullscreen`, and `window.navigator.standalone` to suppress desktop installation prompts when already running as an installed application.
- **Orientation Control:** Automatic invocation of `screen.orientation.lock('landscape')` on supported mobile platforms with visual fallback badge when held in portrait mode.

---

## 7. Extension Recipes

### Adding a New Song to the Trainer
1. Open `index.html` and locate `const TUTORIAL_SONGS = { ... }`.
2. Add a new song key with `title`, `bpm`, and sequential `notes` array using standard note notation (`'C4'`, `'D4'`, `'D#4'`).
3. Add a selector pill button in `#song-tutorial-bar` calling `selectTutorialSong('newkey')`.

### Adding a New Instrument Channel
1. Instantiate dedicated `GainNode` connected to `masterGain`.
2. Add toggle flag to `rackState` object (`rackState.myinstrument = true`).
3. Provide pointer/touch event handlers calling procedural synth oscillators.
4. Clean up active voices during rapid touch releases.
