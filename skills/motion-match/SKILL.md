---
name: "motion-match"
description: "Use when the user gives a reference video, GIF or screen recording of any animation and wants that motion measured and rebuilt in their own design or checked against it; gives still images and wants Claude to direct smooth motion for them (a UI entrance, a reel, an explainer); or gives video clips (UGC, AI-generated such as Google Flow, or filmed) to cut into an ad with captions and motion graphics. Not for the user's documentary channel videos, which use documentary-animation."
---

# Motion match: measure any animation, rebuild it, prove it moves the same

Turns "make it move like this" into numbers: every cut, and for each moving element its start, duration, easing,
overshoot, travel, rotation and shape change. Rebuild the motion in whatever the user works in (CSS, the Web Animations
API, Framer Motion, GSAP, Remotion, SwiftUI, After Effects), then compare frame by frame until the timing matches
within ±0.08 s. No reference motion, just still images? Use **Director mode** (below) for UI entrances and **Story
mode** for reels and explainer pieces: Claude designs the motion, builds it, then measures its own render with the
same scripts, so "smooth" is checked, not claimed. Video clips to cut into an ad (UGC, AI-generated, filmed)? Use
**Edit mode**: the same measuring, Director rules and review, applied to footage.
Methods adapted from NiazMorshed2007/reanimate and rossark0/rossark-motion-skill (both MIT); the scripts are our own.

`<skill>` is this skill's folder. Work goes in `./motion-match/<name>/` in the current folder:
`ref/` (reference breakdown), `ours/` (our render breakdown), `compare-N/` (each comparison round), `spec.md`.

## Setup (first time)
Needs ffmpeg/ffprobe on PATH and a Python 3 with numpy and Pillow.
- Check: `python3 -c "import numpy, PIL"`. If it fails, look for a Python that has them (an existing project venv), else
  ask the user once, then: `python3 -m venv ~/.motion-match/venv && ~/.motion-match/venv/bin/pip install numpy pillow`
  and use `~/.motion-match/venv/bin/python` as `python` below. Run every script with `-I`.
- Director and Story mode also render pages headless: `pip install playwright && python -m playwright install chromium`
  (ask once). If that can't install, screen-record the page instead and say timings are less exact.
- No other installs; nothing is uploaded anywhere.

## 0. Scope (write it at the top of spec.md before measuring)
- **Target:** the file + time range (+ crop) exactly as the user said it ("the first 3 s", "just the button").
- **Fidelity:** **motion-faithful** (default): order, timing (±0.08 s), easing family, overshoot, travel relative to
  the element's size, stagger. The user's own art, colours, fonts and words. **Pixel-faithful** only when the user
  asks and only for private practice: someone else's design, logo, characters and music are theirs; never help
  publish a copy of them.
