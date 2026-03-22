# Advanced Manim Reference

## Table of Contents
1. [Camera Control (2D)](#1-camera-control-2d)
2. [3D Scenes](#2-3d-scenes)
3. [Scene Lifecycle & Setup](#3-scene-lifecycle--setup)
4. [3Blue1Brown Signature Patterns](#4-3blue1brown-signature-patterns)
5. [Performance Tips](#5-performance-tips)
6. [Coordinate System Deep Dive](#6-coordinate-system-deep-dive)

---

## 1. Camera Control (2D)

### MovingCameraScene

Use instead of `Scene` when you need to pan, zoom, or frame-follow:

```python
class CameraDemo(MovingCameraScene):
    def construct(self):
        # Save initial state to restore later
        self.camera.frame.save_state()

        # Zoom in on a point
        self.play(self.camera.frame.animate.scale(0.5).move_to(RIGHT * 2))

        # Follow a moving object
        dot = Dot(ORIGIN, color=RED)
        self.camera.frame.add_updater(lambda m: m.move_to(dot))
        self.play(dot.animate.move_to(RIGHT * 5), run_time=3)
        self.camera.frame.clear_updaters()

        # Restore original view
        self.play(Restore(self.camera.frame))

        # Zoom to fit a group of objects
        group = VGroup(Circle(), Square().shift(RIGHT * 3))
        self.play(self.camera.frame.animate.set_width(group.width + 1).move_to(group))
```

### ZoomedScene

Show a zoomed-in inset:

```python
class ZoomExample(ZoomedScene):
    def __init__(self, **kwargs):
        ZoomedScene.__init__(
            self,
            zoom_factor=0.3,
            zoomed_display_height=2,
            zoomed_display_width=4,
            **kwargs
        )

    def construct(self):
        dot = Dot(ORIGIN)
        self.add(dot)
        self.activate_zooming(animate=True)
        self.zoomed_camera.frame.move_to(dot)
        self.wait()
```

---

## 2. 3D Scenes

### ThreeDScene Basics

```python
class ThreeDDemo(ThreeDScene):
    def construct(self):
        # Set initial camera orientation
        self.set_camera_orientation(
            phi=75 * DEGREES,    # polar angle from top (0=top, 90=side)
            theta=-45 * DEGREES, # azimuthal angle (spin around Z)
            zoom=1.5
        )

        # Ambient rotation (continuous spin)
        self.begin_ambient_camera_rotation(rate=0.1)

        axes = ThreeDAxes()
        sphere = Sphere(radius=1, color=BLUE)

        self.add(axes, sphere)
        self.wait(4)
        self.stop_ambient_camera_rotation()

        # Animated camera move
        self.move_camera(
            phi=30 * DEGREES,
            theta=45 * DEGREES,
            run_time=2
        )
```

### 3D Surface Example

```python
class SurfaceDemo(ThreeDScene):
    def construct(self):
        self.set_camera_orientation(phi=60 * DEGREES, theta=-45 * DEGREES)

        surface = Surface(
            lambda u, v: np.array([u, v, np.sin(u) * np.cos(v)]),
            u_range=[-3, 3],
            v_range=[-3, 3],
            resolution=(20, 20),
            fill_opacity=0.7,
        )
        surface.set_fill_by_checkerboard(BLUE, TEAL, opacity=0.7)

        self.add(surface)
        self.begin_ambient_camera_rotation(rate=0.2)
        self.wait(5)
```

### Fixed-in-Frame (2D labels in 3D scenes)

```python
# Labels that don't rotate with the camera
label = MathTex(r"f(x,y) = \sin(x)\cos(y)").to_corner(UL)
self.add_fixed_in_frame_mobjects(label)
self.play(Write(label))
```

---

## 3. Scene Lifecycle & Setup

### Using setup() and teardown()

```python
class MyScene(Scene):
    def setup(self):
        """Runs before construct(). Use for shared initialization."""
        self.axes = Axes(x_range=[-3, 3], y_range=[-2, 2])
        self.tracker = ValueTracker(0)

    def construct(self):
        self.play(Create(self.axes))
        # self.axes and self.tracker are available here

    def teardown(self):
        """Runs after construct(). Rarely needed."""
        pass
```

### Config — Changing Defaults

```python
# At the top of your script, before class definitions:
config.background_color = "#1a1a2e"   # dark navy
config.frame_width = 16               # widescreen
config.frame_height = 9
config.pixel_width = 1920
config.pixel_height = 1080

# Or as CLI flags:
# manim -qh --background_color=#1a1a2e scene.py SceneName
```

### Scene.next_section() — for videos with chapters

```python
def construct(self):
    self.next_section("Introduction")
    # ... intro animations ...

    self.next_section("Main Proof", skip_animations=False)
    # ... main content ...
```

---

## 4. 3Blue1Brown Signature Patterns

These are the techniques that give 3b1b videos their distinctive look. Apply them for polished, math-educator quality output.

### Pattern A: Smooth Parameter Reveal

The signature "watching math unfold" look — a function traced out as a parameter sweeps:

```python
class TraceCurve(Scene):
    def construct(self):
        ax = Axes(x_range=[-PI, PI], y_range=[-1.5, 1.5])
        t = ValueTracker(-PI)

        # The dot follows the function as t increases
        dot = always_redraw(lambda: Dot(
            ax.coords_to_point(t.get_value(), np.sin(t.get_value())),
            color=YELLOW, radius=0.08
        ))
        # TracedPath leaves a trail behind the dot
        path = TracedPath(dot.get_center, stroke_color=RED, stroke_width=3)

        self.add(ax, path, dot)
        self.play(t.animate.set_value(PI), run_time=5, rate_func=linear)
```

### Pattern B: TransformMatchingTex — Equation Morphing

Parts of equations that are identical stay still; only changed parts animate:

```python
class EquationMorph(Scene):
    def construct(self):
        eq1 = MathTex(r"e^{i\pi} + 1 = 0")
        eq2 = MathTex(r"e^{i\pi} = -1")
        eq3 = MathTex(r"e^{i\theta} = \cos\theta + i\sin\theta")

        self.play(Write(eq1))
        self.wait()
        self.play(TransformMatchingTex(eq1, eq2))
        self.wait()
        self.play(TransformMatchingTex(eq2, eq3))
        self.wait()
```

### Pattern C: LaggedStart for Visual Richness

Multiple objects appearing with a slight cascade — signature 3b1b style:

```python
class LaggedReveal(Scene):
    def construct(self):
        dots = VGroup(*[Dot(RIGHT * i) for i in range(-5, 6)])

        # All appear with staggered starts
        self.play(LaggedStart(
            *[GrowFromCenter(d) for d in dots],
            lag_ratio=0.1,
            run_time=2
        ))

        # Ripple effect across a grid
        grid = VGroup(*[Square(side_length=0.5).shift(RIGHT*i + UP*j)
                        for i in range(-4, 5)
                        for j in range(-3, 4)])
        self.play(LaggedStart(
            *[Indicate(sq) for sq in grid],
            lag_ratio=0.02
        ))
```

### Pattern D: Highlight then Transform

Draw attention to a part, then evolve it:

```python
class HighlightPattern(Scene):
    def construct(self):
        formula = MathTex(r"F = ma")
        self.play(Write(formula))
        self.wait()

        # Indicate calls attention to the whole formula
        self.play(Indicate(formula))

        # Surround with a box
        box = SurroundingRectangle(formula, color=YELLOW, buff=0.2)
        self.play(Create(box))
        self.wait()

        # Expand to full form
        full = MathTex(r"\vec{F} = m\vec{a}")
        self.play(
            ReplacementTransform(formula, full),
            FadeOut(box)
        )
```

### Pattern E: NumberPlane Transform

Show continuous deformation of space — iconic for linear algebra:

```python
class LinearTransform(Scene):
    def construct(self):
        plane = NumberPlane()
        self.add(plane)
        self.wait()

        # Apply a 2x2 matrix transformation
        matrix = [[1, 1], [0, 1]]   # shear
        self.play(plane.animate.apply_matrix(matrix), run_time=3)
        self.wait()

        # Non-linear function (like 3b1b's complex plane videos)
        plane.prepare_for_nonlinear_transform()
        self.play(
            plane.animate.apply_function(
                lambda p: np.array([
                    p[0] + 0.3 * np.sin(p[1]),
                    p[1] + 0.3 * np.sin(p[0]),
                    0
                ])
            ),
            run_time=3
        )
```

### Pattern F: Flash + Emphasis Sequence

Standard teaching moment emphasis:

```python
# Burst of light at a point
self.play(Flash(dot.get_center(), color=YELLOW, flash_radius=0.5))

# Draw attention to a specific part of an equation
self.play(Circumscribe(formula[2], color=RED))

# Wiggle to indicate "this is the important part"
self.play(Wiggle(key_term, scale_value=1.3, rotation_angle=0.05*TAU))
```

---

## 5. Performance Tips

- Use `-ql` (low quality) during development. Only use `-qh` for finals.
- Avoid `--disable_caching` unless you have TracedPath bugs (it significantly slows re-renders).
- Chunk long videos into multiple scene classes and render separately; concatenate in a video editor.
- `self.add()` is instant and free; `self.play()` is rendered. Use `add()` for static setup.
- For heavy `always_redraw` scenes, profile by checking if the lambda does unnecessary work each frame.
- Grant Sanderson's workflow: use interactive mode (`manimgl -se LINE`) to drop into the scene at a specific line and iterate quickly. ManimCE equivalent: use Jupyter `%%manim`.

---

## 6. Coordinate System Deep Dive

```
Screen coordinates:
  ┌─────────────────────────┐
  │ (-7, 4)        ( 7, 4) │  ← TOP
  │                         │
  │         (0,0)           │  ← CENTER (ORIGIN)
  │                         │
  │ (-7,-4)        ( 7,-4) │  ← BOTTOM
  └─────────────────────────┘
  ← LEFT (-7)    RIGHT (7) →

Manim constants:
  UP    = [0,  1, 0]
  DOWN  = [0, -1, 0]
  LEFT  = [-1, 0, 0]
  RIGHT = [ 1, 0, 0]
  ORIGIN= [ 0, 0, 0]

  # Useful numeric constants
  PI = 3.14159...
  TAU = 2 * PI
  DEGREES = PI / 180    # multiply by this to convert
  # e.g.: 45 * DEGREES = PI/4

Screen edges:
  config.frame_width  = 14.222...  (default)
  config.frame_height = 8.0        (default)

  # Edges as floats
  from manim import config
  left_edge = -config.frame_width / 2   # ≈ -7.1
  top_edge  =  config.frame_height / 2  # = 4.0
```
