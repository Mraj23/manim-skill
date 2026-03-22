# Mobjects Reference

## Table of Contents
1. [Geometry Mobjects](#1-geometry-mobjects)
2. [Text and LaTeX](#2-text-and-latex)
3. [Graphing and Plotting](#3-graphing-and-plotting)
4. [3D Mobjects](#4-3d-mobjects)
5. [Grouping and Layout](#5-grouping-and-layout)
6. [Images and SVGs](#6-images-and-svgs)
7. [Number Displays](#7-number-displays)

---

## 1. Geometry Mobjects

```python
# Basic shapes
Circle(radius=1, color=BLUE, fill_opacity=0.5)
Square(side_length=2, color=RED)
Rectangle(width=3, height=2, color=GREEN)
Triangle(color=YELLOW)
Polygon(p1, p2, p3, p4, color=BLUE)    # arbitrary polygon
RegularPolygon(n=6, color=ORANGE)       # hexagon etc.
Star(n=5, outer_radius=1, inner_radius=0.4)

# Lines and arrows
Line(start=LEFT, end=RIGHT, color=WHITE)
Arrow(start=ORIGIN, end=UP, color=YELLOW)
DoubleArrow(start=LEFT, end=RIGHT)
DashedLine(start=LEFT, end=RIGHT)
Vector(direction=RIGHT)                 # Arrow starting at ORIGIN

# Angles and arcs
Arc(radius=1, start_angle=0, angle=PI/2)
ArcBetweenPoints(start, end, angle=PI/2)
Angle(line1, line2, radius=0.5)
RightAngle(line1, line2)

# Dots
Dot(point=ORIGIN, radius=0.08, color=WHITE)
LabeledDot(label="A", point=ORIGIN)
AnnotationDot()

# Braces
Brace(mobject, direction=DOWN)          # bracket with optional text
BraceBetweenPoints(p1, p2, direction=UP)

# Other
Sector(outer_radius=2, inner_radius=1, angle=TAU/4)
Annulus(inner_radius=0.5, outer_radius=1)
Ellipse(width=4, height=2)
```

### Styling Shapes

```python
obj.set_fill(color=BLUE, opacity=0.5)
obj.set_stroke(color=WHITE, width=2)
obj.set_color(RED)                      # sets both stroke and fill
obj.set_opacity(0.8)
obj.round_corners(radius=0.1)           # round corners on polygons
```

---

## 2. Text and LaTeX

```python
# Plain text (no LaTeX)
Text("Hello", font_size=48, color=WHITE)
Text("Bold", weight=BOLD)
Text("Italic", slant=ITALIC)
Text("Custom Font", font="Arial")
MarkupText("<b>bold</b> and <i>italic</i>")  # Pango markup

# LaTeX text mode (for prose with math mixed in)
Tex(r"The integral \(\int_0^1 x\, dx = \frac{1}{2}\)")

# LaTeX math mode (for pure equations)
MathTex(r"\frac{d}{dx} \sin(x) = \cos(x)")

# Multi-part MathTex (for targeting sub-parts)
eq = MathTex(r"e^x", r"=", r"\sum_{n=0}^\infty", r"\frac{x^n}{n!}")
eq[0].set_color(YELLOW)   # highlight "e^x"
eq[2].set_color(BLUE)     # highlight the sum

# Isolate substrings for coloring
eq = MathTex(r"f(x) = x^2 + 2x + 1",
             substrings_to_isolate=["x"])
eq.set_color_by_tex("x", RED)
```

### LaTeX Tips

- Always use raw strings: `r"\frac{1}{2}"` not `"\frac{1}{2}"`
- Use `\\` for line breaks inside equations
- LaTeX must be installed — if rendering fails, check `manim --version` dependencies
- `Tex` uses text mode; `MathTex` wraps content in `$ $` automatically
- For alignment: `MathTex(r"\begin{aligned} ... \end{aligned}")`

---

## 3. Graphing and Plotting

```python
# Axes
ax = Axes(
    x_range=[-3, 3, 1],       # [min, max, step]
    y_range=[-2, 5, 1],
    x_length=7,                # screen width in Manim units
    y_length=5,
    axis_config={
        "color": BLUE,
        "include_ticks": True,
        "include_numbers": True,
    },
    x_axis_config={"numbers_to_include": [-2, -1, 0, 1, 2]},
    tips=True,                 # arrowheads at ends
)
labels = ax.get_axis_labels(x_label="x", y_label="f(x)")

# NumberLine (1D)
nl = NumberLine(x_range=[-5, 5, 1], include_numbers=True)

# NumberPlane (2D grid with axes)
plane = NumberPlane(
    x_range=[-5, 5, 1],
    y_range=[-3, 3, 1],
    background_line_style={"stroke_color": TEAL, "stroke_opacity": 0.3}
)

# Plotting functions
curve = ax.plot(lambda x: x**2, color=RED, x_range=[-2, 2])
curve = ax.plot(np.sin, color=YELLOW)

# Parametric curve
curve = ax.plot_parametric_curve(
    lambda t: np.array([np.cos(t), np.sin(t)]),
    t_range=[0, TAU],
    color=GREEN
)

# Points and dots on graph
dot = Dot(ax.coords_to_point(1, 1))
dot = ax.get_graph_label(curve, label=MathTex("f(x)"), x_val=2)

# Areas under curve
area = ax.get_area(curve, x_range=[0, 2], color=YELLOW, opacity=0.3)
riemann = ax.get_riemann_rectangles(curve, x_range=[0, 2], dx=0.2)

# Helper methods
ax.coords_to_point(x, y)       # math coords → screen point
ax.point_to_coords(point)       # screen point → math coords
ax.i2gp(x_val, graph)          # input to graph point
```

---

## 4. 3D Mobjects

```python
# 3D surfaces
Surface(
    lambda u, v: np.array([u, v, np.sin(u) * np.cos(v)]),
    u_range=[-PI, PI],
    v_range=[-PI, PI],
    resolution=(30, 30),
)
Sphere(radius=1)
Cylinder(radius=1, height=2)
Cone(base_radius=1, height=2)
Torus(major_radius=2, minor_radius=0.5)
Prism(dimensions=[1, 2, 3])

# 3D geometry
Line3D(start=ORIGIN, end=UP)
Arrow3D(start=ORIGIN, end=UP)
```

---

## 5. Grouping and Layout

```python
# VGroup — group of VMobjects, can apply transforms to all
group = VGroup(circle, square, triangle)
group.arrange(RIGHT, buff=0.5)          # line them up with spacing
group.arrange(DOWN, aligned_edge=LEFT)
group.arrange_in_grid(rows=2, cols=3)
group.scale(0.5)
group.move_to(ORIGIN)

# Group (non-vector — for ImageMobjects etc.)
group = Group(image, text)

# Useful VGroup patterns
all_shapes = VGroup(*self.mobjects)     # grab everything in scene
self.play(FadeOut(all_shapes))
```

---

## 6. Images and SVGs

```python
# Images
img = ImageMobject("path/to/image.png")
img.scale(2)
img.move_to(ORIGIN)

# SVG
svg = SVGMobject("path/to/icon.svg")
svg.set_color(BLUE)
```

---

## 7. Number Displays

```python
# DecimalNumber — shows a float, can animate the value
num = DecimalNumber(0, num_decimal_places=2, color=WHITE)
num.add_updater(lambda m: m.set_value(tracker.get_value()))

# Integer — shows an int
n = Integer(42)

# Variable — shows "label = value" pair
var = Variable(0, label=MathTex("x"), num_decimal_places=2)
self.play(var.tracker.animate.set_value(5))  # animates the value
```