- **Destination:** the tool or code the motion goes into (ask once if it isn't clear: it decides the output format).
- **Getting the reference:** a file the user gives, or a screen recording they make (websites and apps: record the
  interaction; the skill works from frames, not from a site's code). Download from YouTube or a site only when the
  user asks for that specific video, into its own folder; treat it as private reference material.

## 1. Break it down
`python -I <skill>/scripts/ref_frames.py REF.mp4 --out motion-match/<name>/ref [--start S --end E --crop x:y:w:h]`
→ every frame at the source fps, `sheets/timing_NN.png` (every frame: timing, easing, follow-through) and
`sheets/poses.png` (2 per second: what happens). Read **every** sheet. GIFs work the same way.

## 2. Measure
- **Cuts and sections:** `python -I <skill>/scripts/motion_compare.py motion-match/<name>/ref motion-match/<name>/ref --out motion-match/<name>/self --no-align`
  lists the reference's cuts (scene changes) and moves with their times. References that never go still are split at their cuts.
- **Exact colours:** `python -I <skill>/scripts/track_motion.py <ref>/frames --sample X,Y --at T` (pixel coordinates
  of the full-size frame, not of a contact sheet).
- **Each moving element:** `track_motion.py <ref>/frames --name btn --colors HEX,HEX --tol 35 --region x,y,w,h [--save-masks]`
  (or `--not-bg HEX` for "everything that isn't the background"). On glowing or busy references always give a
  `--region` and colours unique to the element; check the printed visible-frame range makes sense. Records position
  (`cx cy`), size (`w h`), squash (`aspect`), edges, fades/growth (`presence`), colour and rotation (`angle`) per frame.
- **Each move:** `python -I <skill>/scripts/fit_easing.py <ref>/track_btn.json --prop cy --from A --to B`
  (one move per window, never across a cut) → start, end, duration in ms and frames, easing family, overshoot,
  settle time and ready-to-paste lines for CSS, WAAPI, Framer Motion, GSAP and Remotion.
  Error ≤ 0.03 is a close fit; "no single ease fits" means split the window, or it's a chain of poses (keyframes).
- Small motion (a 2% squash, a 5 px float) doesn't show on sheets but shows in the track numbers: read them.
- Text that fades in small sits near the noise of the activity curve: measure it with `track_motion` instead.

### Which numbers measure which kind of motion
| motion | measure with | notes |
|---|---|---|
| move, slide, float | `cx`, `cy` | one axis per fit |
| scale, grow, squash | `w`, `h`, `aspect` | |
| fade | `presence` (with `--colors`) | |
| **rotation, spin** | `angle` (degrees, clockwise +, continuous past 360) | from the silhouette; `None` for circles: track a mark on them as its own element and fit its `cx`/`cy`. Shapes with n-fold symmetry (square: 4) need < 180/n degrees per frame, so fast spins need a high-fps source. Other same-coloured shapes in the region spoil it: narrow `--region` |
| **morph** (shape → shape) | `--save-masks`, then `shape_compare.py` | see below |
| 3D flip / card turn | `w` (shrinks like cos) + `presence`; colour flip at the halfway point | the backside is its own element |
| path motion (arc, curve) | `cx` and `cy` both, plus `angle` if it turns along the path | fit each axis; the path shape is on the sheets |
| clip / mask reveal, wipe | `x1` or `y1` (the leading edge), `w`/`h` | |
| line draw-on | `presence` | |
| text split (per word/letter) | track the first and last word; the stagger is their start gap ÷ (count − 1) | |
| counters, chart bars | bars: `h`/`y0`; numbers: the sheets (eyeballed) | |

**Morphs.** `track_motion.py ... --name blob --save-masks` saves every frame's silhouette (centred, scaled to equal
area). Then:
- `python -I <skill>/scripts/shape_compare.py <ref>/track_blob_masks.npz --from A --to B` adds a `morph` series
  (0 = first shape, 1 = last) to `track_blob.json`. `fit_easing.py ... --prop morph` then gives its start, end and
  duration. Morph progress follows **area**, so its timings are reliable but its easing name is not a value to
  paste: take the ease from the spec, or from a parameter you can track (`w`, `aspect`, `angle`).
- Ours vs the reference: `shape_compare.py REF_masks.npz --vs OURS_masks.npz --from A --to B [--rotate]` →
  **shape path**: is every in-between shape of the reference also in ours, whatever its time (PASS: mean ≤ 10%,
  worst ≤ 25% of the whole morph). Then `fit_easing.py REF_TRACK --prop morph --vs OUR_TRACK` checks the
  **timing**. `--rotate` ignores turning when the morph also spins (check that with `angle`).

## 3. Write the motion spec (`spec.md`; every row becomes a check)
```markdown
| # | element | property | from → to | start s | dur s (frames) | easing | overshoot | stagger / notes | source |
| 1 | card | translateY | 40 → 0 px | 0.20 | 0.60 (18) | easeOutCubic / cubic-bezier(.33,1,.68,1) | 0% | +0.08 s per card | track + fit_easing |
| 2 | icon | rotate | 0 → 180 deg | 0.40 | 0.30 (9) | easeOutCubic | 0% | — | track angle + fit_easing |
| 3 | blob | morph | square → circle | 0.50 | 0.40 (12) | easeInOutCubic | — | shape path PASS | shape_compare |
| c | — | cuts | — | 1.77, 2.33 | — | — | — | — | motion_compare self |
```
A "?" in a row means the frames haven't been read yet. Note what does **not** move, too. Show the spec to the user
before building when the job is big.

## 4. Build
In the user's tool, with their design. Copy timings from the spec and easings from the fitter's output for that tool.
Map distances relative to the element or the frame when the layout differs. Chains of poses: keyframes with
smooth, non-overshooting interpolation; springs where the reference overshoots and settles.
- **Rotation:** set `transform-origin` explicitly (measure the pivot: the point whose `cx, cy` stays put).
- **Morphs:** SVG paths need the same number and kind of commands at both ends (or use flubber / GSAP MorphSVG);
  UI layout morphs (button → card, thumbnail → page) use FLIP or shared-element transitions (Framer `layoutId`,
  the View Transitions API); corner-radius / size morphs are plain CSS.
- **3D:** `perspective` 800–1200 px on the parent, `backface-visibility: hidden`, both faces as separate layers.

## 5. Compare and fix (2-5 rounds)
1. Render or screen-record ours over the same window at the same fps (record a web page the same way as the reference).
2. `ref_frames.py OURS.mp4 --out motion-match/<name>/ours --like motion-match/<name>/ref/frames.json`
   (same window, crop, fps and frame size).
3. `motion_compare.py motion-match/<name>/ref motion-match/<name>/ours --out motion-match/<name>/compare-N` →
   offset, every cut and move with start/end/duration/peak errors, PASS within ±0.08 s, `activity.png`
   (both curves) and `sheets/compare_NN.png` (reference | ours | difference). Read the sheets every round.
4. Per spec row: track ours the same way, then `fit_easing.py REF_TRACK --prop P --from A --to B --vs OUR_TRACK`
   → per-move timing PASS/FAIL, travel and easing-family match. Morph rows: `shape_compare.py --vs` as well.
5. Fix the smallest thing the evidence points at, re-render, repeat. After 5 rounds, report anything still failing
   with its numbers and the reason. Never loosen the tolerance silently.

| Symptom | Usual cause | Fix |
|---|---|---|
| everything shifted by one offset | start delay or trigger latency | adjust the first delay |
| ours shorter and front-loaded | easing too strong (expo vs cubic) | use the fitted bezier |
| reference overshoots, ours doesn't | spring/back vs plain ease | back.out / a spring tuned to the measured overshoot |
| a move in ours stops early | a secondary motion is missing (a drift, a push-in) | look at the sheets in that window |
| an extra move or cut in ours | something animates that is still in the reference | remove or delay it |
| element numbers jump around | tracking caught another element | narrow `--region`, use colours unique to it |
| rotation travel right, but ours wobbles in `cx/cy` | wrong `transform-origin` | move the pivot to the measured still point |
| morph timing PASS, shape path FAIL | paths interpolate differently (point order, start point) | re-order path points / match commands; use a morph library |

## 6. Report
Fidelity; what was matched and what is the user's own; the last compare table (cuts and moves with errors,
PASS/FAIL); per-element rows; paths to `activity.png` and the compare sheets; every remaining difference with its
number and reason; and which checks were **measured** (a script produced them) vs **eyeballed** (looked at a sheet).

