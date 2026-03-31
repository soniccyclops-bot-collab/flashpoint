# Flashpoint — Design Document

## Problem Statement

The Flash era (roughly 1997–2010) produced an extraordinary density of creative
work — games, animations, interactive art, experimental experiences — made by
individuals and tiny teams with no budget and no professional background.
Nothing since has matched it.

The cause is not distribution (the web still works), not audience (the audience
is still there), and not monetization pressure (Flash was almost entirely
non-commercial). The cause is the tool itself. Flash dropped the skill floor
to a level that has never been replicated. It was all-in-one, approachable,
and immediately productive. A teenager with a weekend could make something real.

Steve Jobs' 2010 decision to block Flash on iOS killed the ecosystem —
not natural decline, not Adobe's mismanagement (though both followed). The
developer community chased mobile and abandoned the browser as a creative
platform. The vacuum has never been filled.

**Flashpoint is the attempt to fill it.**

---

## Core Design Principle: Simplicity Above All Else

Flash's genius was a single unified mental model: the **timeline + stage**.

- The **stage** is what you see. You draw on it, place objects on it.
- The **timeline** is when things happen. Frames move left to right; time moves forward.
- Everything is one of two things: a **shape** (drawn directly) or a **symbol** (a reusable object with its own timeline).
- Code (ActionScript) was attached to frames and objects in the same interface.

This model is **understandable without documentation**. A child who has ever
drawn a flipbook understands the timeline. A child who has ever played with
stickers understands placing objects on a stage. The leap from "I understand
this" to "I made something" was minutes, not weeks.

Modern tools have abandoned this simplicity in pursuit of power. Godot is an
excellent engine — but you must understand scenes, nodes, signals, and the
engine's architecture before a single thing moves. Unity is more powerful still,
and further from approachable. None of them have Flash's ramp.

**Flashpoint's first design constraint: the time from "I opened the tool" to
"something I made is moving on screen" must be under five minutes for someone
with no prior experience.**

---

## What Flashpoint Is

A browser-based multimedia creation tool with:

- **Timeline + Stage editor** — the Flash mental model, modern execution
- **Vector drawing tools** — create shapes directly in the editor, no external asset required
- **Frame-by-frame and tween animation** — same as Flash
- **Scripting** — a simple, approachable scripting language attached to frames and objects
- **Single-file export** — publish to a `.fp` bundle (HTML + WebAssembly), shareable as one file, runnable in any browser
- **Embeddable** — the published file runs without plugins, without installs, on any modern device

What Flashpoint is **not**:

- Not a game engine (though games can be made with it)
- Not a video editor
- Not a 3D tool
- Not trying to be the most powerful tool — trying to be the most approachable

---

## The Mental Model in Detail

### Stage

The stage is a rectangular canvas. Everything visible lives here. You can:
- Draw shapes directly (pencil, shapes, fill, stroke)
- Place symbols (reusable components with their own timelines)
- Import images, audio, and video as assets
- Set the stage size and background color

The stage is what the final published file shows.

### Timeline

The timeline runs across the bottom of the editor. Time moves left to right.
Each horizontal track is a **layer**. Each cell in a layer is a **frame**.

- **Keyframes** — a frame where something is explicitly defined (position, shape, visibility)
- **Empty frames** — continuation of the previous keyframe
- **Tweens** — automatically interpolate position, scale, rotation, and opacity between two keyframes

The playhead shows the current frame. Pressing play scrubs through the timeline
and shows the animation on stage.

### Symbols

A symbol is a reusable object with its own timeline. Symbols are:
- **Movie clips** — have their own playback, can be controlled by script
- **Graphics** — static or synced to the parent timeline
- **Buttons** — interactive objects with up/over/down/hit states

Placing a symbol on stage creates an **instance**. The same symbol can have
many instances, each with independent properties (position, scale, name).

### Scripting Language (Flare)

Flashpoint's scripting language is called **Flare**. It is designed to be:

- Approachable to non-programmers
- Immediately useful with minimal boilerplate
- Attached to frames and object instances (same as ActionScript 2)
- Safe — no file system access, no network access beyond approved APIs

