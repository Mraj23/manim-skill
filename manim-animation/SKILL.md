---
name: manim-animation
description: "Creates Manim animation scenes in Python - the same library used by 3Blue1Brown. Use this skill whenever the user asks to create a Manim animation, write a Manim scene, animate math concepts, visualize equations or algorithms, generate an educational video with code, create a .py file for Manim, or says anything like make an animation of X, animate this concept, help me use Manim, write a Manim scene, make a 3Blue1Brown-style animation, or visualize this mathematically. Always trigger for any request involving Manim, manimce, manimgl, or programmatic math animation in Python."
compatibility: "Requires Python 3.8+, manim (pip install manim), FFmpeg, and optionally LaTeX for equation rendering. Target: ManimCE (Community Edition) v0.18+. Not ManimGL."
---

# Manim Animation Skill

You are an expert at writing Manim (Mathematical Animation Engine) code using the **ManimCE (Community Edition)**. You produce clean, well-commented, runnable Python scenes.

> For deep reference on specific topics, see `references/`:
> - **`references/pacing.md`** — Timing, waits, scene structure, 3b1b philosophy ← read this first for any educational video
> - `references/production-patterns.md` — Multi-scene projects: shared style module, scene decomposition, state persistence, column layout, debug workflow, GIF export ← read this for any project with more than one scene
> - `references/mobjects.md` — Mobject types, geometry, text, LaTeX, graphing
> - `references/animations.md` — Animation classes, rate functions, updaters, ValueTracker
> - `references/advanced.md` — 3D scenes, cameras, custom animations, 3b1b patterns
> - `references/debugging.md` — debug workflow, frame extraction, common errors, visual checklist

---

## The Two Versions of Manim — Always Clarify

There are two incompatible versions:

| Version | Package | Import | Who uses it |
|---|---|---|---|
| **ManimCE** ← use this | `pip install manim` | `from manim import *` | Community, beginners, most tutorials |
| ManimGL | `pip install manimgl` | `from manimlib import *` | Grant Sanderson (3b1b) personally |

**Always write ManimCE code unless the user explicitly asks for ManimGL.** ManimCE is better tested, better documented, and what most people have installed.

---

## Core Architecture — Know This Before Writing Any Code

Every Manim program has three building blocks:

### 1. Scene
The canvas. All code lives inside a `Scene` subclass's `construct()` method.

```python
from manim import *

class MyScene(Scene):
    def construct(self):
        # All animation code goes here
        pass
```

### 2. Mobjects (Mathematical Objects)
Everything visible on screen is a Mobject. Key types:

```python
# Geometry
circle = Circle(radius=1, color=BLUE)
square = Square(side_length=2, color=RED)
line = Line(LEFT, RIGHT)
arrow = Arrow(ORIGIN, UP)
dot = Dot(point=ORIGIN, color=YELLOW)

# Text & Math
text = Text("Hello World", font_size=48)
tex = Tex(r"This is \LaTeX")           # LaTeX (normal mode)
math = MathTex(r"\int_0^\infty e^{-x} dx")  # LaTeX (math mode)

# Grouping
group = VGroup(circle, square)   # Vector group — arrange together
group.arrange(RIGHT, buff=0.5)   # Space them out
```

### 3. Animations
What happens to mobjects over time.

```python
self.play(Create(circle))              # Draw it
self.play(Write(text))                 # Write text stroke by stroke
self.play(FadeIn(square))              # Fade in
self.play(Transform(circle, square))  # Morph one into another
self.play(circle.animate.shift(RIGHT)) # Move it
self.wait(1)                           # Pause for 1 second
```

---

## Positioning System

Manim uses a coordinate system where the screen is roughly ±7 wide, ±4 tall.

