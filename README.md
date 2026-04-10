# overcast, for your past

A meditative browser experience where sound, memory, and a hand-drawn sky meet. Using Tone.js, the Web Audio API, and a single canvas, the piece generates a living skyscape and a unique piece of music shaped by who you are and when you arrive — your timezone, the season, the time of day, and how you move your mouse.

How it works
The project is a self-contained `index.html` that layers several real-time signals into a single experience:

The **sky** is a seeded, parallax-rendered scene of clouds, birds, particles, and a sun — all drawn in a crayon-like aesthetic. Its mood, palette, and cloud density shift with the season and time of day.

The **music** is generated live by Tone.js. Your timezone offset sets the root note, your mouse speed sets the BPM, and the season/time-of-day nudge the harmonic mode (major or minor). Three independent layers — a melodic voice, a plucked arpeggio, and a textural pad — fade in gradually after you click to begin.

Your **microphone** feeds back into the scene: ambient room sound widens the reverb and brightens the sky particles. Your voice is also transcribed live via the Web Speech API, appearing as ephemeral floating captions.

Your **webcam** (optional) estimates brightness, motion, and a rough silhouette focus point. When music swells, clouds gently sketch toward wherever you are in frame.

Everything you **draw or write** leaves a mark. Crayon strokes, typed notes, and sent doodles all pin themselves as softly rotating cards drifting across the sky, and each note subtly reshapes the generated music theme — as if sound can travel backward.

Setup
No build step is required. Open `index.html` directly in a browser.

- **Chrome or Edge** are recommended for full support of the Web Audio API, Web Speech API (`SpeechRecognition`), and `getUserMedia`.
- If you open the file locally via the filesystem (`file://`), the Speech API and some media permissions may be blocked. Running through **localhost** (e.g., the Live Server extension in VS Code) is strongly recommended.

Playing
**Click anywhere** to begin. The music and visual world fade in gradually.

**Drawing** — A toolbox appears in the bottom-left corner once you start. Pick a color swatch, adjust the brush size slider, and paint directly on the sky. Use the text tool to drop a typed note anywhere on the canvas. Hit "send doodle" to pin your drawing as a floating card.

**Writing notes** — Click the "notes ✦" chip to open the notes drawer. Write anything — a thought, a memory, a message to yourself. Click "send" to release it into the sky. Each card you send shifts the music's mood and reshapes the generated theme.

**Mouse height** acts as a filter: moving your mouse toward the top of the screen opens the sound up; moving it down muffles it.

Code walkthrough

### Seeding and ambient signals

Before anything is drawn or played, the script collects a set of read-only signals from the browser environment: `HOUR`, `SEASON`, `TZ_OFFSET` (your timezone offset in minutes), and `mouseSpeed`. Mouse distance is accumulated for the first five seconds after page load and then bucketed into `'still'`, `'slow'`, `'moderate'`, or `'fast'`. A `SESSION_SEED` is also generated using `crypto.getRandomValues` so that even two visits at the exact same time and timezone feel slightly different.

`getSeedNum()` combines all of these into a single 32-bit integer, which is then passed to `mulberry32()` — a fast, seedable pseudo-random number generator (PRNG). Every call to the returned function produces the next number in a deterministic sequence. Because the seed is the same for a given hour + season + timezone + mouse-speed combination, the scene and music are reproducible: opening the page twice in the same minute will produce the same sky layout and the same root note.

### Palette system

`PALETTES` is a nested object mapping `timeOfDay → season → { sky, ground, dark, cloud, particle }`. Each entry contains hex color strings for the sky gradient stops and RGB arrays for particles and clouds. `getTimeOfDay(hour)` maps the current hour to `'morning'`, `'afternoon'`, `'evening'`, or `'night'`.

`FORCE_DAY = true` overrides the palette's sky colors with `BLUE_SKY` — a fixed set of soft blue gradient stops — regardless of the actual time. The cloud and particle colors still inherit their seasonal tint so the scene reads as a bright day with a character that shifts by season.