**Syntax goals:** Python-like readability, no type declarations required, no
class definitions required for simple scripts.

```flare
-- When this frame plays, move the ball
ball.x = ball.x + 5
ball.y = ball.y - 2

-- React to mouse click
on click(ball):
  gotoAndPlay("explode")

-- Simple loop
for i in range(10):
  spawn(spark, stage.mouseX, stage.mouseY)
```

Flare compiles to WebAssembly for the published output. The editor runs an
interpreter for live preview.

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                  Flashpoint Editor               │
│                                                  │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │  Stage      │  │  Timeline                │  │
│  │  (Canvas)   │  │  (layers, keyframes,     │  │
│  │             │  │   tweens, playhead)      │  │
│  └─────────────┘  └──────────────────────────┘  │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │  Drawing    │  │  Script editor           │  │
│  │  tools      │  │  (Flare)                 │  │
│  └─────────────┘  └──────────────────────────┘  │
│  ┌──────────────────────────────────────────┐    │
│  │  Asset library (symbols, imported media) │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  [ File ] [ Edit ] [ Control ] [ Publish ]       │
└──────────────────────────────────────────────────┘
          │                         │
          ▼                         ▼
   .fp project file           published bundle
   (JSON + assets,            (index.html + runtime.wasm
    save/load/share)           + project.fp)
                               runs in any browser
```

### Editor

The editor runs in the browser. No install. Built with:
- **Canvas 2D API** — stage rendering, drawing tools
- **WebAssembly** — Flare interpreter for live preview
- **IndexedDB** — local project storage
- **File System Access API** — open/save to local disk (where supported)

The editor is itself a web page. No Electron, no native wrapper. This means:
- Zero install friction for creators
- Works on any device with a modern browser (including tablets)
- The published file format can be previewed in the same environment as the editor

### Project File Format (`.fp`)

A `.fp` file is a ZIP archive containing:
- `project.json` — timeline data, symbol definitions, scene graph
- `assets/` — imported images, audio, video
- `scripts/` — Flare source files per frame/symbol

Human-readable project format — can be version-controlled, diffed, inspected.

### Published Bundle

When you click **Publish**:
1. Flare scripts compile to WebAssembly
2. Assets are bundled inline (Base64) or alongside as a ZIP
3. Output is a single `index.html` + `runtime.wasm`
4. Runs in any browser, no dependencies

Target: a published file should be under 500KB for a simple animation, under
5MB for a typical game. No external CDN dependencies.

### Flare Runtime

The Flare scripting runtime handles:
- Frame event dispatch (`onEnterFrame`, `onMouseClick`, etc.)
- Stage access (positions, visibility, playback control)
- Basic drawing API (for dynamic content)
- Sound trigger and control
- Symbol instantiation and destruction

The runtime is a small WebAssembly module (~100KB target) that is embedded
in every published file.

---

## Scripting Language Design: Flare

Flare is designed around five core principles:

1. **Readable by a non-programmer** — clear English-like syntax, no symbols where words work
2. **Attached to objects** — scripts live on frames and instances, not in separate files
3. **No setup required** — you can write a one-line script and it works
4. **Errors are helpful** — error messages explain the problem in plain language
5. **Gradual complexity** — simple things are simple, complex things are possible

### Core constructs

```flare
-- Variables (no declaration needed)
score = 0
playerName = "Player 1"

-- Conditionals
if score > 100:
  gotoAndPlay("win")

-- Loops
repeat 5:
  ball.x = ball.x + 10

for item in stage.children:
  item.alpha = 0.5

-- Functions
function moveLeft(thing, amount):
  thing.x = thing.x - amount

moveLeft(player, 10)

-- Events (attached to instances or frames)
on click(button):
  play()

on enterFrame:
  player.rotation = player.rotation + 1

-- Built-in timeline control
play()
stop()
gotoAndPlay(frameNumber)
gotoAndStop("labelName")
gotoAndPlay("scene2")

-- Stage access
stage.width
stage.height
stage.mouseX
stage.mouseY

-- Instance access (by name given in editor)
myBall.x
myBall.y
myBall.visible = false
myBall.alpha = 0.5
myBall.scaleX = 2