```python
# Directions (constants)
UP, DOWN, LEFT, RIGHT, ORIGIN
UL, UR, DL, DR          # diagonals (Up-Left, etc.)
IN, OUT                  # 3D only

# Positioning methods
obj.move_to(ORIGIN)                  # absolute position
obj.shift(RIGHT * 2)                 # relative shift
obj.to_edge(LEFT)                    # snap to screen edge
obj.to_corner(UL)                    # snap to screen corner
obj.next_to(other_obj, RIGHT, buff=0.3)  # place relative to another

# Alignment
obj.align_to(other_obj, LEFT)        # align edges
```

---

## The `.animate` Syntax — Key Pattern

Prepend `.animate` to any method to turn it into an animation:

```python
# Instant (not animated):
square.shift(RIGHT)
square.set_fill(BLUE, opacity=0.5)

# Animated (plays over time):
self.play(square.animate.shift(RIGHT))
self.play(square.animate.set_fill(BLUE, opacity=0.5))
self.play(square.animate.shift(RIGHT).scale(2))  # chain multiple
```

---

## Running Animations in Parallel vs. Series

```python
# Series — one after another:
self.play(FadeIn(circle))
self.play(FadeIn(square))

# Parallel — at the same time:
self.play(FadeIn(circle), FadeIn(square))

# Staggered — same animation, offset start times:
self.play(LaggedStart(
    FadeIn(obj1), FadeIn(obj2), FadeIn(obj3),
    lag_ratio=0.3
))

# Different durations in the same play call:
self.play(
    FadeIn(circle, run_time=2),
    FadeIn(square, run_time=0.5),
)
```

---

## Rate Functions — Control Easing

```python
# Add rate_func to any self.play() call:
self.play(circle.animate.shift(RIGHT), rate_func=smooth)      # default S-curve
self.play(circle.animate.shift(RIGHT), rate_func=linear)      # constant speed
self.play(circle.animate.shift(RIGHT), rate_func=ease_in_out_cubic)
self.play(circle.animate.shift(RIGHT), rate_func=there_and_back)  # yo-yo
self.play(circle.animate.shift(RIGHT), rate_func=rush_from)   # starts fast
self.play(circle.animate.shift(RIGHT), rate_func=rush_into)   # ends fast
```

---

## ValueTracker + Updaters — Dynamic Animations

The most powerful pattern for smooth, parameter-driven animations. A `ValueTracker` holds a number that can be animated; updaters are functions that run every frame.

```python
class DynamicExample(Scene):
    def construct(self):
        tracker = ValueTracker(0)

        # always_redraw redraws the object from scratch every frame
        dot = always_redraw(lambda: Dot(
            point=np.array([tracker.get_value(), 0, 0]),
            color=BLUE
        ))

        label = always_redraw(lambda: MathTex(
            f"x = {tracker.get_value():.2f}"
        ).next_to(dot, UP))

        self.add(dot, label)
        self.play(tracker.animate.set_value(3), run_time=2)
        self.play(tracker.animate.set_value(-2), run_time=2)
        self.wait()
```

**When to use each approach:**

| Situation | Use |
|---|---|
| Simple one-shot motion | `.animate` syntax |
| Object that continuously tracks something | `always_redraw()` |
| Multiple objects sharing one parameter | `ValueTracker` + `add_updater()` |
| Object that follows another | `add_updater(lambda m: m.next_to(...))` |

---

## LaTeX / Math Equations

Always use raw strings (`r"..."`) for LaTeX:

```python
# Inline LaTeX
equation = MathTex(r"\frac{d}{dx} e^x = e^x")

# Multi-part equations (for TransformMatchingTex)
eq1 = MathTex(r"e^x = ", r"\sum_{n=0}^\infty", r"\frac{x^n}{n!}")

# Highlighting individual parts by index
equation[0].set_color(YELLOW)     # first term
equation[1].set_color(BLUE)       # second term

# Morphing between equations — parts that share substrings stay put
self.play(TransformMatchingTex(eq1, eq2))

# Color specific substrings
eq = MathTex(r"f(x) = x^2", substrings_to_isolate=["x"])
eq.set_color_by_tex("x", RED)
```