The `chalky(rgb, amt)` helper blends any color toward a warm off-white (`[235, 235, 235]`) by `amt`. It's called everywhere fills are drawn to add a slightly chalky, washed-out quality consistent with the crayon aesthetic.

### Scene initialization (`initScene`)

`initScene()` seeds a fresh PRNG from `getSeedNum()` and uses it to build every element in the scene.

**Clouds** — Each cloud is an 18-point irregular polygon. For each point, a random radial displacement (`bump`) is multiplied against the cloud's half-width and half-height to create a lumpy outline. Two additional jitter offsets (`wX`, `wY`) are baked in at init time and stay fixed, giving each cloud a stable hand-drawn shape across frames. A `z` value between 0 (near) and 1 (far) controls both rendering size and drift speed via `project()`. Most clouds are assigned far `z` values (using `Math.pow(rand(), 1.75)` to bias toward 1), with a couple of forced near clouds to make the parallax obvious.

**Particles** — Small dots (pollen, dust, light motes) that drift upward. Each has its own vertical velocity `vy`, a horizontal sine wave amplitude `ax`, a frequency `fx`, and a base opacity `op`. Near particles are faster and brighter than far ones.

**Birds** — Simple parallax "V" shapes. Each has a `z` depth, a horizontal speed `sp`, and a flapping phase `ph`. They drift with the wind vector and wrap at the canvas edges.

**Focal element** — One special element chosen from `['bird', 'sun', 'constellation']` based on `Math.floor(Math.abs(TZ_OFFSET) / 60) % 3`. This is more prominent than the background birds and gives each timezone a consistent character. A constellation is a cluster of 6–14 small dots connected by a fixed layout; the focal sun is a separate wobbling disc on top of the ambient sun.

**Wind** — A static `{x, y}` vector seeded once per scene. Clouds, birds, and particles all reference it for directional drift.

**Camera pan** — `view.tx` and `view.ty` are updated every frame in the animation loop based on mouse position and a slow sine-wave sway, giving the feeling of looking around from a fixed reclining position.

### Projection (`project`)

`project(x, y, z)` converts 3D scene coordinates into 2D screen coordinates using a simplified perspective formula. Objects with `z = 0` are "near" (drawn at full size, positioned close to screen edges); `z = 1` is "far" (drawn small, slow, near the vanishing point). The formula stretches depth with a `1.55` multiplier to make parallax feel exaggerated and cinematic. The returned scale factor `sc` is applied to both rendered radius and stroke widths, so far objects are genuinely smaller in every dimension.

### Drawing pipeline

Each animation frame calls these functions in order:

**`drawBackground()`** — Fills the canvas with a three-stop sky gradient tinted by `soundLevel`, `mood.val`, and `contribution.val`. Then overlays 32 short, semi-transparent horizontal strokes at random y-positions to simulate crayon texture. The strokes are redrawn every frame with fresh random positions, so the texture shimmers slightly.

**`drawSun()`** — Renders a radial gradient halo plus a 26-point wobbly disc. Both radius and opacity pulse on a slow sine based on `soundLevel`. The random per-vertex jitter on the disc outline (`(Math.random()-0.5)*1.6`) is re-sampled every frame, giving the sun a continuously trembling hand-drawn edge.

**`drawCloud(c, t)`** — Three-pass rendering. Pass 1: solid chalky fill using the 18-point polygon. Pass 2: 12 short random strokes radiating from the cloud's center to simulate the uneven pressure of a crayon. Pass 3: a wobbly polygon outline where each vertex receives a fresh random jitter each frame, so the outline shakes as if traced by an unsteady hand. Cloud opacity and outline weight increase for near (`z` close to 0) clouds.