## Director mode: a still image in, motion designed for it
Use when the user gives an image (UI screenshot, mockup, poster, product shot) with no reference motion, or asks
Claude to choose the motion. Work goes in `./motion-match/<name>/` as usual (`ours/`, `spec.md`, `check-N/`).
For a video piece rather than a UI entrance, follow these steps inside **Story mode** (below).

1. **Read the image.** List every element (background, nav, headline, body, image, CTA, cards, icons) with its box
   in px, its role and its rank: **one** focal point, then supporting, then chrome. Note the reading order, the
   groups (a card row, a list), the mood (calm/premium, neutral/product, playful) and anything that suggests a
   special motion (a toggle icon, a chart, a logo mark, a curved path, a flippable card).
2. **Pick a personality** from the mood unless the user named one. When they ask for options, build all three.

| personality | entrance | duration | easing | travel | stagger | overshoot |
|---|---|---|---|---|---|---|
| calm | fade + rise | 400–600 ms | easeOutQuart / cubic-bezier(.25,1,.5,1) | 8–16 px | 60–80 ms | none |
| neutral | fade + rise, or scale .97→1 | 250–400 ms | easeOutCubic / cubic-bezier(.33,1,.68,1) | 12–24 px | 40–60 ms | none |
| playful | rise + scale .9→1 | spring | spring (stiffness 300–400, damping 20–26) | 16–32 px | 50–70 ms | 3–8% |