---

## Graphing & Plots

```python
class GraphExample(Scene):
    def construct(self):
        ax = Axes(
            x_range=[-3, 3, 1],   # [min, max, step]
            y_range=[-2, 5, 1],
            axis_config={"color": BLUE},
            tips=True,
        )
        labels = ax.get_axis_labels(x_label="x", y_label="y")

        curve = ax.plot(lambda x: x**2, color=RED, x_range=[-2, 2])
        area = ax.get_area(curve, x_range=[-1, 1], color=YELLOW, opacity=0.3)

        self.play(Create(ax), Write(labels))
        self.play(Create(curve))
        self.play(FadeIn(area))
```

---

## Workflow: How to Write a Scene

1. **Plan the narrative first** — Write a one-sentence-per-step outline. What does the viewer see? What should they understand after each beat? Each sentence becomes a code block.
2. **Concrete before abstract** — Show a specific example before the general formula. This is Grant Sanderson's most important pedagogical rule.
3. **One thing at a time** — Introduce objects sequentially. Reserve parallel `self.play(A, B, C)` for things that are already established on screen.
4. **Sketch the layout** — Where do objects start? Where do they move? Draw it on paper if needed.
5. **Write from top to bottom** — Create objects, add them to scene, animate them.
6. **Add waits generously** — The single most common beginner mistake is not enough `self.wait()`. After every `self.play()`, ask: "does the viewer need time to absorb this?" Default answer is yes.
7. **Clean up** — `FadeOut(VGroup(*self.mobjects))` to clear at end.

### Pacing Quick Reference

```python
# After a title or equation appears
self.wait(2.0)      # minimum; 2.5–3s for key results

# After repositioning (housekeeping)
self.wait(0.3)      # brief — not new information

# After the main payoff/punchline
self.wait(3.5)      # let it breathe — this is the moment

# Between steps in a sequence
self.wait(1.0)      # standard beat

# run_time defaults to use:
self.play(Write(equation), run_time=2.0)     # equations
self.play(Create(shape), run_time=1.0)       # shapes
self.play(Transform(a, b), run_time=1.5)     # transforms
self.play(FadeIn(obj), run_time=0.8)         # fades
# Tracing a curve: run_time=3–6s — viewers follow the dot
```

> **Full pacing guide:** See `references/pacing.md` for wait time tables, run_time guidelines by animation type, scene structure patterns, the 3b1b philosophy in detail, and a fully annotated example scene.

**Template for every scene:**

```python
from manim import *

class DescriptiveName(Scene):
    def construct(self):
        # --- Title / Setup ---
        title = Text("Topic Name", font_size=56)
        self.play(Write(title))
        self.wait(1)
        self.play(title.animate.to_edge(UP).scale(0.6))

        # --- Main content ---
        # ... your animation logic ...

        self.wait(2)
```

---

## CLI Commands — How to Render

```bash
# Low quality (fast, for testing) — 480p 15fps
manim -ql scene.py SceneName

# Medium quality — 720p 30fps
manim -qm scene.py SceneName

# High quality — 1080p 60fps
manim -qh scene.py SceneName

# 4K
manim -qk scene.py SceneName

# Preview immediately after rendering
manim -ql -p scene.py SceneName

# Save as GIF
manim -ql --format=gif scene.py SceneName

# Render a still image (no animation)
manim -s scene.py SceneName

# Jupyter notebook
%%manim -ql SceneName
```

---

## The 3Blue1Brown Philosophy (Grant Sanderson's Design Principles)

Grant Sanderson (3b1b) shared his philosophy directly. Apply these principles when generating animations:

1. **Use Manim when the code mirrors the math.** Programmatic animation earns its complexity when iteration, abstraction, and conditionals make illustrations possible that would otherwise be manual. For simple text slides or basic shape movements, suggest simpler tools (Keynote, PowerPoint).

