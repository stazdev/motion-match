# Motion Match

A Claude skill that turns "make it move like this" into numbers. Give it a reference video, GIF or screen recording of any animation, and it measures the motion of every element:
- every cut;
- each element's start, duration, easing, overshoot, travel, rotation and shape change.

It rebuilds that motion in your own design, then compares the two frame by frame until the timing matches within **±0.08 s**.

## Modes

- **Match mode (reference in → same motion out).**
  1. Break down the reference.
  2. Measure it.
  3. Write a motion spec, where every row becomes a check.
  4. Build it in your stack.
  5. Measure the build the same way.
  6. Compare and fix (2-5 rounds).
  7. Report with evidence.
- **Director mode (a still image in → motion designed for it).** For UI entrances: Claude designs the motion, builds it, then measures its own render with the same scripts, so "smooth" is checked, not claimed.
- **Story mode (images → a motion piece).** For reels, explainers, product and social videos built from your images.
- **Edit mode (video clips → an ad).** Cuts UGC, AI-generated clips (Google Flow, Veo) or filmed footage into an ad with captions and motion graphics, using the same measuring and review.

## Rebuilds in

CSS, the Web Animations API, Framer Motion, GSAP, Remotion, SwiftUI, and After Effects (as values to key in).

## What you can build with it

- **Copied landing-page motion:** a hero section that animates in exactly like a site you admire, rebuilt in your brand.
- **Logo and title reveals:** matched to a reference sting, with measured easing curves.
- **App and UI transitions:** a button press, modal entrance or tab switch matched to a screen recording.
- **Social reels and explainers:** made from still images, with directed camera moves and timed captions.
- **Ads from clips:** UGC or AI-generated clips cut to a hook-problem-solution structure with captions.
- **A motion audit:** "does my animation feel like theirs?", answered with activity curves and a side-by-side sheet.

## Installing

**Claude apps:** zip `skills/motion-match` and upload it under *Customize → Skills* (or add this folder as a plugin).

**Claude Code:**

```bash
cp -R skills/motion-match ~/.claude/skills/
```

## Requirements

- **ffmpeg/ffprobe** on your PATH.
- **Python 3** with `numpy` and `Pillow`. The skill can create its own venv in `~/.motion-match/venv` if you agree.
- **Director, Story and Edit modes:** Playwright with Chromium, for headless page renders (it asks once).

Nothing is uploaded anywhere: all measuring happens locally.

## How to use it

```text
Here's a screen recording of a pricing card animation (ref.mov). Rebuild it in Framer Motion with my colours.
```

```text
Make this still of my app's home screen animate in: logo, then cards staggering up, then the button.
```

```text
Cut these 6 Google Flow clips into a 20-second ad with captions and a hook in the first second.
```

Work goes into `./motion-match/<name>/`:
- `ref/`: the reference breakdown;
- `ours/`: the breakdown of our render;
- `compare-N/`: each comparison round;
- `spec.md`: the motion spec.

## Repository layout

```
skills/motion-match/
  SKILL.md          workflow and modes
  NOTICE.md         credits
  scripts/
    ref_frames.py      extract frames and cuts from a video/GIF
    track_motion.py    track elements and measure position, scale, rotation, colour over time
    fit_easing.py      fit easing curves to measured motion
    motion_compare.py  compare reference vs ours (activity curves, side-by-side sheets)
```

## Credits

The methods are adapted from NiazMorshed2007/reanimate and rossark0/rossark-motion-skill (both MIT). The scripts are an independent implementation.
