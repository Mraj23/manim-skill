# Animations Reference

## Table of Contents
1. [Animation Classes Cheatsheet](#1-animation-classes-cheatsheet)
2. [Updaters In Depth](#2-updaters-in-depth)
3. [ValueTracker Patterns](#3-valuetracker-patterns)
4. [Rate Functions](#4-rate-functions)
5. [AnimationGroup and Sequencing](#5-animationgroup-and-sequencing)
6. [Custom Animations](#6-custom-animations)

---

## 1. Animation Classes Cheatsheet

### Creation Animations
```python
Create(mobject)            # draw VMobject (traces the path)
Write(text)                # write text stroke by stroke
DrawBorderThenFill(shape)  # outline first, then fill
Uncreate(mobject)          # reverse of Create
Unwrite(text)              # reverse of Write
```

### Fade Animations
```python
FadeIn(mobject)
FadeOut(mobject)
FadeIn(mobject, shift=UP)          # fade in while shifting
FadeOut(mobject, scale=2)          # fade out while scaling up
FadeTransform(mob1, mob2)          # fade one into another
FadeTransformPieces(mob1, mob2)    # fade submobject by submobject
```

### Transform Animations
```python
Transform(mob1, mob2)              # mob1 becomes mob2 (mob1 survives)
ReplacementTransform(mob1, mob2)   # mob1 becomes mob2 (mob1 removed)
TransformFromCopy(mob1, mob2)      # keep mob1, animate a copy → mob2
TransformMatchingTex(tex1, tex2)   # LaTeX-aware: matched parts stay still
TransformMatchingShapes(mob1, mob2)# shape-aware matching
ClockwiseTransform(mob1, mob2)
CounterclockwiseTransform(mob1, mob2)
CyclicReplace(mob1, mob2, mob3)    # rotate through objects
```

### Movement Animations
```python
mob.animate.shift(direction)       # preferred — use .animate syntax
mob.animate.move_to(point)
mob.animate.rotate(angle)
mob.animate.scale(factor)
MoveAlongPath(mob, path)           # follow a VMobject path
MoveToTarget(mob)                  # move to previously set target
Rotate(mob, angle=PI)              # rotation animation
```

### Growing / Shrinking
```python
GrowFromCenter(mobject)
GrowFromPoint(mobject, point)
GrowFromEdge(mobject, edge)
GrowArrow(arrow)
SpinInFromNothing(mobject)
ShrinkToCenter(mobject)
```

### Indication / Emphasis
```python
Indicate(mobject)                  # wiggle + brighten
Flash(point, color=YELLOW)         # burst of lines radiating out
Circumscribe(mobject)              # draw box around it
ShowPassingFlash(mobject)          # flash passes along path
Wiggle(mobject)                    # wobble
FocusOn(point)                     # spotlight dot
ApplyWave(mobject)                 # ripple effect
```

### Scene-Level Animations
```python
self.wait(duration=1)              # hold still
self.play(..., run_time=2)         # set duration of any animation
self.add(mobject)                  # instantly add (no animation)
self.remove(mobject)               # instantly remove
self.clear()                       # remove all mobjects
```

---

## 2. Updaters In Depth

Updaters are functions that run **every frame**, letting you create continuous reactive behavior.

### Basic Updater Syntax

```python
# Style 1: lambda (concise)
label.add_updater(lambda m: m.next_to(dot, UP))

# Style 2: named function (clearer for complex logic)
def update_label(mob):
    mob.next_to(dot, UP)
    mob.set_color(RED if dot.get_x() > 0 else BLUE)
label.add_updater(update_label)

# Remove updaters
label.clear_updaters()
label.remove_updater(update_label)

# Temporarily suspend updaters
with self.no_updaters():
    self.play(...)
```

### Updaters with dt (time delta)

Pass a second argument `dt` to access elapsed time per frame — essential for framerate-independent continuous motion:

```python
# BAD — framerate-dependent (jittery at different fps)
dot.add_updater(lambda x: x.shift(RIGHT * 0.1))

# GOOD — framerate-independent via dt
dot.add_updater(lambda x, dt: x.shift(RIGHT * dt))

# Continuously rotating
square.add_updater(lambda mob, dt: mob.rotate(PI * dt))
```

### always_redraw Pattern (3b1b style)

Rebuilds the object from scratch each frame — cleaner than add_updater for objects that depend on a parameter:

```python
tracker = ValueTracker(0)

# This arrow always points from origin to dot's current position
dot = always_redraw(lambda: Dot(
    point=np.array([np.cos(tracker.get_value()), np.sin(tracker.get_value()), 0])
))

arrow = always_redraw(lambda: Arrow(ORIGIN, dot.get_center(), buff=0))

self.add(dot, arrow)
self.play(tracker.animate.set_value(TAU), run_time=4, rate_func=linear)
```

### Updater on Brace (common pattern)

```python
line = Line(LEFT * 2, RIGHT * 2)
brace = always_redraw(lambda: Brace(line, DOWN))
text = always_redraw(lambda: brace.get_tex(f"length = {line.get_length():.1f}"))
self.add(line, brace, text)
self.play(line.animate.scale(2))
```

---

## 3. ValueTracker Patterns

`ValueTracker` is an invisible mobject that stores a number and can be animated smoothly.

### Basic Pattern

```python
t = ValueTracker(0)

# Animate the value changing
self.play(t.animate.set_value(5))          # animate to 5
self.play(t.animate.increment_value(2))   # animate +2

# Read current value (use inside updaters/always_redraw)
current = t.get_value()

# Non-animated instant change (outside self.play)
t.set_value(3)
t += 1.5                    # shorthand
t -= 0.5
```

### Multi-tracker Example: Parametric Curve

```python
class ParametricDemo(Scene):
    def construct(self):
        ax = Axes(x_range=[-3, 3], y_range=[-3, 3])
        t = ValueTracker(0)

        dot = always_redraw(lambda: Dot(
            ax.coords_to_point(np.cos(t.get_value()), np.sin(t.get_value())),
            color=RED
        ))
        path = TracedPath(dot.get_center, stroke_color=YELLOW, stroke_width=2)

        self.add(ax, path, dot)
        self.play(t.animate.set_value(TAU * 2), run_time=6, rate_func=linear)
```

### Linking Multiple Objects to One Tracker

```python
angle_t = ValueTracker(0)

# Both objects react to the same tracker
vector = always_redraw(lambda: Vector(
    direction=np.array([np.cos(angle_t.get_value()), np.sin(angle_t.get_value()), 0])
))
angle_label = always_redraw(lambda: MathTex(
    rf"\theta = {np.degrees(angle_t.get_value()):.0f}°"
).to_corner(UR))

self.add(vector, angle_label)
self.play(angle_t.animate.set_value(PI), run_time=3)
```

---

## 4. Rate Functions

Control the speed profile (easing) of an animation. Add `rate_func=...` to any `self.play()`.

```python
# Standard easing
smooth              # default S-curve (ease in + out)
linear              # constant speed
ease_in_sine        # starts slow
ease_out_sine       # ends slow
ease_in_out_sine
ease_in_cubic
ease_out_cubic
ease_in_out_cubic
ease_in_quart
ease_in_expo
ease_out_expo
ease_in_out_expo
ease_in_back        # overshoots at start
ease_out_back       # overshoots at end
ease_in_bounce
ease_out_bounce

# Manim-specific
there_and_back      # go forward then reverse
there_and_back_with_pause  # go, pause, come back
running_start       # slight reverse before going
rush_from           # start fast, slow down
rush_into           # start slow, end fast
wiggle              # oscillates around position
double_smooth       # double S-curve
not_quite_there(rate_func=smooth, proportion=0.7)  # stops short of target
```

---

## 5. AnimationGroup and Sequencing

```python
# Play multiple animations at the same time
self.play(
    FadeIn(circle),
    FadeIn(square),
    run_time=2
)

# AnimationGroup — group into a single reusable animation
group_anim = AnimationGroup(
    FadeIn(circle),
    FadeIn(square),
    lag_ratio=0.0   # 0.0 = simultaneous, 1.0 = sequential
)

# LaggedStart — staggered start times (3b1b signature look)
self.play(LaggedStart(
    *[FadeIn(dot) for dot in dots],
    lag_ratio=0.1,   # each starts 10% of total duration after the previous
    run_time=3
))

# Succession — sequential with no overlap
self.play(Succession(
    FadeIn(obj1),
    FadeIn(obj2),
    FadeIn(obj3),
))
```

---

## 6. Custom Animations

For when built-in animations don't cover your case:

```python
class CountUp(Animation):
    """Animates a DecimalNumber counting from start to end."""
    def __init__(self, number: DecimalNumber, start: float, end: float, **kwargs):
        super().__init__(number, **kwargs)
        self.start = start
        self.end = end

    def interpolate_mobject(self, alpha: float) -> None:
        # alpha goes 0 → 1 over the animation duration
        # Apply rate_func manually for proper easing:
        alpha = self.rate_func(alpha)
        value = self.start + alpha * (self.end - self.start)
        self.mobject.set_value(value)

# Usage:
number = DecimalNumber(0).scale(3)
self.add(number)
self.play(CountUp(number, 0, 100), run_time=3, rate_func=smooth)
```

---

## Official Documentation

- [Animation reference](https://docs.manim.community/en/stable/reference/manim.animation.html) — every animation class
- [Rate functions reference](https://docs.manim.community/en/stable/reference/manim.utils.rate_functions.html) — all easing curves with visual previews
- [Updaters guide](https://docs.manim.community/en/stable/guides/using_updaters.html) — official guide to add_updater and always_redraw
- [Building blocks tutorial](https://docs.manim.community/en/stable/tutorials/building_blocks.html) — Scenes, Mobjects, Animations explained
- [Example Gallery](https://docs.manim.community/en/stable/examples.html) — working code for ValueTracker, MovingCamera, and more