2. **Separate animation from narration.** Manim generates clips. Use a video editor (Premiere, DaVinci) to sync voiceover. Don't try to time animations to audio inside Manim itself.

3. **Favor smooth parameter-driven motion.** The signature 3b1b aesthetic comes from `ValueTracker` + `always_redraw` — smooth, continuous transitions that feel mathematical rather than discrete.

4. **Tell a story with transformations.** `TransformMatchingTex` for equations, `ReplacementTransform` for shapes — the visual continuity between states carries the explanation.

5. **Use color deliberately.** 3b1b's palette: `BLUE`, `RED`, `YELLOW`, `GREEN` for key elements; muted greys/whites for supporting structure. Highlight specific parts to draw attention.

6. **Grant's workflow (ManimGL-specific but the principle applies):** Develop interactively — render small chunks, check them, iterate. Don't write 200 lines and render the whole thing blind.

---

## Color Palette Reference

```python
# Primary colors
BLUE, RED, GREEN, YELLOW, ORANGE, PURPLE, PINK, WHITE, BLACK

# Shades (DARK → LIGHT: _E, _D, _C, _B, _A)
BLUE_E, BLUE_D, BLUE_C, BLUE_B, BLUE_A
RED_E, RED_D, ...

# Special
TEAL, MAROON, GOLD, GREY

# Hex
color = ManimColor("#FF6B6B")
```

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| LaTeX fails to render | Missing LaTeX install | `pip install manim[latex]` or install TeX Live |
| Animation looks choppy | Using `.shift()` in updater without `dt` | Use `ValueTracker` + `always_redraw` instead |
| Object appears behind another | Z-ordering | Use `.set_z_index(n)` to control layering |
| `Transform` warps unexpectedly | Mismatched point counts | Use `ReplacementTransform` or `FadeTransform` |
| Text too small | Default font size | Set `font_size=48` or larger explicitly |
| `always` not updating with `ValueTracker` | Wrong approach | Use `add_updater()` not `always()` with ValueTrackers |
| Rendering is very slow | High quality flag | Use `-ql` for development, `-qh` only for final |

---

## Output Structure

When generating Manim code, always:
1. Produce a **complete, runnable `.py` file** — include `from manim import *`
2. Name the class **descriptively** (e.g., `FourierTransformExplainer`, not `Scene1`)
3. Add **comments** explaining each section
4. Include the **render command** the user should run
5. Note any **dependencies** (LaTeX required? specific packages?)

For complex requests, read `references/animations.md` for the full animation class reference and `references/advanced.md` for 3D scenes and camera work.

---

## Visual Review Loop — Render, Inspect, Fix

After writing a scene, verify it visually before calling it done. The workflow is: render → extract frames → read them as images → fix issues → repeat.

### Quick layout check (fastest — ~1 second)

```bash
# Render a still image of the final frame
manim -ql -s -p scene.py MyScene
```

Read the resulting PNG as an image and check positioning, overlap, and text size.

### Full animation check

```bash
# Render at low quality
manim -ql scene.py MyScene

# Extract frames at key moments (adjust timestamps to match your scene duration)
ffmpeg -ss 0    -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_start.png
ffmpeg -ss 2.5  -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_mid.png
ffmpeg -sseof -0.1 -i media/videos/scene/480p15/MyScene.mp4 -frames:v 1 frame_end.png
```

Read each PNG as an image and check: nothing clipped off-screen, LaTeX rendered correctly, colors match the request, animation progression makes sense, math is correct.

### Fix and repeat

Apply fixes, re-render with `-ql`, extract frames again. Once everything looks correct:

```bash
manim -qh scene.py MyScene   # final high quality render
```

> **Full debugging guide:** See `references/debugging.md` for the complete debug workflow, render flags, ffmpeg recipes, common errors and fixes, and a full visual checklist.
