# manim-animation

A Claude skill for creating [Manim](https://www.manim.community/) animation scenes in Python — the same library used by [3Blue1Brown](https://www.3blue1brown.com/).

Ask Claude to animate math concepts, visualize algorithms, or build educational videos and it will write complete, runnable Manim scenes with proper pacing, structure, and 3b1b-quality aesthetics.

---

## What it does

- Writes complete, runnable `.py` Manim scene files
- Applies 3Blue1Brown's pacing and scene design philosophy (when to wait, how fast to animate, how to structure a narrative)
- Knows the full Manim API — geometry, LaTeX equations, graphing, cameras, 3D scenes, ValueTracker animations
- Follows production patterns from real multi-scene projects: shared style modules, scene decomposition, column layout, debug workflow
- Guides you through visually reviewing renders with direct `ffmpeg` frame extraction
- Covers debugging — reading Manim errors, common fixes, the static layout workflow

## Example prompts that trigger it

- *"Animate the Fourier transform, showing how sine waves add up"*
- *"Make a 3Blue1Brown-style animation explaining gradient descent"*
- *"Write a Manim scene that visualizes matrix multiplication"*
- *"Animate Euler's identity with a unit circle"*
- *"Create a Manim scene showing how attention weights work in a transformer"*

---

## Installation

### Claude Code

```bash
# Clone and install
git clone https://github.com/yourusername/manim-animation-skill
cp -r manim-animation-skill/manim-animation ~/.claude/skills/

# Or scoped to a specific project
cp -r manim-animation-skill/manim-animation your-project/.claude/skills/
```

Or with npx (once published):
```bash
npx skills add yourusername/manim-animation-skill
```

### Claude.ai

1. Download `manim-animation.skill` from [Releases](../../releases)
2. Go to **Settings → Capabilities → Skills → Upload skill**
3. Select the `.skill` file

---

## Requirements

You need these installed to actually render the animations Claude writes:

```bash
pip install manim          # ManimCE — community edition
brew install ffmpeg        # macOS
# Ubuntu: sudo apt install ffmpeg
# Windows: https://ffmpeg.org/download.html
```

LaTeX is optional but required for equations:
```bash
pip install manim[latex]   # installs a minimal LaTeX distribution
# or install TeX Live / MiKTeX directly for full support
```

---

## What's inside

```
manim-animation/
├── SKILL.md                        # Core instructions Claude loads
└── references/
    ├── pacing.md                   # Timing, waits, scene structure, 3b1b philosophy
    ├── animations.md               # Animation classes, rate functions, updaters, ValueTracker
    ├── mobjects.md                 # All Mobject types — geometry, text, LaTeX, graphing
    ├── advanced.md                 # 3D scenes, cameras, 3b1b signature patterns
    ├── production-patterns.md      # Multi-scene projects, shared style, debug workflow
    └── debugging.md                # Errors, frame extraction, visual checklist
```

The skill uses [progressive disclosure](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — Claude always loads `SKILL.md`, then pulls in reference files only when relevant to the request. A simple animation request loads ~100 tokens of metadata; a complex multi-scene project loads the production patterns reference.

---

## The two versions of Manim

There are two incompatible versions. This skill targets **ManimCE** (the community edition):

| | ManimCE | ManimGL |
|---|---|---|
| Install | `pip install manim` | `pip install manimgl` |
| Import | `from manim import *` | `from manimlib import *` |
| Used by | Most people, all tutorials | Grant Sanderson personally |
| Status | Actively maintained, well documented | Personal tool, less stable |

If you specifically want ManimGL, say so explicitly. Otherwise Claude will write ManimCE code.

---

## Rendering

```bash
manim -ql scene.py MyScene    # Low quality — use during development
manim -qm scene.py MyScene    # Medium quality
manim -qh scene.py MyScene    # High quality — final render only
manim -ql -s scene.py MyScene # Still image — fastest layout check (~1s)
manim -ql -p scene.py MyScene # Render + open preview automatically
```

---

## License

MIT

---

## Related

- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skill repository
- [manim.community](https://www.manim.community/) — ManimCE documentation
- [3b1b/manim](https://github.com/3b1b/manim) — Grant Sanderson's original ManimGL
- [agentskills.io](https://agentskills.io) — Agent Skills open standard
