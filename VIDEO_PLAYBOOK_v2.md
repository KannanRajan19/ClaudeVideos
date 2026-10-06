# Launch-video playbook: app screenshots → motion-graphics product video

You are building an internal product-launch video from 8–12 screenshots of a web app. The target: 1080p, 30 fps, about 75 s, a dark cinematic look, tilted 3D screenshot cards, UI elements lifting off the page, transitions that grow out of the button that was clicked, and an original soundtrack locked to the cuts.

Every number in this document comes from a working, reviewed renderer. Use the numbers as given. Do not substitute "reasonable defaults".

**Starting point:**
- If `launchvideo/engine.py` and `launchvideo/music.py` exist, reuse them. Rewrite only `launchvideo/scenes.py`, `config/screens.json` and `config/storyboard.json` for the new app, and follow §14.
- Otherwise, build the engine from §6–§13, starting from the skeleton in Appendix A.
- If you are running in a hosted sandbox with time limits, write the code and render single frames and short ranges (`--start/--end`). Tell the user to run the full render (10–20 min) on their own machine.

**Why code and not an AI video generator:** text-to-video models redraw the UI, so labels, numbers and layouts come out invented or warped. This video must show the real screens pixel-for-pixel and be re-renderable when the app changes.

---

## 1. Quality bar: what "this quality" means

The finished video must meet all of the following:

1. **Real screens only.** Every UI pixel comes from a supplied screenshot. Never redraw, retype or AI-generate UI. Overlays (callouts, outlines, cursor, typed text) sit on top.
2. **One idea per moment.** At any instant exactly one thing is the focus: a lifted-out element, a headline, or a click. Everything else is dimmed or still.
3. **Motion feels physical.** Expo-out entrances, slight overshoot (back easing) on pops, quintic cursor paths, nothing linear except the slow background drift.
4. **Transitions come from the UI.** The clicked button, icon or tab grows into the next scene. Avoid generic crossfades except at the outro.
5. **Typography is consistent.** One sans family (Inter) for everything, plus one display serif for the product wordmark. Fixed sizes and positions (§6).
6. **Sound is locked to picture.** The music bar grid lands on scene changes, and every pop, click and whoosh has a sound effect from the same cue sheet as the animation.
7. **Text is crisp.** Small table text in screenshots stays readable at 1080p. Screenshots are never upscaled more than about 1.3×.
8. **It is deterministic.** Same inputs give the same video. Everything is in version control, and nothing depends on fonts installed on the machine.

---

## 2. The approach (non-negotiable)

- **Language and stack:** Python 3.10+ with `numpy`, `opencv-python`, `pillow`, `scipy` and `imageio-ffmpeg` (a pip-installed ffmpeg, so no admin rights are needed).
- **Architecture:** a pure function `render(t) -> uint8[1080,1920,3]` for any time `t` in seconds. Frames are produced in a process pool and piped as raw RGB into ffmpeg. There is no timeline state, so any frame can be rendered alone for review.
- **Separate content from code.**
  - `config/screens.json`: one entry per screenshot: file name, a short description of the expected framing (`capture`), and **named regions as fractions (0–1) of the image**: `[x0, y0, x1, y1]`.
  - `config/storyboard.json`: every word that appears on screen.
  - `scenes.py`: timing and choreography only, referring to regions by name.
- **Audio is synthesized in code** (numpy and scipy) from the same cue sheet as the animation, so it has no licensing issues and stays in sync automatically.
- **Do not use:**
  - moviepy `TextClip` or ImageMagick text, which looks cheap.
  - Browser screen recordings, which drop frames.
  - PowerPoint exports.
  - Stock templates.
  - Text-to-video models.
  - System fonts. Bundle the font files (TTF) in the repo.
- **Do not switch to Remotion** (React video) unless the user asks and confirms the company has a license; it needs a paid license above a small team size.

---

## 3. Inputs: collect these before building (ask if missing)

1. **Screenshots:** 8–12 screens that pass the input checks in §4.
2. **Product name** and how to set it as a wordmark (e.g. **ACME** in a heavy serif + **Flow** in a light sans), plus a subtitle (e.g. "PROJECT TRACKER").
3. **Features in story order.** One per screen, each with:
   - a two-line headline (≤ 16 characters per line);
   - a two-line sub (≤ 34 characters per line);
   - 1–3 callout labels (≤ 40 characters).
4. **The flow between screens:** which element the user clicks to get from one screen to the next (an Open button, a history icon, a comment icon, a tab). These drive the transitions.
5. **Audience and channel:** internal email or Teams (16:9, ≤ 18 MB copy), or a presentation (master file).
6. **End-card line:** e.g. "Coming soon", or "Now live" only if it really is live. Also a credit line.
7. **Sensitive-data policy:** blur names, email addresses or IDs? If yes, also run an OCR check on the final frames (§15).
8. **Length:** default 75–80 s. A teaser is about 45 s: intro, 4 features, outro.

---

## 4. Screenshot inputs and regions

### Input requirements (validate these; don't explain capture tools)

Check every screenshot before building. If one fails, stop and tell the user which file, what's wrong, and what to re-capture (e.g. "help-panel.png is 439 px wide; re-capture the panel at 200% browser zoom"). Do not try to compensate by enlarging.

| Check | Pass | Why |
|---|---|---|
| Format | PNG (lossless) | JPEG artefacts ring around small text |
| Full-page width | ≥ 1600 px (ideal ≥ 1900) | Cards show pages about 1000–1220 px wide plus 1.5–2.5× lift-outs |
| Dialog or panel width | ≥ 1000 px (ideal ≥ 1200) | Dialogs are shown about 560–1000 px wide and lifted out larger |
| Lift-out enlargement | ≤ 1.3× source pixels | Above this, text goes soft (§13) |
| Framing | App content only: no browser tabs, address bar, taskbar, desktop or monitor bezel | Chrome around the UI breaks the card look |
| Consistency | Screens that share a layout have identical framing | Their regions then line up across screens |
| State | The state each chapter needs is visible (dialog open, tooltip shown, key row in view) | Nothing can be added that isn't in the image |
| Not a photo of a monitor | No moiré, skew or glare | Use the fallback below only if the user can't provide real screenshots |

### Fallback: the inputs are phone photos of a monitor (results are softer; tell the user)

1. **Rectify.** Fit the four screen edges (Sobel gradients plus line regression, or align to known text columns), then `cv2.warpPerspective` to a flat rectangle. Crop inside the bezel.
2. **Flat-field the glare.** In LAB colour space, estimate the background L with a morphological close (35 px ellipse) plus a Gaussian blur (σ 9). Divide L by it, scaled to 246, only where the background is light.
3. **Kill moiré colour.** Zero the a/b channels except on dark, saturated UI (buttons, status dots) and on intentional red text.
4. **Clean up.** `cv2.fastNlMeansDenoisingColored(img, None, 5, 3, 7, 21)`, then unsharp: `1.6·img − 0.6·blur(σ1.6)`.

### Regions

For each screen, define named regions (fractions of width and height) for everything the choreography touches:

