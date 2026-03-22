# Debugging Manim Scenes

## Table of Contents
1. [The Debug Workflow](#1-the-debug-workflow)
2. [Reading Manim Error Output](#2-reading-manim-error-output)
3. [Layout Debugging — Still Images First](#3-layout-debugging--still-images-first)
4. [Frame Extraction — Inspecting Specific Moments](#4-frame-extraction--inspecting-specific-moments)
5. [Render Flags for Faster Iteration](#5-render-flags-for-faster-iteration)
6. [Common Errors and Fixes](#6-common-errors-and-fixes)
7. [Visual Checklist — What to Look For](#7-visual-checklist--what-to-look-for)

---

## 1. The Debug Workflow

Always go in this order — each step is faster than the one after it:

```
1. Still image (-s)         ~1 second   Check layout and positioning
2. Low quality video (-ql)  ~5-15s      Check animation flow
3. Extract frames (ffmpeg)  ~1 second   Inspect specific moments closely
4. Medium quality (-qm)     ~30-60s     Final check before high quality
5. High quality (-qh)       ~2-5 min    Final render only
```

Never render `-qh` during development. Fix everything at `-ql` first.

---

## 2. Reading Manim Error Output

Manim errors fall into three categories:

**Python errors** — standard tracebacks, fix the code normally.

**LaTeX errors** — the most cryptic. Manim compiles LaTeX in a temp directory
and the actual error is buried:
```
# Look for lines like:
! Undefined control sequence
! Missing $ inserted
# The line number refers to the generated .tex file, not your Python file.
# Check your MathTex/Tex string for typos — missing backslash, unclosed brace, etc.
```
Quick fix: simplify the LaTeX string to the smallest version that reproduces
the error, then build back up.

**FFmpeg errors** — usually permissions or missing output directory. Manim
creates `media/` automatically but will fail if it can't write there.

**Cairo/rendering errors** — often caused by objects with zero or negative
dimensions (e.g. a Rectangle with height=0). Check that all size parameters
are positive.

---

## 3. Layout Debugging — Still Images First

Render a static snapshot of any scene state using `-s`. This is the fastest
possible feedback loop — about 1 second.

```bash
# Render the last frame of the scene as a PNG
manim -ql -s scene.py MyScene

# Output: media/images/scene/MyScene_ManimCE_v<version>.png
# Opens automatically with -p flag:
manim -ql -s -p scene.py MyScene
```

For complex layouts, build a dedicated debug scene that uses `self.add()`
instead of `self.play()` — every object appears instantly, no animation needed:

```python
class DebugLayout(Scene):
    """Run with: manim -s scene.py DebugLayout"""
    def construct(self):
        # Add every element at its intended final position
        # Use self.add(), not self.play() — renders in ~1 second
        title = make_title("My Scene")
        self.add(title)

        boxes = make_token_row(["e", "m", "m", "a"])
        boxes.move_to(LEFT * 2 + UP * 1.5)
        self.add(boxes)

        # Add a reference grid to check alignment (remove when done)
        self.add(NumberPlane(background_line_style={"stroke_opacity": 0.15}))
```

The `NumberPlane` grid trick is especially useful — it shows you the exact
coordinate of every object relative to ORIGIN.

---

## 4. Frame Extraction — Inspecting Specific Moments

Once you have a video, use ffmpeg to pull frames at specific timestamps and
inspect them as images.

```bash
# Extract a single frame at 2.5 seconds into the video
ffmpeg -ss 2.5 -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame.png

# Extract the first frame (t=0)
ffmpeg -ss 0 -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_start.png

# Extract the last frame
ffmpeg -sseof -0.1 -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_end.png

# Extract 5 evenly-spaced frames from a 10-second video (t=0,2,4,6,8)
for t in 0 2 4 6 8; do
  ffmpeg -ss $t -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_t${t}.png
done
```

After extracting frames, read them as images and check against the
visual checklist in section 7.

**Finding the video path:** Manim's output path follows this pattern:
```
media/videos/<python_file_stem>/<quality>/MyScene.mp4

# Examples:
media/videos/scene/480p15/MyScene.mp4      # -ql
media/videos/scene/720p30/MyScene.mp4      # -qm
media/videos/scene/1080p60/MyScene.mp4     # -qh
```

---

## 5. Render Flags for Faster Iteration

```bash
# The most useful flags during development:

-ql          # 480p 15fps — use for all development
-s           # Still image only — fastest possible check
-p           # Preview (open file when done)
-n 5,10      # Only render frames 5 through 10 — great for checking a specific moment
--disable_caching   # Force re-render (use when TracedPath gives stale results)
-a           # Render all Scene classes in the file

# Combine them:
manim -ql -p scene.py MyScene          # render + open preview
manim -ql -s -p scene.py MyScene       # still image + open it
manim -ql -n 30,60 scene.py MyScene    # only frames 30-60 (1 second at 15fps)
```

The `-n` flag is underused and extremely valuable. If you know a bug is in the
middle of a long animation, render only those frames instead of waiting for the
whole thing.

---

## 6. Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `LaTeX Error: File not found` | LaTeX not installed | `pip install manim[latex]` or install TeX Live |
| `Undefined control sequence` | Typo in LaTeX string | Check backslashes — `\frac` not `frac` |
| `Missing $ inserted` | Math outside math mode | Use `MathTex` not `Tex` for equations |
| Objects overlap unexpectedly | `.next_to()` chain drift | Switch to absolute `COL_XS` coordinate positioning |
| Object renders behind another | Z-order conflict | `.set_z_index(2)` on the object that should be in front |
| `Transform` morphs strangely | Mismatched point counts | Use `ReplacementTransform` or `FadeTransform` instead |
| `always_redraw` not updating | Used with `always()` not `add_updater()` | Replace `always()` with `always_redraw(lambda: ...)` |
| Animation stutters at exact frame | Updater not using `dt` | Add `dt` param: `lambda m, dt: m.shift(RIGHT * dt)` |
| TracedPath shows stale path | Caching bug | Run with `--disable_caching` |
| `AttributeError` on `self.emb_labels` | Scene method ran out of order | Check `construct()` calls methods in the right sequence |
| Black/blank frame at start | Object added before it exists | Move `self.add()` calls after object creation |
| Text clipped at screen edge | Object too wide | `.scale_to_fit_width(config.frame_width - 1)` |
| LaTeX renders as boxes | Font/encoding issue | Use `MathTex` for all math; `Text` for plain prose only |
| Slow re-renders on unchanged scene | Cache miss | Normal — cache warms after first render |

---

## 7. Visual Checklist — What to Look For

When inspecting frames extracted from a render, check each of these
systematically. Work through them in order — layout issues at the start frame
are the most common and fastest to fix.

**First frame (t=0):**
- [ ] No objects visible that shouldn't be there yet
- [ ] Title positioned correctly at top
- [ ] Background color is correct (not default black if you set a custom one)

**Layout (all frames):**
- [ ] Nothing extends outside the frame boundary (±7 wide, ±4 tall)
- [ ] No two objects overlapping unintentionally
- [ ] Text is large enough to read (font_size ≥ 24 for body, ≥ 36 for labels)
- [ ] Consistent spacing between elements (use `buff=` not manual shifts)

**Text and equations:**
- [ ] LaTeX rendered correctly — no missing symbols, no garbled output
- [ ] `MathTex` used for equations, `Text` for prose (not mixed up)
- [ ] Multi-column values aligned with `\phantom{-}` where needed
- [ ] Labels don't overlap the objects they're labeling

**Colors and style:**
- [ ] Key elements use the project's named color constants
- [ ] Highlight colors (yellow, orange) used sparingly — only for emphasis
- [ ] Fill opacity not accidentally 0 or 1 when a partial fill was intended

**Mid-animation frames:**
- [ ] Objects moving smoothly — no sudden jumps between frames
- [ ] `ValueTracker`-driven objects updating continuously
- [ ] No objects from a previous scene lingering on screen

**Final frame:**
- [ ] Scene ends in a clean state — either faded out or at the intended resting position
- [ ] No partially-completed animations frozen mid-way
- [ ] The key result/insight is still visible if the scene doesn't fade out

---

## Official Documentation

- [FAQ](https://docs.manim.community/en/stable/faq/index.html) — installation issues, LaTeX problems, common errors
- [Output settings](https://docs.manim.community/en/stable/tutorials/output_and_config.html) — all CLI flags including `-n`, `--disable_caching`, `--save_sections`
- [Quickstart](https://docs.manim.community/en/stable/tutorials/quickstart.html) — if something fundamental isn't working, start here
- [GitHub Issues](https://github.com/ManimCommunity/manim/issues) — search before filing; most bugs are already documented
- [Discord](https://www.manim.community/discord) — active community, good for installation help