-- Sound
sound("explosion").play()
music("background").loop()
```

### Comparison with ActionScript 2

Flare is deliberately simpler than ActionScript 2. No class definitions, no
type annotations, no inheritance for typical use cases. The target audience
is someone who has never programmed before. ActionScript 2's complexity was
a learning curve; Flare's design removes it.

Advanced users who need more can still write complex Flare — the language
supports functions, closures, lists, and maps. But none of that is required
to make a working game or animation.

---

## What Makes This Different From Existing Tools

| Tool | Where it falls short |
|------|---------------------|
| **Godot** | Requires understanding the scene/node architecture before anything works |
| **Unity** | Large install, complex project setup, primarily 3D-oriented UX |
| **GDevelop** | Closest competitor; event-based rather than timeline-based, browser output is good but the tool model is different |
| **Scratch** | Block-based is great for children but condescending to teenagers; no timeline |
| **Animate CC** | Adobe's Flash successor; $55/month subscription, professional-oriented, abandoned approachability |
| **Bitsy** | Extremely constrained (intentionally); too limited for most creative visions |
| **Pico-8** | Fantasy console constraints are inspiring but require programming first; no visual timeline |

Flashpoint's niche: timeline + stage mental model, vector drawing built in,
scripting that a non-programmer can read and write, browser-native output,
free and open-source.

---

## Implementation Plan

### Phase 0: Core Editor Shell
- Browser-based canvas editor
- Stage with basic shape tools (rectangle, ellipse, pencil, fill)
- Timeline with layers and keyframes
- Basic playback (scrub, play, stop)
- Project save/load (JSON, IndexedDB)

### Phase 1: Animation
- Tweening (motion, scale, rotation, alpha)
- Symbol system (create, place, edit symbols)
- Onion skinning
- Frame labels

### Phase 2: Scripting
- Flare language parser and interpreter
- Frame scripts and instance scripts
- Event system (click, enterFrame, keyDown, etc.)
- Stage API (access instances by name, control playback)

### Phase 3: Assets
- Image import (PNG, JPEG, SVG)
- Audio import (MP3, OGG)
- Asset library panel

### Phase 4: Publish
- Flare → WebAssembly compiler
- Single-file bundle export
- Preview mode (test published output in the editor)

### Phase 5: Polish
- Better drawing tools (bezier curves, gradient fills, text)
- Scene management (multiple scenes in one project)
- Undo/redo
- Keyboard shortcuts

---

## Language and Runtime Implementation

The Flare interpreter (phases 2-3) will be implemented in Common Lisp (SBCL)
and compiled to WebAssembly. This gives us:
- A real language implementation with proper semantics
- SBCL's native performance characteristics
- The ability to run the interpreter both in the editor (for live preview) and
  in the published bundle (for the runtime)

The Flare → WebAssembly compiler (phase 4) is a separate path for published
output: compile Flare directly to WASM bytecode for maximum performance in
the final published file.

---

## Design Constraints (Non-Negotiable)

1. **Zero install to create** — the editor runs in a browser tab, no download
2. **Zero install to view** — published files run in any browser, no plugins
3. **Free and open-source** — the tool is a cultural project, not a product
4. **Single-file publish** — one file to share, one file to host
5. **Works on tablets** — the audience includes teenagers without laptops
6. **Under 5 minutes to first creation** — the approachability bar from the problem statement

---

## Open Questions

1. **Flare syntax:** Python-style indentation vs. explicit `end` keywords? Indentation is more modern but can trip up beginners. Explicit keywords are more forgiving for whitespace errors.

2. **Drawing tools first or timeline first?** Starting with drawing gives immediate creative satisfaction. Starting with timeline gives immediate understanding of the core model. Probably drawing first for the initial experience.

3. **Collaboration:** Real-time multiplayer editing (like Figma) would be powerful but adds significant complexity. Phase 2+ consideration.

4. **Sound in the editor:** Live sound during scrubbing is the Flash experience but requires careful Web Audio API work.

5. **Mobile creation:** Designing the editor for touchscreen input is non-trivial. Tablet support is a constraint but the tools UX for touch input (drawing, timeline scrubbing) needs specific design work.
