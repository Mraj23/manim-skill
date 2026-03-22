# Pacing, Timing & Scene Design

This is the guide most Manim tutorials skip. Getting the math to render is the easy part — making it *feel* like a well-paced explanation is what separates a confusing animation from an intuitive one.

---

## Table of Contents
1. [The Golden Rule of Pacing](#1-the-golden-rule-of-pacing)
2. [self.wait() — The Complete Reference](#2-selfwait--the-complete-reference)
3. [run_time — How Fast Animations Move](#3-run_time--how-fast-animations-move)
4. [Scene Structure — Start, Middle, End](#4-scene-structure--start-middle-end)
5. [The 3b1b Pacing Philosophy](#5-the-3b1b-pacing-philosophy)
6. [Annotated Pacing Example](#6-annotated-pacing-example)
7. [Scene Setup Patterns](#7-scene-setup-patterns)
8. [Multi-Scene Video Structure](#8-multi-scene-video-structure)
9. [Common Pacing Mistakes](#9-common-pacing-mistakes)
10. [Timing Reference Cheatsheet](#10-timing-reference-cheatsheet)

---

## 1. The Golden Rule of Pacing

> **Every time you show something new on screen, give the viewer time to see it before you move on.**

Manim beginners almost always write too little `self.wait()`. When *you* write the animation, you already know what's coming — so two seconds feels long. To a first-time viewer, two seconds is barely enough to register what just appeared.

The practical rule: **read the text or equation out loud at normal speed. Whatever that takes — that's your minimum wait time.**

---

## 2. self.wait() — The Complete Reference

```python
self.wait()          # Default: 1 second
self.wait(2)         # 2 seconds
self.wait(0.5)       # Half second — only for brief transitions
self.pause()         # Frozen frame (updaters suspended) — same as wait with frozen_frame=True
```

### Wait with a condition (advanced)
```python
# Wait until a condition is true, up to 5 seconds max
self.wait_until(lambda: tracker.get_value() > 3, max_time=5)
```

### When wait() freezes vs. keeps updating
- `self.wait()` with no active updaters → frozen frame (efficient, no render cost)
- `self.wait()` with active updaters → keeps running the render loop (e.g., a rotating object keeps rotating during the pause)
- `self.pause()` → always a frozen frame regardless of updaters

### Wait times by situation

| Situation | Recommended wait |
|---|---|
| Title card appears | `2.0 – 3.0s` |
| New equation or formula just written | `2.0 – 3.5s` |
| Simple shape appears | `0.5 – 1.0s` |
| Key insight or punchline moment | `3.0 – 4.0s` — let it breathe |
| Between animation steps in a sequence | `1.0 – 1.5s` |
| After a transform (viewer needs to process) | `1.5 – 2.0s` |
| After adding a label to something | `1.0s` |
| Quick housekeeping (repositioning) | `0.0 – 0.3s` |
| End of scene before fade-out | `2.0 – 3.0s` |
| After the final reveal | `3.0s+` |

---

## 3. run_time — How Fast Animations Move

`run_time` controls the duration of an animation in seconds. Default is **1 second** for most animations.

```python
self.play(Create(circle))                  # 1s (default)
self.play(Create(circle), run_time=2)      # 2s — slower, more deliberate
self.play(Create(circle), run_time=0.5)    # 0.5s — quick appearance
```

### run_time guidelines by animation type

| Animation | Typical run_time | Notes |
|---|---|---|
| `Write(short_text)` | `1.0 – 1.5s` | Scales with text length |
| `Write(long_equation)` | `2.0 – 3.0s` | Give time for each symbol |
| `Create(simple_shape)` | `0.5 – 1.0s` | |
| `Create(axes)` or `Create(grid)` | `1.5 – 2.5s` | Complex paths take longer |
| `Transform(a, b)` | `1.0 – 1.5s` | |
| `TransformMatchingTex` | `1.5 – 2.0s` | Many parts moving |
| `FadeIn / FadeOut` | `0.5 – 1.0s` | |
| `GrowFromCenter` | `0.5 – 1.0s` | |
| Moving camera (`animate.scale`) | `1.5 – 2.5s` | Slow camera = more professional |
| Tracing a curve via ValueTracker | `3.0 – 6.0s` | Viewers follow the dot |
| `LaggedStart` of many objects | `2.0 – 4.0s` | Spread out the entrance |
| `NumberPlane.apply_matrix` | `2.0 – 3.0s` | Space transforms need time |
| `FadeOut` closing a scene | `1.0 – 1.5s` | |

### Setting a default run_time for an animation class
```python
# Set default once at top of construct() to keep consistent timing
Write.set_default(run_time=1.5)
FadeIn.set_default(run_time=0.8)
```

---

## 4. Scene Structure — Start, Middle, End

Every good Manim scene has three beats. This isn't arbitrary — it mirrors how human short-term memory works.

### The Three Beats

```
SETUP (5–15% of total time)
  → Establish context. What are we looking at?
  → Introduce the objects. Simple, not cluttered.
  → Don't animate yet — build the stage first.

DEVELOPMENT (70–80%)
  → One idea at a time. Show it, wait, then continue.
  → Build complexity gradually — don't reveal everything at once.
  → Use emphasis (Indicate, Flash, SurroundingRectangle) to direct attention.
  → Pause after every significant step.

PAYOFF (10–20%)
  → The "aha" moment. The result. The punchline.
  → Give it more wait time than anything else.
  → Optionally: zoom in, highlight, or hold the final state.
  → Clean exit — FadeOut or clear screen before ending.
```

### Annotated scene skeleton

```python
class WellPacedScene(Scene):
    def construct(self):

        # ── SETUP ──────────────────────────────────────────────
        title = Text("The Pythagorean Theorem", font_size=52)
        self.play(Write(title), run_time=1.5)
        self.wait(2)                    # Let title breathe
        self.play(title.animate.to_edge(UP).scale(0.65))
        self.wait(0.5)                  # Short — repositioning, not new info

        # ── DEVELOPMENT ────────────────────────────────────────

        # Step 1: Introduce the triangle
        triangle = Polygon(ORIGIN, RIGHT*3, UP*4, color=WHITE)
        self.play(Create(triangle), run_time=1.5)
        self.wait(1.5)                  # Viewer registers the shape

        # Step 2: Label the sides
        a_label = MathTex("a").next_to(triangle, LEFT, buff=0.2)
        b_label = MathTex("b").next_to(triangle, DOWN, buff=0.2)
        c_label = MathTex("c").move_to(triangle.get_center() + RIGHT*1.5 + UP*0.5)
        self.play(Write(a_label), Write(b_label), Write(c_label), run_time=1.2)
        self.wait(2)                    # Three new labels — viewer needs time

        # Step 3: Show the squares on each side (builds complexity gradually)
        sq_a = Square(side_length=4, color=BLUE, fill_opacity=0.2)
        # ... (positioning logic)
        self.play(DrawBorderThenFill(sq_a), run_time=1.5)
        self.wait(1.5)

        # ── PAYOFF ─────────────────────────────────────────────
        theorem = MathTex(r"a^2 + b^2 = c^2", font_size=60, color=YELLOW)
        theorem.to_edge(DOWN, buff=1)
        self.play(Write(theorem), run_time=2.0)
        self.wait(3.5)                  # THE moment — maximum wait

        # Clean exit
        self.play(FadeOut(VGroup(*self.mobjects)), run_time=1.5)
```

---

## 5. The 3b1b Pacing Philosophy

These are Grant Sanderson's actual design principles, synthesized from his FAQ, October 2024 Substack post, and his open-source video code.

### "Concrete before abstract"

> When explaining new topics, put concrete and specific ideas before general and abstract frameworks. This runs contrary to how most of us start explaining things we already understand well.

In Manim terms: **show a specific example before showing the general formula.** Draw the triangle, *then* write `a² + b² = c²`. Animating the formula first and then illustrating it is backwards.

### "The animation should mirror the explanation"

Grant's code is unusually readable because the structure of the `construct()` method mirrors the structure of the narration. Each `self.play()` block corresponds to a sentence or clause in the voiceover. If you were to read the code out loud, it would sound like the video.

Practical rule: **write a sentence outline of your explanation first, then translate each sentence into a code block.**

### "Use pauses as punctuation"

`self.wait()` is the visual equivalent of a period or comma. Grant uses them constantly — sometimes 3–5 `self.wait()` calls per minute of video. The pauses aren't dead air; they're processing time.

A common pattern in his code:
```python
self.play(Write(eq))
self.wait(2)          # "Here's the equation."
self.play(Indicate(eq[2]))
self.wait(1)          # "Notice this term."
self.play(Transform(eq, eq_simplified))
self.wait(2.5)        # "Watch what happens."
```

### "One thing at a time"

Grant almost never introduces more than one new concept in a single `self.play()` call. Parallel animations (`self.play(A, B, C)`) are reserved for things that are *already established* — introducing three things simultaneously is visually chaotic.

```python
# ✅ Grant's style: sequential introduction
self.play(Create(circle))
self.wait(1)
self.play(Create(label))
self.wait(1)

# ❌ Avoid for new elements: parallel introduction
self.play(Create(circle), Write(label), Create(arrow))
```

### "Don't animate for animation's sake"

From his FAQ:
> Be sure to ask yourself if what you're doing benefits from being programmatic at all. If all you're looking to do is simple moving/fading animations, using something simple like Keynote or PowerPoint might take you farther than you'd expect.

Manim earns its complexity when: the code directly reflects the math, or when iteration/abstraction makes illustrations possible that would otherwise be impossibly tedious to do manually. If your animation is mostly text fades and bullet points, use a simpler tool.

### "Manim generates clips — edit in a video editor"

Manim outputs raw clips. Narration sync, music, chapter markers, and multi-scene concatenation all belong in a video editor (DaVinci Resolve, Premiere, Final Cut). Don't fight Manim to try to do these things in Python.

---

## 6. Annotated Pacing Example

A complete example showing pacing decisions at each step, with rationale in comments:

```python
from manim import *

class EulersFormula(Scene):
    def construct(self):

        # SETUP: title card
        title = MathTex(r"e^{i\pi} + 1 = 0", font_size=72)
        subtitle = Text("Euler's Identity", font_size=32, color=GREY_B)
        subtitle.next_to(title, DOWN, buff=0.4)

        self.play(Write(title), run_time=2.0)
        # 2s wait: this is the first thing the viewer sees — give them the moment
        self.wait(2.0)
        self.play(FadeIn(subtitle, shift=UP * 0.3), run_time=1.0)
        # 2.5s: subtitle adds context, viewer reads both title and subtitle
        self.wait(2.5)

        # Move title to top to make room for the explanation
        self.play(
            title.animate.to_edge(UP).scale(0.7),
            FadeOut(subtitle),
            run_time=1.2   # Fast — repositioning is not new information
        )
        # 0.5s: short pause after a housekeeping move
        self.wait(0.5)

        # DEVELOPMENT: introduce the unit circle
        circle = Circle(radius=2, color=BLUE)
        axes = Axes(x_range=[-3, 3, 1], y_range=[-3, 3, 1],
                    x_length=6, y_length=6, axis_config={"color": GREY})

        self.play(Create(axes), run_time=1.5)
        # 0.8s: axes are context, not the main idea — don't over-pause
        self.wait(0.8)
        self.play(Create(circle), run_time=2.0)
        # 1.5s: the unit circle is the main stage — viewer should register it
        self.wait(1.5)

        # Add the angle
        angle_t = ValueTracker(0)
        dot = always_redraw(lambda: Dot(
            point=np.array([2*np.cos(angle_t.get_value()),
                            2*np.sin(angle_t.get_value()), 0]),
            color=YELLOW, radius=0.1
        ))
        radius_line = always_redraw(lambda: Line(
            ORIGIN, dot.get_center(), color=YELLOW
        ))

        self.play(FadeIn(dot), Create(radius_line), run_time=0.8)
        self.wait(1.0)

        # Sweep the angle — this is the VISUAL CORE of the explanation
        # Long run_time: the viewer follows the dot as it traces the circle
        self.play(angle_t.animate.set_value(PI), run_time=4.0, rate_func=linear)
        # 1.5s: the sweep just finished — pause to let the geometry sink in
        self.wait(1.5)
        self.play(angle_t.animate.set_value(TAU), run_time=4.0, rate_func=linear)
        self.wait(2.0)

        # PAYOFF: connect the geometry to the formula
        connection = MathTex(
            r"e^{i\theta} = \cos\theta + i\sin\theta",
            font_size=44, color=WHITE
        ).to_edge(DOWN, buff=1.0)

        self.play(Write(connection), run_time=2.5)
        # 3.5s: this IS the payoff — maximum wait
        self.wait(3.5)

        # Final emphasis
        self.play(Indicate(connection, scale_value=1.15), run_time=1.0)
        self.wait(3.0)

        # Clean exit
        self.play(FadeOut(VGroup(*self.mobjects)), run_time=1.5)
```

---

## 7. Scene Setup Patterns

### Pattern A: Title-then-shrink (most common)
```python
title = Text("Topic Name", font_size=56)
self.play(Write(title))
self.wait(1.5)
self.play(title.animate.to_edge(UP).scale(0.65))
# Now there's room for the main content below
```

### Pattern B: Axes first, objects second
```python
ax = Axes(x_range=[-4, 4], y_range=[-3, 3])
labels = ax.get_axis_labels()
self.play(Create(ax), Write(labels), run_time=1.5)
self.wait(0.5)
# Now plot curves, points, etc. on top
```

### Pattern C: Build complexity in layers
```python
# Add the scaffolding without animation (instant, no visual noise)
self.add(background_grid)

# Then reveal the main object
self.play(Create(main_curve), run_time=2)
self.wait(1.5)

# Then add annotations on top
self.play(Write(label), run_time=1)
self.wait(1)
```

### Pattern D: Highlight-then-evolve
```python
# Introduce the equation
self.play(Write(eq))
self.wait(2)

# Draw attention to a specific part before transforming
box = SurroundingRectangle(eq[2], color=YELLOW)
self.play(Create(box))
self.wait(1.5)                        # "Look at THIS term"

# Now transform — viewer's eye is already on the right place
self.play(FadeOut(box), TransformMatchingTex(eq, eq_new))
self.wait(2)
```

### Pattern E: Clear and restart (for multi-part explanations)
```python
# End of Part 1
self.play(FadeOut(VGroup(*self.mobjects)), run_time=1.0)
self.wait(0.5)

# Part 2 begins clean
new_title = Text("Part 2: Why This Works")
self.play(FadeIn(new_title))
self.wait(1.5)
```

---

## 8. Multi-Scene Video Structure

For longer explanations, split into multiple Scene classes rather than one giant `construct()`. Render them separately and concatenate in a video editor.

```python
class Introduction(Scene):
    """~30 seconds. Hook + what we'll cover."""
    def construct(self): ...

class ConcreteExample(Scene):
    """~90 seconds. A specific, tangible case."""
    def construct(self): ...

class GeneralPattern(Scene):
    """~60 seconds. Abstract the pattern."""
    def construct(self): ...

class Proof(Scene):
    """~120 seconds. The rigorous version."""
    def construct(self): ...

class Conclusion(Scene):
    """~20 seconds. Recap + punchline."""
    def construct(self): ...
```

### Using next_section() for chapter markers
```python
def construct(self):
    self.next_section("Introduction")
    # ...

    self.next_section("Main Derivation")
    # ...

    self.next_section("Conclusion")
    # ...

# Render with sections saved:
# manim --save_sections scene.py MyScene
```

---

## 9. Common Pacing Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| **No waits** | Animation feels like a slideshow on fast-forward | Add `self.wait(1–2)` after every `self.play()` |
| **Too many things at once** | Viewer doesn't know where to look | Introduce objects one at a time |
| **Abstract before concrete** | Viewers are lost before the example appears | Show the specific case first, then generalize |
| **Uniform run_times** | Everything feels mechanical, robotic | Use 0.5s for minor moves, 3s for key reveals |
| **No payoff pause** | Climactic moment rushes past | `self.wait(3.5)` at minimum after the key result |
| **Frenetic camera** | MovingCamera scenes feel nauseating | Camera movements: `run_time=2+`, smooth rate_func |
| **Walls of LaTeX** | Screen fills with equations, viewer loses track | Reveal one term at a time; use `TransformMatchingTex` |
| **Forgetting to FadeOut** | Scene ends abruptly mid-screen | End every scene with a clean fade or wait |
| **Over-animating** | Simple points get elaborate entrances | `self.add()` for context; reserve `self.play()` for ideas |

---

## 10. Timing Reference Cheatsheet

```
WAITS                               RUN_TIMES
─────────────────────────           ─────────────────────────────────
Title card first appears:  2.5s     Writing text (short):       1.0s
Equation just appeared:    2.0s     Writing text (equation):    1.5–2.5s
After a transform:         1.5s     Creating a shape:           0.8–1.5s
After adding a label:      1.0s     Transform:                  1.0–1.5s
After key insight:         3.5s     TransformMatchingTex:       1.5–2.0s
Quick repositioning:       0.3s     FadeIn / FadeOut:           0.5–1.0s
Before fade-out:           2.0s     Tracing a curve:            3.0–6.0s
End of scene hold:         3.0s     Camera move (pan/zoom):     1.5–2.5s
                                    LaggedStart (many objs):    2.0–4.0s
                                    NumberPlane transform:      2.0–3.0s
                                    Rotating (full turn):       3.0–4.0s
```

When in doubt: **go slower**. You can always speed up a clip in your video editor, but you can't add understanding time that was never there.

---

*Rule of thumb from watching hundreds of 3b1b videos: the average wait between meaningful visual events is about 1.8 seconds. If your scene averages under 1 second, it's almost certainly too fast.*

---

## Official Documentation

- [Scene reference](https://docs.manim.community/en/stable/reference/manim.scene.scene.Scene.html) — self.wait(), self.pause(), next_section(), all scene methods
- [Output settings](https://docs.manim.community/en/stable/tutorials/output_and_config.html) — --save_sections for chapter-based rendering
- [3b1b FAQ on animations](https://www.3blue1brown.com/faq) — Grant's own philosophy on when to use Manim and when not to
