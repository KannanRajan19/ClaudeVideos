# AGENTS.md: launch-video generator

This repo turns app screenshots into a motion-graphics product launch video (1920×1080, 30 fps, about 78 s, H.264 + AAC) with an original synthesized soundtrack.

**Before doing anything, read `docs/VIDEO_PLAYBOOK.md` in full.** It is the spec. Use its numbers (colours, sizes, timings, easings, audio levels) as given; do not substitute defaults.

## Non-negotiables

- **Real screens only.** Every UI pixel comes from `screens/*.png`. Never redraw, retype or generate UI. Overlays sit on top.
- **Code-driven and deterministic.** Use Python (`numpy`, `opencv-python`, `pillow`, `scipy`, `imageio-ffmpeg`) with a pure `render(t)` function, piped into ffmpeg. Do not use text-to-video models, moviepy `TextClip`, or screen recordings.
- **No hard-coded pixel positions in scene code.** Positions are named regions (fractions 0–1) in `config/screens.json`. All on-screen words live in `config/storyboard.json`.
- **Bundled fonts only** (`fonts/*.ttf`, Inter + Playfair Display, OFL). Never rely on system fonts.
- **No admin installs.** ffmpeg is resolved from `FFMPEG` env → PATH → `imageio_ffmpeg.get_ffmpeg_exe()`.
- **Never commit `screens/` or `output/`.** They contain internal company data and are git-ignored.
- **Copy must be true.** Use only facts visible in the screenshots or given by the user. Don't write "Now live" unless it is. No names, IDs, record numbers or customer data in overlay text.

## If the engine already exists (`launchvideo/engine.py`, `music.py`)

Reuse it. For a new app, change `config/screens.json`, `config/storyboard.json` and `launchvideo/scenes.py` (timeline, choreography, `build_cues()`). Touch the engine only for genuinely new components.

## Workflow (stop and show the user at each ★)

1. Build `tools/annotate.html` (the region and annotation editor, playbook §4) and pass its headless acceptance test. Run `python -m launchvideo check`. ★ Show the region overlays and flag low-resolution captures (dialogs < 1000 px wide, pages < 1600 px).
2. Write the storyboard and timing table. ★ Get approval before animating.
3. Build or adjust scenes; render three full-size stills (intro, a lift-out, a transition midpoint). ★
4. `python -m launchvideo preview` (20–24 key frames). Fix issues. ★
5. `python -m launchvideo music`; plot the waveform against the scene boundaries. ★
6. `python -m launchvideo render --max-mb 18`; ffprobe both files; inspect frames pulled from the encoded MP4. ★ Deliver.

## Verification is mandatory

After every visual change, render the affected frames and **look at them**. Before claiming success, describe what each checked frame shows: overlaps, edge clipping, card leaning toward its text, regions on the right UI element, text sharpness at 1:1. "The code ran" is not verification. Follow playbook §15 and the §17 checklist.