**`drawBirds(t)`** — Sorts birds far-to-near for correct depth ordering, then draws each as a two-segment quadratic bezier (left wing, right wing). The control point's y-offset is driven by a sine of time to produce a flapping motion. Near birds are larger, more opaque, and flap more dramatically.

**`drawCloudSketchToFocus(t)`** — Only runs when `soundLevel > 0.12`. Picks random clouds and draws faint quadratic bezier strokes from their projected position toward the camera-derived focus point (`camFocusX`, `camFocusY`). The number of strokes and their opacity scale with `soundLevel` and `camFocusConf`, so this effect is only visible when music is loud and the camera can see you.

**`drawParticles(t)`** — Moves each particle upward by `vy * (1 + soundLevel * 1.25)` and applies a horizontal sine drift. Particles that leave the top of the canvas are respawned at the bottom. Each dot is drawn as a `ctx.arc` with a tiny random position and radius jitter to make it feel alive. `micLevel` adds additional brightness to particle opacity.

**`drawMusicInk(t)`** — Draws 6–40 short, roughly-horizontal drifting strokes (count scales with `soundLevel`). These represent "sound writing the sky" — visible marks left by the music. Stroke length, width, and a time-based wobble offset all scale with `soundLevel`.

**`drawBreathingFrame(t)`** — Composites two dark overlays: a radial gradient darkening the corners (vignette), and a linear gradient darkening the bottom third of the frame. Both deepen slightly with `soundLevel`, giving the sensation of the world contracting and breathing with the music.

**`drawVignette()`** — A secondary radial gradient centered on the canvas, lightly darkening the edges at all times.

**`drawEyelids()`** — The "eyes opening" effect at the start. Two black rectangles (top and bottom) shrink away from the center as `blinkOpen` increases from 0 to 1. The gap between them is `H * (0.62 + blinkOpen * 0.36)`. Natural blinks are triggered on a timer (~16–26 seconds) and animated as a quick triangle-envelope dip in `blinkOpen` lasting about 16 frames.

### Blink and reveal system