- **Lift-outs:** filter chips, key columns, a toolbar, a dialog.
- **Click targets:** a button or an icon.
- **Sequences:** `row_1…row_6`, `tab_1…tab_6`, `chip_1…chip_9`.
- **Content to reveal:** `message_1…3`.
- **Anchor points** for callouts.

**Never hard-code pixel coordinates in scene code.** Every position the choreography uses must be a named region in `config/screens.json`.

### `config/screens.json` format

```json
{
  "screens": {
    "tasks": {
      "file": "screens/tasks.png",
      "capture": "Task list page: app content only, from the top banner to the bottom of the table.",
      "regions": {
        "status_filters": [0.759, 0.159, 0.987, 0.203],
        "go_button":      [0.864, 0.824, 0.892, 0.858]
      },
      "region_meta": {
        "status_filters": {"role": "liftout", "help": "The status filter chips (All / Open / Closed …)", "target_width": 760, "label_key": "status_filters"},
        "go_button":      {"role": "click",   "help": "One Open/Go button; it grows into the next scene"}
      }
    }
  }
}
```

- **`regions`:** `[x0, y0, x1, y1]` as fractions of the image (0 = left/top, 1 = right/bottom). They survive re-captures at a different resolution.
- **`region_meta`:** optional, but required for the editor below.
  - `role` is one of `liftout | click | sequence | reveal | anchor | sample`.
  - `help` is one line saying what to draw the box around.
  - `target_width` is the on-screen width in px the scene gives a lift-out; the editor uses it to warn about upscaling.
  - `label_key` links the region to its callout text in `storyboard.json` (`<screen>.labels.<label_key>`).
- **`storyboard.json`** holds all text: per screen `chapter`, `headline` (two lines), `sub` (two lines) and `labels` (callouts keyed by region name).

### `check` command

`python -m launchvideo check` draws every region, labelled and colour-coded by role, over its screenshot and writes `output/check/<screen>.png` (max 1800 px wide). For each screen it also prints the source size and a LOW-RES warning if a lift-out would be enlarged more than 1.3×.

### Region and annotation editor: `tools/annotate.html` (required deliverable)

The editor is how the user lines up highlights and edits callout text on their own screenshots without touching code. Build it as **one self-contained HTML file**: inline CSS and JS, no libraries, no network, no uploads. It must work when opened from disk (`file://`) in Chrome or Edge.

**Loading**
- Two file pickers: (1) `screens.json` and `storyboard.json` (accept both at once), (2) one or more screenshot images.
- Match images to screens by file name (case-insensitive) against `screens.<key>.file`.
- A screen selector lists every screen with a status dot: green (image loaded, all required regions inside the image), amber (warnings), red (missing image or region).

**Canvas**
- Show the screenshot with every region drawn: semi-transparent fill plus a 1.5 px outline, colour by role (`liftout` mint, `click` amber, `sequence` blue, `reveal` violet, `anchor` red, `sample` grey), and the name tag above the top-left corner.
- The selected region gets a 3 px outline and 8 resize handles (corners and edges).
- Zoom: Fit / 100% / 200%, and Ctrl+wheel. Scrolling pans when zoomed.
- Show the expected framing (`capture`) above the canvas.

**Editing regions**
- Click a region on the canvas or in the side list to select it. The list shows name, role, `help` text and the four fractions (3 decimals).
- Drag inside the box to move it, drag a handle to resize, and drag on empty canvas to redraw the selected region.
- Arrow keys nudge by 1 image px; Shift+arrows by 10 px. Ctrl+Z / Ctrl+Y for undo/redo (keep at least 50 steps).
- Clamp to [0, 1], store 5 decimals, and keep x0 < x1 and y0 < y1.
- Overlays use the canvas pointer position relative to the displayed image. Hidden placeholder or empty-state elements must use `display:none` so they never sit over the canvas and swallow mouse events.

**Annotations (text)**
- For the selected screen, show editable fields for `chapter`, the two `headline` lines, the two `sub` lines, and the callout `labels` of every region with a `label_key`.
- Live character counters with limits: headline ≤ 16 per line, sub ≤ 34, label ≤ 40. Turn red when over.
- Selecting a region with a `label_key` focuses its label field.

**Previews and warnings** (the quality guard)
- **Lift-out preview:** for a `liftout` region, show its crop at the size it will appear in the video (`target_width`) and print the scale factor. Warn amber above 1.3× and red above 1.6×, with the fix: "re-capture this screen at a higher resolution (e.g. 200% browser zoom)".
- **Callout preview:** draw the callout pill (PANEL fill, MINT stroke, mint dot, Inter 600 at 24 px, or the system sans as a stand-in) under the lift-out preview, with the current label text, so overflow is visible.
- **Validation list** per screen:
  - regions outside the image;
  - zero-size regions;
  - regions that `help` says should be inside another region but aren't (e.g. `chip_*` inside `filter_chips`);
  - sequence items out of left-to-right or top-to-bottom order;
  - missing images.

**Saving**
- Buttons: **Download screens.json** and **Download storyboard.json**. Pretty-print with 2-space indent, keep key order, and add a trailing newline.
- The page tells the user to replace the files in `config/` and re-run `check`.
- Never write anything except those two downloads.

**Acceptance test (run it yourself before handing over)**
Drive the page headlessly with Playwright or Chromium:
1. Load the two JSON files and one screenshot.
2. Select a region, redraw it by dragging, nudge it with Shift+Arrow, and resize it with a handle.
3. Edit one label.
4. Download both files.

Assert that only the edited region and label changed and every other value is byte-identical. Also take a screenshot of the editor and look at it.

---

## 5. Storyboard

### Structure (about 78 s)

| Time (s) | Scene | Content |
|---|---|---|
| 0–5 | **Intro** | A dot pops, becomes a mint pill, then morphs into the dark logo card. Wordmark rises in. "Introducing" above, tagline below. Card flies up and out. |
| 5–11 | **Value proposition** | A big two-line statement and one visual idea from the product, e.g. a row of pills for the teams or stages it covers, each ticking to done, with a counter ("4/4 COMPLETE"). The last pill grows into scene 1. |
| 10–70 | **Feature chapters** (8–10 of them, 5–8 s each) | Numbered chapters "01 <FEATURE>" … "10 <FEATURE>". Each follows the chapter grammar below. Scenes overlap by 0.5–0.8 s for transitions. |
| 69–73 | **Recap** | A mosaic of all screens. Four one-word beats summarising the product (e.g. "Plan. Track. Review. Ship.") land on the music. The mosaic collapses to the centre. |
| 73–78 | **End card** | Logo card, end line ("Coming soon"), credit line (e.g. "BUILT BY THE <TEAM NAME>"), fade to black. |

### Chapter grammar (one feature, 5–8 s)

```
t0        card enters (tilted, 1.0 s expo-out) + chapter label + headline words + sub lines
t0+1.2    lift-out #1 (0.55 s in, ~1.0 s hold with callout pill, 0.4 s out); rest of card dims
t0+3.1    lift-out #2 / sequence highlight (rows, tabs, chips stepping 0.16–0.30 s apart)
t0+5.0    cursor moves to the next target (0.6–1.0 s quintic), hover ring, click ring + click SFX
t_end     transition grows out of the clicked element (0.6 s)
```

