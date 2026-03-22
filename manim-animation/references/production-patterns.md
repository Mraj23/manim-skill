# Production Patterns

These patterns are extracted from real multi-scene Manim projects. Apply them
when writing anything beyond a single short animation.

## Table of Contents
1. [Project Structure — Shared Style Module](#1-project-structure--shared-style-module)
2. [Scene Decomposition — `self.scene_*()` Methods](#2-scene-decomposition--self-scene_-methods)
3. [State Persistence Between Scenes](#3-state-persistence-between-scenes)
4. [Column Layout with Precomputed X-Positions](#4-column-layout-with-precomputed-x-positions)
5. [Reusable Helper Methods](#5-reusable-helper-methods)
6. [The Lookup-Table Highlight Pattern](#6-the-lookup-table-highlight-pattern)
7. [Pipeline Boxes with Arrows](#7-pipeline-boxes-with-arrows)
8. [Growing Bar Chart](#8-growing-bar-chart)
9. [Debug Workflow — Static Layouts First](#9-debug-workflow--static-layouts-first)
10. [Clearing the Scene](#10-clearing-the-scene)
11. [LaTeX Alignment with `\phantom`](#11-latex-alignment-with-phantom)
12. [GIF Export](#12-gif-export)
13. [Determinism](#13-determinism)

---

## 1. Project Structure — Shared Style Module

Any project with more than one scene file should have a shared module that
sets global config and defines named color constants. This ensures visual
consistency and means you only change colors in one place.

```python
# manim_common.py — copy and customize for each project

from manim import *
import numpy as np

np.random.seed(42)  # deterministic layouts

# --- Global config (set once here, imported everywhere) ---
BACKGROUND_COLOR = "#0a0a0a"
config.background_color = BACKGROUND_COLOR
config.pixel_height = 1080
config.pixel_width = 1920
config.frame_rate = 60

# --- Named color palette ---
PRIMARY_BLUE   = "#3b82f6"
PRIMARY_GREEN  = "#10b981"
PRIMARY_ORANGE = "#f59e0b"
PRIMARY_RED    = "#ef4444"
PRIMARY_YELLOW = "#fbbf24"
PRIMARY_PURPLE = "#8b5cf6"
PRIMARY_GRAY   = "#404040"

# --- Shared helpers ---
def make_title(text: str, subtitle: str | None = None) -> VGroup:
    title = Text(text, font_size=48, color=WHITE).to_edge(UP)
    if subtitle:
        sub = Text(subtitle, font_size=28, color=GRAY_B).next_to(title, DOWN, buff=0.2)
        return VGroup(title, sub)
    return VGroup(title)
```

In each scene file:
```python
from manim import *
from manim_common import PRIMARY_BLUE, PRIMARY_GREEN, make_title
# config is already set by the import — no need to set it again
```

**Why this matters:** Without it, every scene file re-declares the same hex
codes. One typo creates a visual inconsistency that's hard to track down.
Config settings like `pixel_height` and `frame_rate` need to be consistent
across all files or Manim silently produces different-sized outputs.

---

## 2. Scene Decomposition — `self.scene_*()` Methods

For animations with multiple logical beats, break `construct()` into named
`self.scene_*()` methods. Each method handles one conceptual step.

```python
class TokenEmbeddingsAnimation(Scene):
    def construct(self):
        self.scene_string_to_tokens()   # "emma" → character boxes
        self.scene_token_ids()          # chars → integer IDs via vocab
        self.scene_embeddings()         # IDs → embedding vectors
        self.scene_position_embeddings() # add positional info

    def scene_string_to_tokens(self):
        title = make_title("Tokenization", "How a string becomes numbers")
        self.play(FadeIn(title), run_time=1.0)
        self.wait(1.5)
        # ...

    def scene_token_ids(self):
        # ...
```

Benefits:
- `construct()` reads like a table of contents — the story is immediately clear
- Each method can be developed and tested in isolation
- Avoids a 300-line monolithic `construct()` that's impossible to navigate
- Closely mirrors how Grant Sanderson structures his own video code

**When to use it:** Any scene longer than ~60 lines of animation code, or
anything with more than two distinct conceptual phases.

---

## 3. State Persistence Between Scenes

When objects from one `self.scene_*()` method need to survive into the next,
store them on `self` at the end of the method. Consume and clean them up in
the next one.

```python
def scene_embeddings(self):
    # ... build and animate things ...

    # Store whatever the next scene needs
    self.emb_labels = emb_labels        # the embedding vectors on screen
    self.emb_char_labels = char_labels  # character labels at top
    self.emb_m_box = m_box              # highlight box to remove next

def scene_position_embeddings(self):
    # Clean up what's no longer needed
    self.play(
        FadeOut(self.emb_m_box),
        FadeOut(self.emb_same_note),
        FadeOut(self.emb_problem_note),
        FadeOut(self.emb_title),
        run_time=0.6,
    )
    # Now use what was kept
    for emb_lbl, x in zip(self.emb_labels, COL_XS):
        plus = MathTex(r"+").next_to(emb_lbl, DOWN, buff=0.35)
        # ...
```

**Rules:**
- Name attributes clearly: `self.emb_labels` not `self.labels`
- Only store what the *next* scene actually uses — don't accumulate everything
- Always `FadeOut` or `self.remove()` stale objects at the start of the consuming scene — don't let them silently persist

---

## 4. Column Layout with Precomputed X-Positions

When you have multiple columns of objects that need to align across rows
(characters, IDs, embedding vectors, position vectors all in the same column),
precompute x-positions as a module-level constant and reference them everywhere.

```python
# At module level — outside the class
COL_XS = [0.45, 2.15, 3.85, 5.55]  # 4 columns, evenly spaced

class MyScene(Scene):
    def construct(self):
        chars = ["e", "m", "m", "a"]

        # Row 1: character labels
        char_labels = VGroup()
        for ch, x in zip(chars, COL_XS):
            lbl = Text(ch, font_size=48, color=WHITE)
            lbl.move_to(UP * 1.9 + RIGHT * x)
            char_labels.add(lbl)

        # Row 2: ID labels — same x positions, different y
        id_labels = VGroup()
        for tid, x in zip([4, 12, 12, 0], COL_XS):
            lbl = MathTex(rf"\text{{id}} = {tid}", font_size=24)
            lbl.move_to(UP * 1.25 + RIGHT * x)
            id_labels.add(lbl)

        # Row 3: embedding vectors — same x, lower y
        for tex, x in zip(emb_tex, COL_XS):
            t = MathTex(tex, font_size=24)
            t.move_to(UP * -0.3 + RIGHT * x)
```

This is far more reliable than chaining `.next_to()` calls across rows, which
accumulates positioning errors. It also makes it trivial to add a new row or
adjust all columns at once by editing one list.

---

## 5. Reusable Helper Methods

Extract any pattern you use more than once into a `_helper()` method on the
scene class. Prefix with `_` to indicate it's internal.

**Token row helper:**
```python
def _make_token_row(self, tokens, spacing=0.25):
    """Horizontal row of rounded-rectangle token boxes."""
    boxes = VGroup()
    for tok in tokens:
        is_bos = tok == "BOS"
        rect = RoundedRectangle(
            width=1.3 if not is_bos else 1.5,
            height=0.8,
            corner_radius=0.1,
            fill_color=PRIMARY_GRAY if not is_bos else "#1a1a2e",
            fill_opacity=1,
            stroke_color=PRIMARY_BLUE if not is_bos else PRIMARY_PURPLE,
            stroke_width=2.5,
        )
        lbl = Text(tok, font_size=38 if not is_bos else 26, color=WHITE)
        lbl.move_to(rect)
        boxes.add(VGroup(rect, lbl))
    boxes.arrange(RIGHT, buff=spacing)
    return boxes
```

**Pipeline stage box helper:**
```python
def _make_stage_box(self, text, width=5.0, color=PRIMARY_GRAY, height=0.75):
    """Consistent pipeline stage box with label."""
    rect = RoundedRectangle(
        width=width, height=height,
        corner_radius=0.1,
        fill_color=color, fill_opacity=0.9,
        stroke_color=WHITE, stroke_width=1.5,
    )
    lbl = Text(text, font_size=20, color=WHITE)
    lbl.move_to(rect)
    return VGroup(rect, lbl)
```

**Pulse utility (from shared module):**
```python
def pulse(mobj, color=PRIMARY_YELLOW, scale=1.08, run_time=0.35):
    """Flash an object to draw attention, then restore it."""
    return Succession(
        mobj.animate.set_color(color).scale(scale),
        mobj.animate.scale(1 / scale).set_color(WHITE),
        run_time=run_time,
    )

# Usage:
self.play(pulse(key_label))
```

---

## 6. The Lookup-Table Highlight Pattern

A clean way to show a lookup: display a table on the left, then for each
element being looked up, briefly highlight its row with a `SurroundingRectangle`
while the result appears on the right. Then fade the rectangle.

```python
# wte_rows is the VGroup of MathTex rows in the table
# wte_row_indices maps each token to its row in the table
wte_row_indices = [3, 5, 5, 0]  # e→row3, m→row5, m→row5, a→row0

for i in range(4):
    row_idx = wte_row_indices[i]
    # Flash the source row
    hi = SurroundingRectangle(wte_rows[row_idx], color=PRIMARY_YELLOW, buff=0.03)
    self.play(Create(hi), run_time=0.3)
    # Reveal the result
    self.play(FadeIn(emb_labels[i], shift=UP * 0.1), run_time=0.4)
    # Remove highlight
    self.play(FadeOut(hi), run_time=0.2)
```

The short rhythm (0.3 / 0.4 / 0.2) creates a crisp "look here → result appears
→ move on" cadence without dragging. Use it any time you're showing one-to-one
correspondence between a source and a result.

---

## 7. Pipeline Boxes with Arrows

Build a vertical (or horizontal) pipeline of stage boxes by creating them,
arranging them as a `VGroup`, then adding connecting arrows between adjacent
pairs in a loop.

```python
# Create all stage boxes
stage_input       = self._make_stage_box("Input: token 'e' at position 1", color=PRIMARY_GRAY)
stage_embed       = self._make_stage_box("Token Emb + Position Emb",        color=EMBED_COLOR)
stage_transformer = self._make_stage_box("Transformer Block\n(Attention + MLP)", color=MODEL_COLOR, height=1.1)
stage_output      = self._make_stage_box("Output: 27 logits",                color=OUTPUT_COLOR)

# Arrange vertically
pipeline = VGroup(stage_input, stage_embed, stage_transformer, stage_output)
pipeline.arrange(DOWN, buff=0.35)
pipeline.next_to(title, DOWN, buff=0.45)

# Add arrows between adjacent stages
arrows = VGroup()
for i in range(len(pipeline) - 1):
    a = Arrow(
        pipeline[i].get_bottom(), pipeline[i + 1].get_top(),
        buff=0.08, stroke_width=2.5, color=GRAY_A,
    )
    arrows.add(a)

# Add dimension annotations to the right of each box
dims = VGroup(
    Text("id: 4",      font_size=18, color=GRAY_B),
    Text("16D vector", font_size=18, color=GRAY_B),
    Text("16D vector", font_size=18, color=GRAY_B),
    Text("27 logits",  font_size=18, color=GRAY_B),
)
for d, s in zip(dims, pipeline):
    d.next_to(s, RIGHT, buff=0.15)

# Animate building the pipeline one stage at a time
for i in range(len(pipeline)):
    anims = [FadeIn(pipeline[i]), FadeIn(dims[i])]
    if i > 0:
        anims.append(Create(arrows[i - 1]))
    self.play(*anims, run_time=0.7)
    self.wait(0.6)
```

Key details: the arrow loop runs `len(pipeline) - 1` times; the arrow for
stage `i` is added at the same time as stage `i` appears (not before), so the
arrow and its destination box arrive together.

---

## 8. Growing Bar Chart

Animate bars growing from zero height rather than appearing fully formed.
The technique: save the target state with `.copy()`, squash all bars to near-zero
height, add them to the scene, then animate each bar `.become()`-ing its target.

```python
# Build bars at their final heights
bars = VGroup()
for i, (tok, prob) in enumerate(zip(tokens, probs)):
    bh = max(0.03, prob * max_bar_height / max_prob)
    is_max = (tok == "m")
    bar = Rectangle(
        width=bar_width * 0.8,
        height=bh,
        fill_color=PRIMARY_GREEN if is_max else PRIMARY_BLUE,
        fill_opacity=0.95 if is_max else 0.7,
        stroke_width=1.5 if is_max else 0.5,
    )
    bar.move_to(np.array([x_position, bh / 2, 0]))  # anchor at bottom
    bars.add(bar)

# Save target state before modifying
bars_target = bars.copy()

# Squash to near-zero (must happen before self.add/self.play)
for bar in bars:
    bar.stretch(0.01, dim=1, about_edge=DOWN)

# Show axis and labels first
self.play(FadeIn(axis), FadeIn(bar_labels), run_time=0.5)

# Instant FadeIn of squashed bars (invisible at height ~0)
self.play(FadeIn(bars), run_time=0.15)

# Grow to full height
self.play(
    *[bar.animate.become(target) for bar, target in zip(bars, bars_target)],
    run_time=1.0,
)

# Label the max bar
pct_label = MathTex(r"42\%", font_size=22, color=PRIMARY_GREEN)
pct_label.next_to(bars[max_idx], UP, buff=0.08)
self.play(FadeIn(pct_label), run_time=0.4)
```

Note: bars are anchored at `bh / 2` on the y-axis so their bottom edge sits on
`y=0`. The `about_edge=DOWN` in `stretch()` keeps the bottom pinned while
squashing the top — this is what makes the grow animation go upward.

---

## 9. Debug Workflow — Static Layouts First

Before writing any `self.play()` animations, verify your layout is correct by
building a static debug scene that uses `self.add()` only. Render it with
`manim -s` (still image, renders in ~1 second) to check positioning.

```python
class DebugMyLayout(Scene):
    """Static snapshot — use manim -s to check layout before animating."""
    def construct(self):
        title = make_title("My Scene Title")
        self.add(title)                          # instant, no animation

        tok_boxes = _make_token_row(["e", "m", "m", "a"])
        tok_boxes.move_to(LEFT * 1.5 + UP * 2.2)
        self.add(tok_boxes)

        # Add every element you plan to animate — check nothing overlaps,
        # clips the screen edge, or sits in unexpected positions
        output_box = RoundedRectangle(width=1.4, height=0.8)
        output_box.move_to(RIGHT * 5.5 + DOWN * 1.0)
        self.add(output_box)
```

Render:
```bash
manim -s debug_scenes.py DebugMyLayout
# Opens the PNG immediately — check layout in ~1s instead of waiting for full render
```

**Workflow:**
1. Write the debug scene with `self.add()` for every element
2. Run `manim -s` — instant PNG
3. Fix positioning issues
4. Only then write the animated version with `self.play()`

This is the ManimCE equivalent of Grant Sanderson's interactive `checkpoint_paste()`
workflow. You're checking the scene state before committing to a full render.

Organize debug scenes in a separate file (`debug_scenes.py`) so they don't
clutter your production files. Keep them — they're useful when you revisit
a scene months later.

---

## 10. Clearing the Scene

**Option A — fade everything:**
```python
self.play(*[FadeOut(m) for m in self.mobjects], run_time=1.0)
```

**Option B — instant clear (between major sections):**
```python
self.clear()
```

**Option C — selective fade (keep some things):**
```python
# Fade everything except title
self.play(
    *[FadeOut(m) for m in self.mobjects if m is not title],
    run_time=0.8
)
```

Prefer Option A for scene transitions the viewer will see. Option B is fine
between `self.scene_*()` methods when there's already a title transition
covering the cut. Never let objects from a previous scene silently persist
into the next one — it causes confusing visual artifacts.

---

## 11. LaTeX Alignment with `\phantom`

When displaying multiple equations or vectors in a column, values with and
without minus signs won't align because `-` takes up space. Use `\phantom{-}`
to insert invisible space where the minus would be:

```python
# Misaligned (minus signs throw off column alignment):
r"[0.23, -0.45]"
r"[0.61,  0.24]"   # this won't line up

# Aligned with \phantom{-}:
r"[0.23,\;-0.45]"
r"[0.61,\;\phantom{-}0.24]"   # invisible space matches the width of '-'
```

Also use `\;` (medium space) for manual spacing inside MathTex when values
feel cramped. These two tricks together make vector/matrix displays look
professional rather than haphazard.

---

## 12. GIF Export

Two-pass FFmpeg GIF using `palettegen`/`paletteuse` — significantly better
color quality than naive GIF conversion:

```bash
# Step 1: generate an optimized palette from the video
ffmpeg -i media/videos/scene/480p15/MyScene.mp4 \
  -vf "fps=18,scale=960:-1:flags=lanczos,palettegen" \
  /tmp/palette.png

# Step 2: apply the palette to produce the GIF
ffmpeg -i media/videos/scene/480p15/MyScene.mp4 \
  -i /tmp/palette.png \
  -lavfi "fps=18,scale=960:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=sierra2_4a" \
  -loop 0 \
  MyScene.gif
```

Adjust `fps=18` and `scale=960` to taste. Lower fps reduces file size;
`scale=960` is half of 1920px and renders well on most screens.

---

## 13. Determinism

Always set `np.random.seed(42)` at module level (before any class definitions)
when your scene uses random values for positions, colors, or data. Without it,
re-rendering produces different layouts, which breaks the debug workflow and
makes visual review comparisons meaningless.

```python
import numpy as np
np.random.seed(42)   # put this right after imports, before config and classes
```

---

## Official Documentation

- [Plugins](https://docs.manim.community/en/stable/guides/using_plugins.html) — extending Manim with third-party packages
- [Configuration](https://docs.manim.community/en/stable/guides/configuration.html) — all config fields, custom_config.yml
- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skill repo with production examples (docx, pptx, pdf skills show similar patterns)
- [3b1b/videos](https://github.com/3b1b/videos) — real multi-scene project code organized by year
