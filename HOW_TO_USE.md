# How to use the launch-video playbook (for you, not the model)

## Setup

| Situation | What to do |
|---|---|
| **Same style, different app (fastest, best quality)** | Put `AGENTS.md` in the repo root and `VIDEO_PLAYBOOK.md` in `docs/` of the `clr-launch-video` repo. Codex will reuse the engine and only rewrite the scenes and the two config files. |
| **No repo, or a new style** | Give the model `VIDEO_PLAYBOOK.md` and the screenshots. It builds the engine from the spec. Expect 3–5 review rounds. |
| **Codex (CLI, IDE or desktop)** | Codex loads `AGENTS.md` automatically. That file tells it to read the playbook, so you don't have to paste anything. |
| **ChatGPT in the browser** | Attach `VIDEO_PLAYBOOK.md` and the screenshots. Use it for code and single frames; run the full render (10–20 min) on your PC. |

## Taking the screenshots

- **Tool and format:** Snagit or Win+Shift+S. Save as **PNG**, never JPG.
- **Full pages:** browser at 100% zoom. Crop to the app content only: no browser tabs, address bar or taskbar. Aim for ≥ 1900 px wide.
- **Dialogs and side panels:** press Ctrl + until the browser is at about 200%, then capture just the dialog. Aim for ≥ 1200 px wide; at 100% they come out too small and look soft in the video.
- **Matching layouts:** frame pages that share a layout (e.g. a list page and its comments page) identically.
- **Show the state you want:** open the dialog, hover the tooltip, scroll the key row into view.
- **Where they go:** save them in `screens/` with the file names listed in `config/screens.json`.

The model checks each screenshot against these rules and tells you which ones to re-capture.

## Prompts

**Kickoff:**
> Read AGENTS.md and docs/VIDEO_PLAYBOOK.md fully. Screenshots are in screens/. Product: <name> (<one-line description>). Wordmark: "<SERIF WORD>" + "<light word>", subtitle "<SUBTITLE>". Features in story order: 1) <screen>: <what it shows> … Flow between screens: <screen A> → click <element> → <screen B> … Audience: internal, shared by email. End line: "Coming soon". Credit: "BUILT BY THE <TEAM>". Do milestone 1 only, then stop and show me the check images.

**Storyboard approval:**
> Write storyboard.json and a timing table. Follow the chapter grammar in §5. Headlines ≤ 16 characters per line, subs ≤ 34. Use only facts visible in the screenshots. List any claims I need to confirm. Stop after this.

**Revisions:**
> In chapter 03 the lift-out covers the headline. Move its target to x ≈ 700 and re-render the preview at 25.2 and 27.2. Show me the frames.

> The history dialog looks soft. Tell me its source width and the enlargement factor in the lift-out, then suggest the capture zoom I should use.

> Render frames at the transition midpoints and confirm each transition starts on the clicked element.

**Final:**
> Run the full render with --max-mb 18. ffprobe both files, pull 6 frames from the encoded copy, check them against §15 and §17, and report.

## Tips

- **Approve one milestone at a time:** regions, then words, then three stills, then the preview sheet, then the full render. Most bad results come from letting it run end-to-end.
- **Make it show you frames.** If it says "done" without showing frames, ask: "Render the preview sheet and describe each frame."
- **Use the strongest coding model you have.** This is a long, multi-file task.