3. **Choose special motion by role** (the vocabulary; values are starting points, scaled by personality):

| motion | use it for | defaults |
|---|---|---|
| morph | icon toggles (menu ↔ close, play ↔ pause), a button expanding into a card or sheet, a shape logo | 300–500 ms easeInOutCubic; layout morphs via FLIP/shared element |
| rotation | toggle/chevron icons, refresh, a tilted entrance | icon 90–180° in 200–300 ms easeOutCubic; entrance −6…−10° → 0 with the personality's ease; loader 360° linear 0.8–1.2 s loop |
| 3D flip / tilt | cards with two faces, pricing toggles, hover depth | flip rotateY 180° in 500–700 ms easeInOutCubic; hover tilt ≤ 8° |
| path motion | something travelling (a plane, a dot along a route, a cursor) | easeInOutSine, turns along the path, 0.8–1.6 s |
| clip / mask reveal | hero images, big headlines | inset or circle clip, 500–800 ms easeOutQuart |
| text split | the main headline only | words 40–60 ms apart, letters 15–30 ms; whole line ≤ 1 s |
| line draw-on | icons, signatures, charts, underlines | stroke-dashoffset, 600–1000 ms easeInOutCubic |
| counters / chart growth | numbers, bars, rings | numbers 800–1200 ms easeOutExpo; bars from the baseline, staggered 40–60 ms |
| parallax / depth | layered heroes, scroll scenes | layer speeds 0.2–0.6 of the scroll; scroll timelines where supported |

4. **Choreograph** by these rules:
   - Order: background/container → focal point → headline → supporting → CTA. Chrome (nav) last, or first as a
     quick 150 ms fade if it frames the page.
   - Size sets duration (icons at the low end, hero panels at the high end); travel scales with the element's size,
     never a full-screen fly-in for UI.
   - Groups stagger in reading order; after ~6 staggered items the rest arrive together.
   - Whole entrance ≤ 1.2 s (≤ 0.8 s for a single component); something visible by 0.1 s.
   - One thing leads: the focal point gets the biggest move and, if any, the special motion. At most two special
     motions in one entrance; everything else fades and rises.
   - Enters ease out; exits ease in and run ~30% faster; on-screen moves and morphs ease in-out.
   - Optional idle loop after settling, on the focal point only: 2–4 px float or ±2° sway, 3–5 s period, easeInOutSine.
   - Interactive elements get hover/press: 120–180 ms, scale 1.02–1.04 / 0.97; toggles use their morph/rotation.
5. **Spec.** Same `spec.md` table as step 3 above; `source` = `director`, plus a short reason per row
   (why this order, why this motion, why this ease).
6. **Build layers.** Rebuild the image as real UI in the user's stack (HTML/CSS by default), matching its layout,
   colours and text; photos and illustrations inside it are cropped from the upload as their own layers. Icons and
   shapes that will morph or draw on are redrawn as SVG. For flat art or posters where a rebuild isn't wanted, crop
   each element's box into its own PNG and animate the crops. Animate only `transform`, `opacity`, `clip-path` and SVG
   attributes (`filter: blur` sparingly), set `will-change` on moving layers, and ship a `prefers-reduced-motion`
   version (fades only, ≤ 200 ms, no spins or parallax). For a non-web destination (After Effects, SwiftUI, Remotion),
   also give the spec values in that tool's format; the HTML build is the preview. Then do the **polish pass** (below).