### Copy rules

- **Headlines:** two lines; line 2 is mint. Pattern: *claim / payoff* ("Every request." / "One queue.").
- **Subs:** plain facts, no adjectives. Only use numbers the screenshots show.
- **Callouts:** state what the element does ("Filter by status"), not marketing.
- **Never claim status that isn't true.** "Now live" only if it really is live.
- **No PII in overlays:** no record numbers, IDs, customer data or people's names in your own labels.

### Reference timeline (copy the rhythm, not the content)

| Scene | Start–end (s) | Transition in |
|---|---|---|
| Intro | 0.0–5.0 | – |
| Value proposition | 5.0–11.0 | – |
| 01 | 10.2–18.8 | Last pill of the value scene grows (0.8 s, colour fill) |
| 02 | 18.2–23.6 | Circle iris from a clicked icon (0.6 s) |
| 03 | 23.0–31.3 | Clicked button grows (0.6 s, dark-mint fill) |
| 04 | 30.7–37.6 | Circle iris from a clicked icon (0.6 s) |
| 05 | 37.0–43.4 | Horizontal push, mint seam (0.6 s) |
| 06 | 42.8–46.8 | Clicked button grows (0.6 s) |
| 07 | 46.2–53.5 | Vertical push, mint seam (0.6 s) |
| 08 | 52.3–61.0 | Domain-shape block wipe (1.2 s) |
| 09 | 60.3–65.4 | Rounded rect grows from the focal area (0.7 s) |
| 10 | 64.7–69.8 | Clicked pill grows (0.7 s) |
| Recap + end card | 69.2–78.0 | Zoom-out crossfade (0.6 s) |

**Alternate sides.** Put the text on the left and the card on the right for most chapters. Every third chapter or so, flip to text on the right (x = 1290) and the card on the left.

---

## 6. Design system

### Canvas
1920×1080, 30 fps, sRGB. Work in float32, 0–1 linear sRGB values. No gamma maths is needed.

### Palette

| Token | Hex | Use |
|---|---|---|
| MINT | `#42DE9E` | Accent: headline line 2, outlines, callout dots, click rings, seams |
| MINT_D | `#1F5C4A` | Dark accent fills (pill-grow transitions, check dots) |
| WHITE | `#F7FAFA` | Headlines, callout text |
| SUB | `#9EB0B5` | Sub lines, chapter labels, kicker |
| INK | `#0F1417` | Text on light UI elements |
| RED | `#F2574F` | Flag highlights (an error, a failed check, a "No" value) |
| PANEL | `#13181B` | Callout pills, logo card |
| UI_GREEN | `#6BC499` | The app's own green (selected tab); match the product |
| BG top → bottom | `#0B0E10` → `#07090A` | Vertical gradient |

**Pick your accent from the product's own UI colour** (mint suits a green status UI). Keep one accent colour.

### Background (static base plus slow life)

- **Gradient:** vertical, from BG top to BG bottom.
- **Two soft colour glows:** a Gaussian falloff `exp(-2.2·d²)`.
  - At (22%, 25%), radius 900 px, colour (0.06, 0.25, 0.20) × 0.55.
  - At (85%, 85%), radius 1000 px, colour (0.05, 0.14, 0.18) × 0.5.
- **Dot grid:** 36 px pitch, dot radius about 1.6 px, intensity 0.035 × vignette.
- **Six floating outline shapes:**
  - Types and sizes: pills 260×64, 180×50 and 220×56; circles 90 and 44; a 300×170 rect with radius 18.
  - Stroke: MINT, 1.5 px, 6% opacity.
  - Motion: drift ±30 px in x and ±22 px in y (sinusoids 0.2–0.25 rad/s), and rise 4 px/s with wrap-around.
- **Finishing, on every frame:**
  - Vignette multiply `0.8 + 0.2·v`, where `v = clip(1 − 0.55·r², 0.3, 1)`.
  - Film grain σ 0.006 at half resolution.
  - Global fade-in 0.5 s and fade-out 0.6 s.
  - White flash on big hits: `0.18·e^(−9Δt)`.

### Typography (bundle the TTFs)
- **Inter** 200 / 400 / 600 / 700 / 800 and **Playfair Display** 900. Both are SIL Open Font License; include the license files.
- **Fonts on Windows:** from the npm packages `@fontsource/inter` and `@fontsource/playfair-display`, converting `.woff` to `.ttf` with `fontTools`.
- **Rendering:** draw text with Pillow into a cached alpha mask at full size, never scaled bitmaps. Apply tracking (letter spacing) per character.

| Element | Font | Size | Colour | Position |
|---|---|---|---|---|
| Chapter label | Mint capsule line 36×3 px, then the number (Inter 700, 20, tracking 2), then the label (Inter 600, 20, tracking 6) | 20 | MINT / SUB | x = 120 (or 1290), y = 330 |
| Headline | Inter 800 | 52–58 | Line 1 WHITE, line 2 MINT | y = 410, line pitch 70 |
| Sub lines | Inter 400 | 26 | SUB | 20 px below the headline, pitch 38 |
| Callout pill | Inter 600 | 24 | WHITE on PANEL | Height 48, side padding about 31 px |
| Wordmark | Playfair Display 900 at 150 + Inter 200 at 132 | – | WHITE | Centred in the logo card |
| Wordmark subtitle | Inter 600, tracking 20→9 | 21 | SUB | 78 px below the wordmark centre |
| Kicker / tagline | Inter 400 | 28 / 32 | SUB / WHITE | y = 330 / 780 |
| Recap words | Inter 800 | 76 | WHITE (last word MINT) | Centre |

---

## 7. Components (exact specs)

### 7.1 Tilted screenshot card (the main visual)

- **Perspective camera:** focal length 1700 px. The card is a plane, rotated `rx` (pitch), `ry` (yaw) and `rz` (roll) about its centre. Project the corners and `warpPerspective` the screenshot.
- **Size on screen:** 1000–1220 px wide. Centre at x ≈ 1250 when the text is on the left, ≈ 690 when the text is on the right; y ≈ 560.
- **Entry (1.0 s, expo-out):**
  - Rotate from about 3× the rest angles to rest, e.g. `rx 14→3`, `ry 28→8`, `rz −4→0`.
  - Rise 120–240 px.
  - Zoom from 1.25→1.0 or 0.92→1.0.
- **Lean:** the card leans **toward** the text. With the text on the left, the card's left edge sits farther back (`ry > 0` in this convention). Flip the sign when the text is on the right. Check it visually: a card angled away from its copy looks wrong.
- **During the scene:** slow push, zoom ×1.00→1.04–1.05 over the scene.
- **Look:**
  - Rounded corners, radius 16 px, applied in screenshot pixels as an alpha mask.
  - Premultiplied alpha throughout.
  - Soft drop shadow: blur the card's alpha (σ 30 px), shift it 22 px down, and multiply the background by `1 − 0.55·shadow`.