`updateBlinkAndReveal(t)` runs every frame. `revealLevel` is a 0–1 value that controls the global canvas opacity — the world starts invisible and fades in as `soundLevel` rises above a threshold, smoothed with a simple exponential filter (`revealLevel * 0.94 + target * 0.06`). `blinkOpen` drives the eyelid geometry and also contributes to reveal. Once your `contribution` reaches 5 (meaning you've submitted 5 notes or doodles), the eyelids fully lock open and `revealLevel` clamps to 0.98 — the world stays bright in response to your engagement.

### Animation loop

`loop(t)` is called by `requestAnimationFrame` with a DOMHighResTimeStamp. It computes a smoothed `soundLevel` by blending `musicLevel` and `micLevel`, then updates the camera pan (`view.tx`, `view.ty`) using mouse position and a slow sine wave. All drawing functions receive `t` so they can animate continuously without accumulating state in variables.

### Audio architecture

`startAudio()` is triggered on the first click. It initializes a shared effects chain:

```
each layer → Tone.Reverb → Tone.Filter → Tone.EQ3 → Tone.Limiter → Tone.Volume → destination
```

A `Tone.Meter` taps off the limiter to measure output level, which drives `musicLevel`.

**Root note** — `getRootNote()` maps `TZ_OFFSET` to one of 12 chromatic notes. UTC maps to G4; offsets step up 7 semitones per hour of timezone west of UTC (PST → C4). This means every timezone consistently gets a distinct tonal center.

**Scale** — `buildScale(rootNote, useMinor)` generates a pentatonic scale (major or minor) from the root by transposing a fixed set of semitone intervals (`[0,2,4,7,9,12,14,16]` for major) using `Tone.Frequency().toMidi()`.

**BPM** — Set from `mouseSpeed`: still → 52, slow → 62, moderate → 74, fast → 86.

**Melody layer** — A `Tone.Synth` (sine or triangle oscillator) passes through a `Tone.Chorus` and a volume node. `melSeq` is a `Tone.Sequence` running 8th-note steps. Each step calls `pickMelNote()`, which mostly draws the next note from the recurring `motifNotes` array, occasionally substituting a nearby chord tone (22% chance) or a random scale passing tone (12% chance). Leap size is limited to a seventh by shifting by octaves.

**Arpeggio layer** — A `Tone.PluckSynth` runs a 16-step 16th-note sequence picking random notes from the previous bar's chord. It adds bright metallic punctuation without rhythmic predictability.

**Rhythm layer** — A `Tone.NoiseSynth` with a very short pink-noise envelope runs a 16-step pattern. `mode.rhythmFill` (0.18–0.26 depending on mode) controls the probability that each step is active, producing a sparse, rain-on-glass texture rather than a recognizable beat.

**Pad layer** — A `Tone.PolySynth` with a long attack (1.8s) and release (2.8s) plays full chords from the current chord progression once per measure via a `Tone.Loop`. It fades in last (22 seconds after click) for a gradual swell.

All layers stagger their fade-in times to build from silence: melody arrives at 6+ seconds, rhythm at 9+ seconds, and pad at 22+ seconds. The exact base delay grows with how long the user waited before clicking, rewarding patience with a longer silence.

### Motif and theme mutations

A 5–6 note `motifNotes` array is generated at startup using the PRNG: a starting scale index is picked, then each subsequent note steps ±1 or ±2 scale degrees with some probability weighting toward stepwise motion. This motif is the recognizable melodic thread that repeats throughout.

`themeEngine.applyNote(text)` is called each time you submit a note or send a doodle. It passes the text through `xmur3()` — a simple string-hashing function — to produce a deterministic seed integer. `reseedTheme(seed, strength)` then:

1. XORs the new seed into the running `themeSeed` so changes are cumulative
2. Picks a new chord progression variant (e.g., I–IV–V–vi instead of I–V–vi–IV)
3. Rebuilds `motifNotes` in-place with a new contour, optionally transposing the whole motif by ±2 semitones for a "new chapter" feel
4. Ramps BPM by up to ±5 bpm, shifts the filter cutoffs, adjusts reverb wet and decay
5. Randomly changes the melody oscillator type (`sine`, `triangle`, `sawtooth`) and the chorus depth and rate
6. Re-randomizes the rhythm pattern density
7. Triggers a brief melody dip-and-swell to mark the moment of change

The effect is that the music accumulates character as you write — each note you leave behind nudges the sound in a direction that can't be undone.

### Microphone (`setupMic`)

Creates a Web Audio `MediaStreamSource` connected to an `AnalyserNode` inside Tone's shared `AudioContext`. Each animation frame, `getByteFrequencyData` is sampled and averaged to produce `micLevel` (0–1). A higher `micLevel` pushes `reverbNode.wet.value` toward 0.62 (from a base of 0.34), making the space feel larger and more echoing when the room is loud. It also additively brightens particle opacity in `drawParticles`.

### Camera (`setupCamera`)

Requests `getUserMedia({ video: true })` and draws frames to a 96×72 off-screen canvas each animation tick. Three values are derived per frame:

- **`camLevel`** — average luma across all pixels. Measures how bright the room is.
- **`camMotion`** — average absolute difference in luma versus the previous frame. Measures movement.
- **`camFocusX/Y`** — a weighted centroid of pixels that are either dark (likely hair/face) or moving. Used as an approximate gaze/presence point.

All three are smoothed with exponential filters (`* 0.90 + new * 0.10`) to prevent jitter. `camFocusConf` is a confidence value that rises when the weighted centroid has enough mass; it gates how strongly `drawCloudSketchToFocus` pulls the sketch lines toward you.

### Speech-to-text

`startSpeechToText()` creates a `webkitSpeechRecognition` (or `SpeechRecognition`) instance set to `continuous = true` and `interimResults = true`. Each `onresult` event pushes the current interim or final transcript as a `<div class="cap">` in the captions container. Captions fade out after 5.2 seconds. The recognizer auto-restarts via `onend` as long as `speechActive` is true. There is no error surfacing — failures are silently swallowed so the absence of the feature feels seamless.

### Paint layer (`initPaint`)

A second off-screen `<canvas>` (`paintC`) is created once and composited on top of the scene each frame with `ctx.drawImage(paintC, 0, 0)`. Strokes are stored as `{ color, size, eraser, pts: [{x,y}...] }` objects. `renderPaint()` replays all strokes from scratch each frame using `lineCap: 'round'` and `lineJoin: 'round'`. Eraser strokes use `globalCompositeOperation: 'destination-out'` to punch holes in the paint layer rather than drawing opaque white.

The text tool (`paintTextMode`) spawns a floating `<textarea>` at the clicked position with inline styles matching the design. When the textarea is committed (via Ctrl/Cmd+Enter or blur), its text is passed to `pinScrawl()` and `floatCard()`.

The "send doodle" button calls `paintC.toDataURL('image/png')` to snapshot the paint canvas and passes the data URL to `pinDoodle()`, which creates a `<div class="scrawl">` containing an `<img>` at a random screen position. Scrawls and float-cards are capped at 7 visible at once and fade out after 24 seconds.

### Mood and contribution

Two running values shape the visual atmosphere:

**`mood`** (`-1` to `+1`) — `nudgeMood(x)` shifts `mood.target` by `x`. It's called when you start a stroke (`+0.05`), pick a dark swatch (`-0.08`), or pick a light swatch (`+0.03`). Negative mood adds a blue-grey overlay in `drawBackground()` proportional to `Math.min(0.16, (-mood.val) * 0.12)`; positive mood makes the sky gradient slightly brighter and warmer.

**`contribution`** (0 to 40) — `bumpContribution(n)` adds to `contribution.target`. It's bumped by 2 each time you send a note, send a doodle, or commit a text box. `contribution.val` is smoothed toward the target each frame. In `drawBackground()`, the sky lightens slightly as `contribution.val` rises (`rgba(255,255,255, contribution * 0.055)`), giving visible feedback that your engagement is brightening the world. Once it reaches 5, the eyelids lock open permanently.

Tweaking
- **BPM** — Change the values in `{ still:52, slow:62, moderate:74, fast:86 }` to adjust tempo per mouse-speed tier.
- **Reverb** — `decay: 5.5` in the `Tone.Reverb` constructor controls space size. Lower it for a drier, closer feel.
- **Cloud density** — Change `nClouds` values in `{ spring:9, summer:8, autumn:10, winter:11 }` to add or remove clouds.
- **Root note logic** — Edit `getRootNote()` to use a fixed note if you prefer a consistent tonal center.
- **Motif length** — `motifLen = 5 + Math.floor(aRand() * 2)` controls how many notes are in the recurring theme.
- **Note cards** — The `while (scrawlsEl.children.length > 7)` line caps floating scrawls at 7. Raise it to allow more.

Troubleshooting
**No sound** — Click directly on the page. Browsers require a user gesture to start an AudioContext. If audio still doesn't start, check that the tab is not muted.

**Speech captions don't appear** — The Web Speech API is only available in Chrome and Edge. Ensure you are serving from `localhost` and that microphone permissions have been granted.

**Webcam focus effects don't appear** — Camera access is optional and silently skipped if denied. Grant camera permission in your browser settings and reload.

**Music sounds different each time** — That's intentional. The seed is derived from your timezone, season, time of day, and mouse behavior, so the piece varies by when and how you arrive.

Credits
- [Tone.js](https://tonejs.github.io/) — Web Audio synthesis and scheduling
- Web Audio API — Microphone analysis
- Web Speech API — Live speech transcription
- MediaDevices API — Webcam access
- [Google Fonts](https://fonts.google.com/) — "Lora" typeface for the handwritten aesthetic