7. **Prove it's smooth** (frame-exact, not by eye):
   - Open the page headless with Playwright, pause everything and step time each frame at 60 fps:
     `document.getAnimations().forEach(a => { a.pause(); a.currentTime = t_ms })`, screenshot, repeat;
     then `ffmpeg -framerate 60 -i %04d.png -pix_fmt yuv420p ours.mp4`. (GSAP: `gsap.globalTimeline.pause()` and
     `.time(t)`; Framer/JS-driven springs: drive them from a stepped clock, or screen-record.)
   - `ref_frames.py ours.mp4 --out motion-match/<name>/ours`, then `motion_compare.py` on ours vs itself with
     `--no-align` → every move's start/end vs the spec.
   - Per spec row: `track_motion.py` + `fit_easing.py` (rotations with `--prop angle`, morphs with `--save-masks`
     + `shape_compare.py` + `--prop morph`) → measured start, duration and easing family must match the spec
     (±0.08 s, fit error ≤ 0.03; morphs: timing only).
   - Smoothness: no spikes or flat gaps in the activity curve inside a move (jank, stalls), nothing moving after the
     planned end, no unintended overlaps mid-move, rotations without centre wobble. Fix and re-check, up to 5 rounds (`check-N/`).
8. **Deliver** the code, a preview (publish the page as an artifact or send the MP4) and the spec with its measured
   checks. If the user doesn't like the feel, change the personality or one row, not everything.

### Polish pass (every Director build, before the smoothness check)
What separates "animated" from "smooth and alive":
- **Overlap, don't queue:** the next element starts when the previous one is 50–70% done; nothing waits for a full stop.
- **Soft landings:** big entrances settle with 1–3% overshoot (e.g. cubic-bezier(.2,1.1,.9,1)); small ones without.
  Linear only for loops and constant spins.