### 7.2 Lift-out: the signature move

- **What it is:** a region of the screenshot (named in `screens.json`) is copied **flat** (un-tilted) and flies from its projected position on the card to a larger target position. Make it 1.5–2.5× bigger, but **keep it from being upscaled more than ~1.3× relative to source pixels**; choose the target width from the source size.
- **In:** `ease_back(s=1.1)` over 0.55 s. **Hold:** about 1.0–1.4 s. **Out:** `ease_io` over 0.4 s back to the card.
- **While it is out:**
  - The card dims to 45% brightness (`dim = 0.55 · lift_amount`, with `lift_amount` eased over 0.4 s).
  - The lifted copy gets a radius 10–14 px, a MINT outline 2.5 px with glow (an exponential falloff over 12 px), and a drop shadow at 55%.
- **Inside a lift-out:** you can step a highlight across sub-regions (chips or tabs, 0.16–0.18 s each), or pulse a red rounded outline on a flagged value with `0.5 + 0.5·sin(7t)`. Map sub-region coordinates proportionally from the source region into the lifted box.

### 7.3 Callout pill

- **Look:**
  - Height 2× the font size (48).
  - PANEL fill at 96%, MINT stroke 1.5 px at 80%.
  - A MINT dot of radius 0.22× the font size, inset 0.95× the font size from the left.
  - Inter 600, 24 px text.
- **Timing:**
  - Appears 0.35 s after its lift-out starts and leaves 0.4 s before the lift-out ends.
  - Pop: scale 0.6→1 with `ease_back` over 0.45 s; alpha in over 0.15 s.
  - Exit: `ease_in²` over 0.3 s.
- **Optional anchor:** a MINT line 2.2 px from the pill edge to the target point, drawn on with expo-out over 0.4 s. It ends in a 6 px dot plus a 13 px ring at 60%.
- **Placement:** keep pills ≥ 70 px from the frame edges.

### 7.4 Cursor, hover, click, typing

- **Cursor:**
  - The classic arrow: black outline, white fill, drawn as a polygon at 4× supersampling then downsampled, scale 2.4, with a soft shadow offset (4, 5) px.
  - Path: quintic ease (`ease_io5`), 0.6–1.0 s, entering from off the card.
  - Press: shrink 12% for 0.12 s.
- **Hover:** a MINT circle stroke (2.5 px, radius 16–18, glow 0.7), fading in over 0.4 s before the click.
- **Click ring:** a MINT circle, radius 8→46 with `ease_out³` over 0.45 s, stroke 3 px, alpha `1−k`. Play the `click` SFX at the same instant.
- **Button feedback:** fill the button's projected box with MINT at 60–85%, fading out over 0.45–0.7 s.
- **Typing into a field:**
  - Cover the placeholder with the field's own colour, sampled as the median of pixels just right of the text.
  - Type 1 character per 36 ms in Inter 400 at 62% of the field height, colour INK.
  - Caret blinks at 2.5 Hz (on 60%).
  - Play a `key` SFX every other character.

### 7.5 Sequences and builders

- **Row ticks:** each row gets a MINT outline pulse (`sin(πk)` over 0.45 s), then a MINT check disc to its right (radius 15, `ease_back`), 0.3 s apart.
- **Revealing content:** messages "type in" with a cover rectangle (the background colour sampled from the frame) that slides off left to right with expo-out over 0.5 s, 0.5 s apart.
- **Chips popping in:** each chip is drawn from its own region, scaling 0→1 with `ease_back(s=1.8)` over 0.35 s, 0.18 s apart, over a cover patch of the bubble's colour.
- **Chart bars growing:** a cover rectangle over the bar area retracts upward with expo-out over 1.0 s.
- **Counters:** count up from 0 with `ease_out³` over about 1.5 s, in Inter 800 at 300 px.

---

## 8. Motion language

```python
def clamp(x, a=0.0, b=1.0): return max(a, min(b, x))
def lin(t, t0, t1): return clamp((t - t0) / (t1 - t0)) if t1 > t0 else float(t >= t1)   # progress 0..1
def mix(a, b, x): return a + (b - a) * x
def ease_io(x):  x = clamp(x); return x * x * (3 - 2 * x)                 # smoothstep
def ease_io5(x): x = clamp(x); return x**3 * (x * (6 * x - 15) + 10)     # quintic (cursor)
def ease_out(x, p=3): x = clamp(x); return 1 - (1 - x) ** p
def ease_in(x, p=3):  x = clamp(x); return x ** p
def ease_back(x, s=1.6):                                                  # overshoot pop
    x = clamp(x) - 1; return 1 + (s + 1) * x**3 + s * x**2
def ease_out_expo(x): x = clamp(x); return 1 if x >= 1 else 1 - 2 ** (-10 * x)
def ease_in_expo(x):  x = clamp(x); return 0 if x <= 0 else 2 ** (10 * x - 10)
def ease_io_expo(x):
    x = clamp(x)
    if x in (0, 1): return x
    return 2 ** (20 * x - 10) / 2 if x < .5 else (2 - 2 ** (-20 * x + 10)) / 2
def fade(t, t_in, t_out, d_in=.3, d_out=.3):
    return clamp(min((t - t_in) / d_in if d_in else 1, (t_out - t) / d_out if d_out else 1))
```

| Motion | Easing | Duration |
|---|---|---|
| Card entry | `ease_out_expo` | 0.9–1.1 s |
| Word reveal (per word) | `ease_out_expo`, masked slide-up of 1.15× the ascent | 0.55 s, stagger 0.07 s (subs 0.03 s) |
| Word exit | `ease_in²`, slide up 0.5× the size plus fade | 0.35 s, stagger 0.03 s |
| Pops (pills, chips, check discs) | `ease_back` (s 1.6–2.0) | 0.35–0.45 s |
| Lift-out in / out | `ease_back(1.1)` / `ease_io` | 0.55 s / 0.4 s |
| Cursor travel | `ease_io5` | 0.6–1.0 s |
| Transitions | `ease_io_expo` | 0.6–0.8 s (block wipe 1.2 s) |
| Exits (logo, mosaic collapse) | `ease_in_expo` | 0.5–0.6 s |
| Background drift | sinusoid / linear | continuous |

**Rules**
- Text and the card enter together, but the text is staggered 0.15 s after the chapter label, and line 2 is 0.12 s after line 1.
- Never move two focal things at once. Lift-outs, callouts and clicks happen in sequence.
- Every element that appears also leaves deliberately. No hard cuts mid-scene.
- Leave 0.3–0.5 s of stillness before a click so the eye finds the cursor.

---

## 9. Transitions catalogue

All transitions are a function `trans(t)` that renders **both** scenes at time `t` and composites them. Scenes must overlap in time by the transition length.

1. **Shape-grow (the signature).**
   - The element that was clicked (a pill, button or tab) is taken as a box on screen and grows into a rounded rect covering the frame plus 60 px: `lerp(box0, box1, ease_io_expo)`, with radius `mix(r0, 40)`.
   - Inside: the next scene, optionally tinted with the element's colour, which fades from 100% to 0% between 35% and 85% of the transition.
   - Rim: a MINT 4 px stroke with glow, alpha `(1−x)^0.5`.
2. **Circle iris:** the same as shape-grow, starting from a 32–36 px circle on a clicked icon, growing to cover the diagonal (radius = max corner distance).
3. **Push:** the old scene slides out and the new one in, horizontally or vertically, with a 6 px MINT seam at the boundary. Use it for a "same context, next view" step.
4. **Domain-shape wipe:** shapes from the next screen cover the frame (e.g. for a dashboard screen, blocks or tiles in the dashboard's colours). Each scales up with stagger `0.035·col + 0.02·row` in the first half, the scene swaps at the midpoint, then they scale down with `ease_in_expo`. Play a soft `block` SFX per block.
5. **Zoom-out crossfade:** only into the recap. The old scene scales 1.0→0.85 while crossfading.

Each transition plays a `whoosh` SFX that starts 0.2 s before the transition and lasts 0.5 s.

---

## 10. Intro, recap and end card recipes

### Intro (0–5 s)

| Time (s) | Action |
|---|---|
| 0.35–0.70 | A 26 px MINT dot pops (`ease_back`) |
| 1.0–1.8 | The dot stretches into a 520×90 MINT pill (`ease_io_expo`) with a soft glow |
| 1.8–2.4 | The pill morphs into the logo card: 860×260, radius 30, PANEL fill, MINT stroke 1.6 px at 55%, shadow |
| 2.2 | **Hit:** wordmark parts rise in (serif word, then light word +0.12 s), subtitle from +0.45 s with tracking 20→9. Hit SFX plus white flash 0.18 |
| from 1.35 | "Introducing" (Inter 400, 28, SUB, tracking 6) at y = 330 |
| from 3.05 | Tagline (Inter 400, 32, WHITE) at y = 780 |
| 4.35–4.95 | The card shrinks to 60% and flies up 420 px (`ease_in_expo`); text exits |

### Recap mosaic (69.2–73 s)

1. **Grid:** 4×3 tiles of 520×300 with 40 px gaps. The middle two cells of row 2 stay empty for the words.
2. **Tiles:** each is a "cover" crop of a screen at 80% brightness, radius 14, shadow. They pop in (`ease_back 1.3`) 0.06 s apart and drift outward to ×1.08.
3. **Four words** on beats at 70.5, 71.0, 71.5 and 72.0 s: scale 1.25→1 (`ease_back 2.0`); each holds 0.5 s. The last word is MINT and holds until the collapse. Play a `hit2` SFX (pluck chord plus clap) per word.
4. **Collapse:** 72.4–73.0, tiles shrink into the centre (`ease_in_expo`). Leave a **0.5 s silence** in the music before 73.0.

### End card (73–78 s)

| Time (s) | Action |
|---|---|
| 72.8–73.4 | The card grows 40→860×260 (`ease_io_expo`) |
| 73.0 | Final hit on the downbeat |
| 73.2 | Wordmark |
| 74.0 | End line (Inter 600, 30, MINT, tracking 2) at card centre + 200 |
| 74.8 | Credit (Inter 600, 18, SUB, tracking 8) at y = 1000 |
| 77.0–77.9 | Everything fades out; global fade over the last 0.6 s |

---

## 11. Audio: original score plus sound effects

Use 48 kHz stereo 16-bit WAV, synthesized in numpy and scipy with a fixed random seed (42) so it is deterministic.

### Grid and sections

| Item | Value |
|---|---|
| Tempo | **120 BPM**, beat 0.5 s, bar 2 s |
| Bar grid | Bars start at t = 1.0, 3.0, 5.0 … so scene changes on odd seconds land on downbeats |
| Chord loop | D – A – Bm – G, one chord per bar |
| Pad voicings | D [50, 54, 57, 62]; A [49, 52, 57, 61]; Bm [50, 54, 59, 62]; G [50, 55, 59, 62] |
| Bass roots | 38, 33, 35, 31 |

Arpeggios are 8-step patterns from the chord, two octaves up (e.g. D: 74 78 81 78 74 81 78 86).

| Section | Time (s) | What plays |
|---|---|---|
| intro | 0–5 | Pad, a 16th pluck arpeggio brightening, a noise riser at 3.4–5.0, a reversed cymbal into 5.0 |
| build | 5–11 | Kick on every beat, offbeat hats, bass 8ths (filter 600 Hz), arpeggio; no clap |
| groove | 11–23 | Adds clap on beats 2 and 4, bass filter opens to 900 Hz |
| full | 23–53, 61–69 | Adds 16th hats and a bell motif every other bar |
| lift | 53–61 | Brighter pad (filter 2.2 kHz), open hats on off-beats |
| drive | 69–72.5 | Groove continues under the recap words |
| gap | 72.5–73.0 | Silence |
| end | 73.0+ | Kick, cymbal, sub drop, sustained pad, a bell arpeggio (74, 78, 81, 86 every 0.25 s), long fade |

Snare fills: 16ths in the last 0.5 s before the big changes (23, 37, 53, 61).

### Instruments (synthesis recipes)

| Instrument | Recipe | Level |
|---|---|---|
| Kick | Sine with pitch 45 + 110·e^(−t/35 ms) Hz, amplitude e^(−t/220 ms), plus a 3 kHz noise click (4 ms), `tanh(1.8×)` | 0.85 |
| Clap | Three band-passed (1–3.5 kHz) noise bursts at 0, 11 and 22 ms; the last decays over 90 ms | 0.30 |
| Hats | 7 kHz high-passed noise, decay 20 ms (closed) or 90 ms (open) | 0.10 / 0.045 for 16ths |
| Bass | Saw plus a square sub-octave plus a sine sub, LPF 900 Hz, decay 120 ms, `tanh` | 0.30 |
| Pluck | Saw plus square, filter sweep 300 + 4000·e^(−t/60 ms) Hz, decay 160 ms, with a dotted-8th echo at 35% panned opposite | 0.11 |
| Pad | Three detuned saws per note (−8, 0, +7 cents, spread L/R), LPF 1500 Hz, 0.25 Hz amplitude LFO ±8% | 0.42 |
| Bell | Partials ×1, 2, 2.76 and 5.4 with decreasing decays | 0.035–0.06 |
| Side-chain | Pads duck 55% on every kick, release τ = 130 ms (the "pump" that makes it feel produced) | – |

**Reverb:** convolve with a 2.2 s exponentially decaying noise impulse response (τ 0.35 s), band-limited to 250 Hz–6 kHz, 15 ms pre-delay, wet 0.7. Send levels per voice are 0.02–0.6.

**Master chain:**
1. High-pass at 28 Hz.
2. Normalise to the peak.
3. Soft clip with `tanh(1.3x)/tanh(1.3)`.
4. Fade-out over the last 2.2 s.
5. Normalise the peak to 0.92.

Target section loudness is about −12 dBFS RMS, with the intro around −20.

### Sound effects (generated from the same cue list as the animation)

Build `cues = [(kind, time, gain)]` inside the scene code: every pop, click, tick, whoosh and hit is registered where the animation happens. The music renderer consumes that list.

| Kind | Sound | Gain |
|---|---|---|
| `whoosh` | Band-pass noise sweep 500→7000 Hz, 0.5 s, sine envelope, starting 0.2 s early | 0.16 |
| `pop` | Sine blip 520→1100 Hz, 90 ms | 0.12 |
| `tick` | Sine blip 2400→2600 Hz, 35 ms | 0.06 |
| `check` | Sine blip 1100→1700 Hz, 70 ms | 0.07 |
| `click` | 3.4 kHz sine (3 ms decay) plus a 2.5 kHz+ noise snap | 0.35 |
| `key` | 20 ms band-passed (1.5–5 kHz) noise tick | 0.08 |
| `success` | Two bells (MIDI 81, then 88 at +90 ms) | 0.06 |
| `block` | 110 Hz thud, 60 ms | 0.18 |
| `hit` | Cymbal plus sub drop 60→35 Hz; also triggers the 0.18 white flash | 0.08 / 0.25 |
| `rise` | 1 s band-pass noise riser 400→9000 Hz | 0.22 |

**Licensed music instead:** if the company prefers a library track, first detect its beat grid, then place scene changes on downbeats. Keep the synthesized sound effects.

---

## 12. Render and encode pipeline

```
launchvideo/
  engine.py   easing, Plate (screenshot pyramid), Cam (perspective), SDF shapes, text, cursor, background, transitions
  config.py   load screens.json (regions → pixels) and storyboard.json
  scenes.py   one function per scene + trans(t) + build_cues() + render(t)
  music.py    make_score(cues, path, dur)
  cli.py      check | preview | still <t> | music | render [--start --end --workers --crf --max-mb]
config/screens.json, config/storyboard.json, fonts/*.ttf (+ OFL), tools/annotate.html
screens/ and output/ are git-ignored (screenshots contain internal data)
```

- **Frames:** `multiprocessing.Pool(n_cores, initializer=load_config)` with `imap(render_frame, range(f0, f1), chunksize=4)`, writing bytes to ffmpeg's stdin. The initializer loads the screenshots once per worker. Speed is about 0.4 s per frame per core in numpy, so a 78 s render takes about 15 minutes on 2 cores.
- **Master encode:**

```bash
ffmpeg -y -f rawvideo -pix_fmt rgb24 -s 1920x1080 -r 30 -i - -i score.wav \
  -vf scale=out_color_matrix=bt709:out_range=tv -c:v libx264 -preset slow -crf 16 \
  -pix_fmt yuv420p -colorspace bt709 -color_primaries bt709 -color_trc bt709 \
  -c:a aac -b:a 256k -shortest -movflags +faststart master.mp4
```

- **Copy sized for sharing** (two-pass, sized to fit): `video_kbps = MB·8e6/duration/1000·0.93 − 160`, `-maxrate 2.2×`, `-bufsize 3.3×`, AAC 160k. **Use 18 MB for email**: Outlook and Gmail cap attachments at 20–25 MB, and encoding adds about 35%.
- **ffmpeg location:** use `FFMPEG` env var → `shutil.which('ffmpeg')` → `imageio_ffmpeg.get_ffmpeg_exe()`. Never ask the user to install ffmpeg with admin rights.
- **Windows:** read images with `np.fromfile` + `cv2.imdecode`, which handles non-ASCII paths. Guard the entry point for multiprocessing spawn (`__main__.py`). Give each worker process its own temp files.

---

## 13. Image-quality rules (this is where "crisp" comes from)

1. **Pyramid with no upscaled levels.** Build the screenshot pyramid at **half-octave steps** (×0.707), using `INTER_AREA`. For each warp, pick the smallest level whose scale is ≥ the on-screen zoom. Never shrink more than about 0.7× in a single bilinear warp; that shimmers. Never enlarge a reduced level.
2. **Limit upscaling.** Don't enlarge source pixels more than about 1.3×. If a lift-out needs more, make it smaller or ask the user for a higher-resolution screenshot.
3. **Premultiplied alpha** for cards and crops. This avoids dark fringes on the rounded corners.
4. **Draw shapes with signed distance fields (SDF).** Rounded rects, circles and capsules use per-pixel coverage `clip(0.5 − d, 0, 1)`, which gives anti-aliasing with no supersampling. Compute only inside each shape's bounding box, for speed.
5. **Text** comes from Pillow at the final size. Masked reveals shift the mask, not a scaled bitmap.
6. **Keep grain and vignette subtle** (σ 0.006). Heavy grain destroys small UI text after H.264.
7. **Encode the master at CRF 16** and judge sharpness **on the encoded file**, not on raw frames.

---

## 14. Workflow and milestones (stop and show the user at each ★)

1. **Setup.** Create a venv, install the packages, and copy the fonts. Build `tools/annotate.html` (§4) and pass its acceptance test. Run `check` with the screenshots. ★ Show the region overlays and list any screens that are too low-resolution (dialogs < 1000 px wide, pages < 1600 px).
2. **Storyboard.** Write `storyboard.json` and a timing table (scene, start/end, transition-in, lift-outs, click target). ★ Get approval on the words and flow **before** any animation work.
3. **Engine plus three stills.** Render one frame each of the intro (t = 2.9), a mid-chapter lift-out, and a transition at its midpoint. ★ Show them at full size.
4. **Preview sheet.** Render 20–24 key frames into a contact sheet (§15). Fix what's wrong. ★ Show the sheet.
5. **Music.** Run `make_score(cues)`. Plot the waveform with the scene boundaries marked, and confirm the downbeats line up. ★ Share the WAV.
6. **Full render**, then the size-targeted copy. Check the encoded file (§15). ★ Deliver the master plus the sized copy, with a one-paragraph summary and a list of anything the user must verify (claims, numbers, wording).

**Revisions:** change config or text first, scene code second, engine last. After every change, re-run `preview` before the full render.

---

## 15. QA protocol: look at the pixels; never assume

**The agent must open and look at rendered images.** "The code ran" is not "it looks right".

- **Contact sheet times:** 2.9, 9.5, then for each chapter: mid-entry (t0+0.5), each lift-out at full hold, the click moment, and each transition at its midpoint. Then 71.0 and 75.5. Use 640×360 thumbnails, 3 per row, with the time printed on each.
- **1:1 crops:** 600×300 px crops of lift-outs and small table text, to check sharpness.
- **On each frame, check:**
  - nothing overlaps (text column vs card vs callouts);
  - callouts point at the right element;
  - nothing is cut off at the frame edges;
  - the card leans toward the text;
  - regions sit on the real UI elements;
  - there are no leftover cover rectangles;
  - colours are consistent.
- **Transitions:** check the frames at 25%, 50% and 75% through. The shape must start exactly on the clicked element.
- **Audio:** plot the waveform and spectrogram with vertical lines at the scene changes. Check there is no clipping (peak ≤ 0.95), the gap at 72.5–73.0 is silent, and the hit lands at 73.0.
- **Final file:** use `ffprobe` to confirm the duration (78.0 s), 1920×1080 at 30 fps, h264 High with yuv420p and bt709, AAC 48 kHz stereo, and the file size. Pull 4–6 frames **from the MP4** and inspect them.
- **If anything was redacted:** OCR one frame per second of the final MP4 (tesseract, `--psm 11`, on both normal and inverted images) and flag every run of 4 or more digits that isn't a year.

---

## 16. Common failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| UI text looks soft or blurry | Low-resolution capture enlarged; or a reduced pyramid level enlarged | Ask the user for a higher-resolution screenshot (§4); use the half-octave pyramid rule (§13) |
| Text shimmers or looks jagged during motion | Shrinking more than 2× in one bilinear warp | Use pyramid levels with `INTER_AREA` |
| Rainbow ripples, grey glare | Photo of a monitor | Use real screenshots; else the §4 cleanup |
| Highlight boxes miss the element | Hard-coded pixels; framing changed | Fractional regions + the `check` overlay + `tools/annotate.html` |
| Editor ignores clicks or drags | An invisible overlay (empty-state message, label) sits on top of the canvas | Hide it with `display:none` (`[hidden]{display:none!important}`); test drag in a headless browser |
| Card faces away from the headline | Wrong `ry` sign | Card's near edge toward the text; check visually |
| Callout pill runs off screen | Anchored to a box near the edge | Clamp x to ≥ 70 px; flip alignment to the right |
| Two things move at once and it feels busy | Overlapping lift-outs, callouts or text | Sequence them; one focus at a time |
| Music feels "off" from the cuts | Bar grid not aligned; SFX hand-placed | Bars on odd seconds; generate sound effects from the scene cue list |
| Harsh or clipping audio | No master chain | High-pass, normalise, `tanh` soft clip, peak 0.92 |
| Fonts look different on another PC | System font fallback | Bundle the TTFs; load by path |
| Email bounces the video | File too large after encoding | Use the two-pass copy sized to 18 MB |
| Parallel render produces garbage or crashes | Workers sharing temp files or state | Per-worker initialiser; process-ID-named temp files |
| Agent says "done" but it's wrong | No visual check | §15 is mandatory; describe what each frame shows |
| Invented numbers or claims in the copy | Gaps filled with guesses | Only use facts from the screenshots or the user; list the claims for the user to verify |

---

## 17. Acceptance checklist

- [ ] Every UI pixel comes from a screenshot; no redrawn UI.
- [ ] `tools/annotate.html` loads the configs and screenshots, edits regions and labels, warns about low-resolution lift-outs, and passed its headless acceptance test.
- [ ] Each chapter has the label, a two-line headline (line 2 mint), two sub lines, 1–3 lift-outs with callouts, and a click into the next scene.
- [ ] Every transition starts from the element that was clicked (or is a push, block wipe or the final crossfade).
- [ ] No overlaps or edge clipping, and every card leans toward its text.
- [ ] Small UI text is readable in the encoded MP4 at 100% view.
- [ ] Music downbeats land on scene changes, there is a silent gap before the end-card hit, and nothing clips.
- [ ] The end line is accurate ("Coming soon" or "Now live"), and there is no PII in the overlays.
- [ ] The master (CRF 16) and the size-targeted copy are both delivered, and ffprobe output is checked.
- [ ] Screenshots and renders are git-ignored, and the README explains capture → check → preview → render.

---

## Appendix A: minimal engine skeleton (tested; extend it, don't replace it)

This renders a 4-second proof clip: the background, a tilted screenshot card entering, a word-by-word headline, a lift-out with a MINT outline, and an H.264 encode. Run `python skeleton.py path/to/screenshot.png`.

```python
import math, subprocess, sys, shutil
import numpy as np, cv2
from PIL import Image, ImageDraw, ImageFont

W, H, FPS, DUR = 1920, 1080, 30, 4.0
MINT = np.array([0.26, 0.87, 0.62], np.float32); WHITE = np.array([0.97, 0.98, 0.98], np.float32)
FONT = 'fonts/inter800.ttf'   # bundle Inter (OFL)

def clamp(x, a=0., b=1.): return max(a, min(b, x))
def lin(t, t0, t1): return clamp((t - t0) / (t1 - t0))
def mix(a, b, x): return a + (b - a) * x
def ease_out_expo(x): x = clamp(x); return 1 if x >= 1 else 1 - 2 ** (-10 * x)
def ease_back(x, s=1.1): x = clamp(x) - 1; return 1 + (s + 1) * x**3 + s * x**2

def load_plate(path, radius=16):
    img = cv2.cvtColor(cv2.imdecode(np.fromfile(path, np.uint8), cv2.IMREAD_COLOR), cv2.COLOR_BGR2RGB).astype(np.float32) / 255
    h, w = img.shape[:2]; a = np.zeros((h, w), np.float32)
    cv2.rectangle(a, (radius, 0), (w - radius, h), 1, -1); cv2.rectangle(a, (0, radius), (w, h - radius), 1, -1)
    for c in [(radius, radius), (w - radius, radius), (radius, h - radius), (w - radius, h - radius)]:
        cv2.circle(a, c, radius, 1, -1, cv2.LINE_AA)
    rgba = np.dstack([img * a[..., None], a])                      # premultiplied
    levels, s = [(1.0, rgba)], 1.0
    while min(w, h) * s / 2 ** .5 >= 64:                           # half-octave pyramid
        s /= 2 ** .5
        levels.append((s, cv2.resize(rgba, (round(w * s), round(h * s)), interpolation=cv2.INTER_AREA)))
    return w, h, levels

def pick(levels, zoom):
    best = levels[0]
    for s, im in levels:
        if zoom <= s * 1.02: best = (s, im)
    return best

def rot(rx, ry, rz):
    rx, ry, rz = map(math.radians, (rx, ry, rz))
    X = np.array([[1, 0, 0], [0, math.cos(rx), -math.sin(rx)], [0, math.sin(rx), math.cos(rx)]])
    Y = np.array([[math.cos(ry), 0, math.sin(ry)], [0, 1, 0], [-math.sin(ry), 0, math.cos(ry)]])
    Z = np.array([[math.cos(rz), -math.sin(rz), 0], [math.sin(rz), math.cos(rz), 0], [0, 0, 1]])
    return Z @ Y @ X

def project(pts, pw, ph, zoom, R, cx, cy, f=1700):
    P = np.column_stack([(pts[:, 0] - pw / 2) * zoom, (pts[:, 1] - ph / 2) * zoom, np.zeros(len(pts))]) @ R.T
    z = P[:, 2] + f
    return np.column_stack([f * P[:, 0] / z + cx, f * P[:, 1] / z + cy]).astype(np.float32)

def card(frame, plate, width, cx, cy, rx, ry, rz, dim=0.0):
    pw, ph, levels = plate; zoom = width / pw
    s, img = pick(levels, zoom)
    src = np.float32([[0, 0], [pw, 0], [pw, ph], [0, ph]])
    M = cv2.getPerspectiveTransform(src * s, project(src, pw, ph, zoom, rot(rx, ry, rz), cx, cy))
    lay = cv2.warpPerspective(img, M, (W, H), flags=cv2.INTER_LINEAR, borderValue=(0, 0, 0, 0))
    sh = np.roll(cv2.GaussianBlur(lay[..., 3], (0, 0), 30), 22, axis=0)
    frame *= 1 - 0.55 * sh[..., None]
    rgb = lay[..., :3] * (1 - dim); frame *= 1 - lay[..., 3:4]; frame += rgb
    return M

def rrect_alpha(box, r):
    x0, y0, x1, y1 = box                                            # clip to frame: shapes may extend off-screen
    X0, Y0, X1, Y1 = max(0, int(x0) - 2), max(0, int(y0) - 2), min(W, int(x1) + 3), min(H, int(y1) + 3)
    ys, xs = np.mgrid[Y0:Y1, X0:X1].astype(np.float32) + .5
    qx = np.abs(xs - (x0 + x1) / 2) - (x1 - x0) / 2 + r; qy = np.abs(ys - (y0 + y1) / 2) - (y1 - y0) / 2 + r
    d = np.hypot(np.maximum(qx, 0), np.maximum(qy, 0)) + np.minimum(np.maximum(qx, qy), 0) - r
    return (X0, Y0, X1, Y1), d

def stroke(frame, box, r, color, width=2.5, alpha=1.0):
    (X0, Y0, X1, Y1), d = rrect_alpha(box, r)
    a = np.clip(width / 2 + .5 - np.abs(d), 0, 1)[..., None] * alpha
    glow = np.exp(-np.abs(d) / 12)[..., None] * 0.5 * alpha
    reg = frame[Y0:Y1, X0:X1]; reg += glow * color; reg *= 1 - a; reg += a * color

def text(frame, s, size, x, y, color, rise=0.0):
    f = ImageFont.truetype(FONT, size); asc, desc = f.getmetrics(); pad = size // 2
    im = Image.new('L', (int(f.getlength(s)) + 2 * pad, asc + desc + 2 * pad)); ImageDraw.Draw(im).text((pad, pad), s, font=f, fill=255)
    a = np.asarray(im, np.float32) / 255
    off = int(rise * asc * 1.15)
    if off: a = np.vstack([np.zeros((off, a.shape[1]), np.float32), a[:-off]]); a[int(pad + asc * 1.28):] = 0  # masked slide-up
    x0, y0 = int(x - pad), int(y - pad - asc * .62); h, w = a.shape
    reg = frame[y0:y0 + h, x0:x0 + w]; m = a[:reg.shape[0], :reg.shape[1], None]
    reg *= 1 - m; reg += m * color

def background():
    yy, xx = np.mgrid[0:H, 0:W].astype(np.float32)
    f = np.array([0.043, 0.055, 0.062]) * (1 - yy / H)[..., None] + np.array([0.028, 0.036, 0.040]) * (yy / H)[..., None]
    d = np.hypot(xx - .22 * W, yy - .25 * H) / 900; f += np.exp(-2.2 * d * d)[..., None] * np.array([.06, .25, .20]) * .55
    return f.astype(np.float32)
BG = background()

def render(t, plate):
    f = BG.copy()
    e = ease_out_expo(lin(t, 0.2, 1.2))
    lift = ease_back(lin(t, 1.8, 2.35)) * (1 - clamp((t - 3.4) / 0.4))
    M = card(f, plate, 1120, 1250, 560 + (1 - e) * 200, mix(14, 3, e), mix(28, 8, e), mix(-4, 0, e), dim=0.55 * clamp(lift))
    x, fnt = 120, ImageFont.truetype(FONT, 58)                      # word-by-word reveal, positions from the font
    for i, wd in enumerate('Every request. One queue.'.split()):
        p = ease_out_expo(lin(t, 0.5 + 0.07 * i, 1.05 + 0.07 * i))
        if p > 0: text(f, wd, 58, x, 410, WHITE, rise=1 - p)
        x += fnt.getlength(wd + ' ')
    if lift > 0.001:                                                # lift-out of the top-right quarter
        pw, ph, levels = plate; reg = np.float32([[pw * .55, 0], [pw, 0], [pw, ph * .25], [pw * .55, ph * .25]])
        on_card = cv2.perspectiveTransform(reg[None], M)[0]
        src_box = (on_card[:, 0].min(), on_card[:, 1].min(), on_card[:, 0].max(), on_card[:, 1].max())
        tw = min(760, pw * .45 * 1.3); th = tw * (ph * .25) / (pw * .45)
        dst = tuple(mix(a, b, lift) for a, b in zip(src_box, (1260 - tw / 2, 260 - th / 2, 1260 + tw / 2, 260 + th / 2)))
        s, img = pick(levels, (dst[2] - dst[0]) / (pw * .45))
        A = cv2.getAffineTransform(reg[:3] * s, np.float32([[dst[0], dst[1]], [dst[2], dst[1]], [dst[2], dst[3]]]))
        crop = cv2.warpAffine(img, A, (W, H), flags=cv2.INTER_LINEAR, borderValue=(0, 0, 0, 0))
        (X0, Y0, X1, Y1), d = rrect_alpha(dst, 12); m = np.zeros((H, W, 1), np.float32)
        m[Y0:Y1, X0:X1, 0] = np.clip(.5 - d, 0, 1)
        rgb = crop[..., :3] / np.maximum(crop[..., 3:4], 1e-3)
        f[:] = f * (1 - m) + rgb * m
        stroke(f, dst, 12, MINT, 2.5, clamp(lift))
    g = min(1, t / .5) * min(1, (DUR - t) / .6)
    return np.clip(f * g * 255 + .5, 0, 255).astype(np.uint8)

if __name__ == '__main__':
    plate = load_plate(sys.argv[1])
    try:
        import imageio_ffmpeg; ff = shutil.which('ffmpeg') or imageio_ffmpeg.get_ffmpeg_exe()
    except ImportError:
        ff = shutil.which('ffmpeg')
    p = subprocess.Popen([ff, '-y', '-loglevel', 'error', '-f', 'rawvideo', '-pix_fmt', 'rgb24', '-s', f'{W}x{H}', '-r', str(FPS),
                          '-i', '-', '-c:v', 'libx264', '-preset', 'slow', '-crf', '16', '-pix_fmt', 'yuv420p', 'skeleton.mp4'],
                         stdin=subprocess.PIPE)
    for i in range(int(DUR * FPS)): p.stdin.write(render(i / FPS, plate).tobytes())
    p.stdin.close(); p.wait(); print('wrote skeleton.mp4')
```

From here, add the remaining pieces: the full background (dot grid, shapes, grain, vignette), the callout pill, cursor and click ring, `words_in` exits, transitions with `trans(t)`, scenes driven by `screens.json` regions, the music module, the CLI and the process pool.