- **Anticipation and follow-through on the hero:** a 2–4% pull-back before a big move; one small secondary motion
  after it lands (a dot bounces on the card's corner, a glow pulses once, a satellite snaps into place).
- **Motion blur on fast moves:** anything travelling more than ~40% of its own size per frame gets a directional blur
  (4–12 px) that clears as it settles. Text arrives from blur 8–12 px → 0, not only from opacity.
- **Never fully still:** while a beat holds, keep a slow drift (scale 1 → 1.02–1.04, or a 6–12 px rise over the
  hold) and ambient accents moving (background shapes turning 10–30° or floating over 6–12 s). The activity curve
  must not sit at zero for more than 0.6 s before the end.
- **Shape continuity:** when one idea becomes the next, morph the object (circle → rounded square → icon) instead of
  cutting.
- **Depth:** soft shadows under lifted elements; background accents lower-contrast or blurred so the focal point
  stays sharpest.

### Story mode: a motion piece from images (reel, explainer, product or social video)
Use when the user wants a video rather than a UI entrance: one image brought to life for 5–30 s, several images in
sequence, or images plus a script or voiceover. Default 1080×1920 at 30 fps for social, 1920×1080 for widescreen;
ask once if the platform is unclear. Never copy a reference piece's layout or art; use its lessons, encoded here.
1. **Beats.** One idea per beat (an image, a headline, a number, a chart). Each beat: **in** 0.3–0.6 s → **hold**
   long enough to read (≥ 0.4 s + 0.2 s per word; with voiceover, follow its timing and put each word on its spoken
   time) → **out/transition** 0.2–0.4 s. Typical beat 2–4 s.
2. **Entrances** (pick per beat; never the same one twice in a row):

| entrance | how | feel |
|---|---|---|
| grow from a line | scaleY 0 → 1 from a 2–4 px line; corners round off as it lands | ~0.5 s easeOutCubic, ~2% overshoot |
| word by word | each word from blur 10 px + 6–10 px rise; words ~0.2–0.3 s apart | 0.3–0.4 s per word, ease out |
| keyword sweep | the key phrase sharpens letter by letter from heavy blur, left to right, in a heavier, larger face | 0.6–0.9 s |
| mixed type | connective words in a light italic serif, key words in a bold sans or the brand colour | — |
| counter | the number counts up with a soft glow, then its label arrives word by word | 0.9–1.3 s easeOutExpo/Quart |
| scale from a dot | grows from a point, optionally turning in 3D (rotateY 30–60° → 0) | 0.5–0.8 s ease out |
| ring reveal | a dotted or thin ring draws around the subject while a circular clip opens on it | 0.5–0.8 s |
| data build | bars grow from the baseline staggered 60–100 ms, the trend line draws on, labels pop in ~0.4–0.6 s apart | 1–2 s total |
| fly in and snap | a small element arrives on a curve with motion blur and clicks into place | 0.3–0.4 s strong ease out |

3. **Transitions** (vary them; a hard cut only on a strong beat in the audio):

| transition | how | feel |
|---|---|---|
| zoom-through | the focal element scales up past the frame into the next scene | 0.2–0.3 s ease **in** (accelerating); the next scene lands with ease out |
| push | the old scene slides out one side while the next focal element enters from the other, blurred | 0.3–0.4 s, strong ease out (≈ .2,1.1,.9,1) |
| morph | the focal shape becomes the next beat's shape | 0.3–0.5 s easeInOut |
| match cut | cut on the same position, shape or colour | instant |
| drift-out | the beat drifts up 20–40 px and fades as the next fades up | ~0.3 s |

4. **Ambient layer:** 1–3 accent shapes (sparkles, arcs, blobs) in the brand colours that drift and turn slowly
   through the whole piece and change place or shape at transitions: they carry continuity across scenes.
5. **Build** in HTML + GSAP (or Remotion if the user works in it) as one timeline with a label per beat. The user's
   images are layers: cut the subject out when it sits on a plain background, otherwise use clip reveals and crops.
   Render frame-exact with the stepped clock (Director step 7) to MP4; see **Render notes** below.
6. **Check** with Director step 7 plus: a rise in the activity curve at every beat and no dead stretch > 0.6 s; every
   hold ≥ its reading time; no two transitions in a row of the same kind; `poses.png` (2 frames per second) reads as
   the story on its own.
7. **Director's review (at least one round):** look at `poses.png`, the timing sheets and `activity.png` like a motion
   designer and name the three weakest moments (a dead hold, a stiff entrance, an abrupt cut, two things competing
   for attention). Fix those, re-render, re-check. Report what changed.

### Edit mode: cut video clips into an ad (UGC, AI-generated such as Google Flow/Veo, or filmed)
Use when the user gives several clips (e.g. 10 × 8 s) and wants one finished ad. Default 1080×1920, 30 fps, 15–40 s;
ask once for the platform, the length and the call to action.
1. **Gather.** The clips in order (or a script of what each one says), logo, product shots, the offer/CTA, brand colours
   and fonts. Copy the clips into `./motion-match/<name>/clips/` (downloads: in their own folder). Probe each with
   `ffprobe` (size, fps, duration, audio yes/no) and normalise them all to one size and fps first (scale/crop, `fps=30`,
   audio 48 kHz).
2. **Break every clip down.** `ref_frames.py clips/NN.mp4 --out motion-match/<name>/clips/NN`, then `motion_compare.py`
   on it with `--no-align` → its moves and still stretches. Read every poses sheet. Log each clip in `clips.md`: what is
   said, the usable window (start–end s), the best 1–2 moments (a gesture, a product reveal, a reaction) and defects
   (warped hands, a face or outfit that changes, garbled text, a frozen or morphing start, speech cut off). For speech,
   note each line's start and end: from the user's script or timestamps, or a transcript if a speech-to-text tool
   installs (try `pip install faster-whisper` once; if its model can't download, ask for the script).
3. **Structure** (UGC ad):

| part | time | job |
|---|---|---|
| hook | 0–2 s | the strongest line or visual first (a bold claim, a problem, a reaction); its caption on screen by 0.2 s |
| problem | 2–6 s | one relatable pain |
| product | 6–15 s | the product in hand or on screen: the turn |
| proof | 15–25 s | results, before/after, another angle or voice |
| CTA | last 3–5 s | the offer and what to do; end card held ≥ 1.5 s |

   Scale the times to the length asked for. Skip each clip's first 0.3–0.5 s by default (AI clips often start frozen
   or morphing). Cut on speech boundaries, never mid-word.
4. **Edit rules.**
   - Shots average 1.5–3 s; no shot runs over 4 s without a punch-in (scale 1 → 1.12–1.2 on a cut, or a 0.25 s ease)
     or a graphic arriving.
   - Cut on motion: end a shot while something moves and enter the next mid-move (peaks on the activity curve); match
     cuts on position or gesture where two clips allow it.
   - J/L cuts: start the next line's audio 2–4 frames before its picture, or carry the last word over the cut.
   - Speed ramps only where nobody speaks (b-roll, a product reveal): 1 → 1.5–2× → 1, eased; speech stays at 1×.
   - Hide defects: punch in past a warped hand, cut away before a face changes, cover a glitch with a graphic.
   - Consistency: grade every clip toward the hero clip (ffmpeg `eq`/`colorbalance`), one caption style throughout.
   - Handheld shake (1–3 px, 2–3 Hz) only on a locked-off clip among handheld ones.
5. **Graphics layer** (Director rules): captions word by word on the spoken timing, 2–4 words per card, bold sans,
   keyword in the brand colour, popping in over 0.12–0.18 s with 4–6% overshoot, inside the 9:16 safe area (keep
   ~250 px clear at the top and ~350 px at the bottom for the platform's UI); product callouts (fly in and snap, a line
   drawing on to the product); an offer badge (scale from a dot + one pulse); the logo; an end card (Story-mode
   entrances and transitions). Write the cut as `edl.json` (per shot: clip, in, out, timeline start, speed, zoom) and
   build the graphics in HTML + GSAP on a transparent background against the same times.
6. **Render.**
   - Footage: cut with ffmpeg from `edl.json` (trim, `setpts` for speed, scale/crop for punch-ins), concatenated with
     2–4 frame audio crossfades.
   - Graphics: frame-exact with the stepped clock, page background `transparent`, screenshots with
     `omit_background=True` → a PNG sequence with alpha.
   - Composite: `ffmpeg -i edit.mp4 -framerate 30 -i gfx/%04d.png -filter_complex "[0][1]overlay=format=auto"`, speech
     loudness-normalised to about −14 LUFS (`loudnorm`).
7. **Check** the final with the Story-mode checks, plus: a cut or a graphic at least every 4 s; hook text visible by
   0.2 s; every caption on screen ≥ its reading time and within ±0.1 s of its first spoken word; nothing important in
   the platform UI zones; no cut mid-word (an RMS dip in the audio within ±2 frames of each cut); no clicks at cuts.
   Then the Director's review: the three weakest moments, fixed, re-rendered.
8. **Deliver** the ad MP4, a short note of which clips and moments were used and what was hidden, and the source
   (`edl.json`, the graphics HTML, the render script). Offer cut-downs (15 s, a 6 s hook) from the same `edl.json`.

### Render notes (Director, Story and Edit mode)
- The `seek()` that Playwright calls must return a plain value (the time), never the GSAP timeline: `page.evaluate`
  tries to serialise what it gets back and hangs.
- Headless Chrome: launch with `--disable-gpu --no-sandbox --disable-dev-shm-usage --allow-file-access-from-files`.
- Elements that fade up from blur only register in `track_motion` once clearly visible, so their measured start runs
  ~0.2–0.3 s after the tween's start: compare landing times, or write visible timings into the spec.
- A layer whose resting state is visible shows early (a sheen band parked inside its mask, a gradient at its start
  position): check frame 0 and the frame before each element's entrance.
- Iris and wipe reveals between different backgrounds: give the mask a soft edge (60–100 px) so the edge reads as
  light, not as a hole.

## Limits
- Edit mode can't change what is inside a generated clip (a face, an outfit, a hand, a word): it can only choose,
  crop, cover or cut around it. Speech that a clip cuts off can't be completed; ask for clips whose lines finish
  inside them.
- Director and Story mode choose taste-based motion from rules; it is "smooth and consistent" by measurement, but
  whether it fits the brand is the user's call: offer the other personalities rather than insisting.
- Rotation is measured in the image plane only; 3D turns are inferred from width and face changes, roughly.
- Morph checks compare silhouettes (outline and fill), not inner detail or colour inside the shape.
- 3D references: camera and object **motion** only, not models, materials or lighting.
- Colour tracking needs an element with distinct colours; on busy live-action footage rely on cuts, the activity curve
  and the sheets, and say the per-element numbers are rough.
- Compressed, low-frame-rate or tiny references limit precision; ask for a better source when it matters.
- Websites: works from a screen recording. To read a site's exact CSS/JS timings, ask the user to share the code
  or the declared values instead.