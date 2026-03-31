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
- Interactivity is added by attaching code to frames and object instances.

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
- **Common Lisp scripting** — real Lisp, attached to frames and objects, with a clean macro surface for beginners
- **Single-file export** — publish to a bundle (HTML + WebAssembly), shareable as one file, runnable in any browser
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

### Scripting: Common Lisp with a Macro Surface

Flashpoint's scripting is **Common Lisp** — no separate scripting language.
There is no "Flare" or any invented language. The scripting surface is a
macro package that provides a clean, minimal API for beginners while being
full CL for anyone who wants more.

**Why not a custom scripting language?**

Flash needed ActionScript because the `.swf` distribution model required a
baked-in runtime — there was no other way to add interactivity. Flashpoint
publishes to HTML + WebAssembly running in a browser. The runtime is already
Common Lisp compiled to WASM. Adding a separate language would mean designing,
implementing, parsing, compiling, and maintaining it for no gain. The language
is already there.

**Why Common Lisp specifically?**

Scheme (and Lisp generally) has the most approachable syntax of any language —
not because it looks like English, but because it has exactly one rule:
`(operator arguments)`. That's it. Python, despite its reputation, has dozens
of special cases, sigils, and implicit behaviors. With one rule, a beginner
thinks about *computation*, not syntax. SICP taught programming to MIT
freshmen with Scheme for decades on exactly this principle.

**The macro surface for beginners:**

A macro package provides clean, event-driven syntax that a non-programmer can
read immediately:

```lisp
;; React to a mouse click
(on-click button
  (goto-and-play "level2"))

;; Move something every frame
(on-enter-frame
  (incf (x player) 5))

;; Respond to keyboard
(when-key :space
  (setf (visible bullet) t))

;; Simple condition
(when (> score 100)
  (goto-and-play "win"))

;; Variables
(defvar score 0)
(defvar player-name "Player 1")
```

That's real Common Lisp — no toy language, no wall between "beginner mode"
and "advanced mode." When the beginner is ready for more, they're already
writing CL. The graduation path is learning more of the same language.

**SBCL condition system as error handling:**

SBCL's condition system is wrapped to produce friendly, plain-language error
messages. A missing variable name becomes "I don't know what 'bal' is — did
you mean 'ball'?" not a raw stack trace. This is a presentation concern, not
a language design concern.

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
│  │  tools      │  │  (Common Lisp)           │  │
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
- **WebAssembly** — SBCL-compiled runtime for live script evaluation
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
- `scripts/` — Common Lisp source files per frame/symbol (plain text, `.lisp`)

Human-readable project format — can be version-controlled, diffed, inspected.
Scripts are plain `.lisp` files — editable in any text editor, not locked into
the Flashpoint editor.

### Published Bundle

When you click **Publish**:
1. Scripts are bundled with the SBCL-compiled runtime WASM
2. Assets are bundled inline (Base64) or alongside as a ZIP
3. Output is a single `index.html` + `runtime.wasm`
4. Runs in any browser, no dependencies

Target: a published file should be under 500KB for a simple animation, under
5MB for a typical game. No external CDN dependencies.

### The Runtime

The runtime (compiled SBCL → WebAssembly) handles:
- Frame event dispatch (`on-enter-frame`, `on-click`, `when-key`, etc.)
- Stage access (positions, visibility, playback control)
- Basic drawing API (for dynamic content)
- Sound trigger and control
- Symbol instantiation and destruction

The macro package is loaded at runtime startup. Scripts attached to frames
and instances are evaluated against the running CL image.

---

## Scripting API Reference (Macro Surface)

The beginner-facing API. Every macro expands to internal engine calls.

### Events

```lisp
(on-enter-frame &body body)           ; runs every frame
(on-click instance &body body)        ; mouse click on instance
(on-release instance &body body)      ; mouse button release
(on-mouse-over instance &body body)   ; mouse enters instance bounds
(on-mouse-out instance &body body)    ; mouse leaves instance bounds
(when-key key &body body)             ; keyboard key pressed
```

### Timeline Control

```lisp
(play)                    ; play from current frame
(stop)                    ; stop playback
(goto-and-play target)    ; target is frame number or label string
(goto-and-stop target)
(next-frame)
(prev-frame)
(current-frame)           ; returns current frame number
```

### Instance Properties

```lisp
(x instance)              ; get/setf x position
(y instance)              ; get/setf y position
(width instance)          ; get/setf width
(height instance)         ; get/setf height
(rotation instance)       ; get/setf rotation in degrees
(scale-x instance)        ; get/setf x scale factor
(scale-y instance)        ; get/setf y scale factor
(alpha instance)          ; get/setf opacity (0.0–1.0)
(visible instance)        ; get/setf boolean visibility
```

All properties are setf-able:
```lisp
(setf (x ball) 100)
(incf (x ball) 5)
(setf (visible enemy) nil)
```

### Stage

```lisp
(stage-width)             ; stage width in pixels
(stage-height)            ; stage height in pixels
(mouse-x)                 ; current mouse X position
(mouse-y)                 ; current mouse Y position
```

### Symbols

```lisp
(spawn symbol-name x y)   ; create a new instance of symbol
(destroy instance)        ; remove instance from stage
(instances-of symbol-name) ; list of all instances of symbol
```

### Sound

```lisp
(play-sound name)         ; play a sound asset once
(loop-sound name)         ; loop a sound asset
(stop-sound name)         ; stop a playing sound
```

---

## What Makes This Different From Existing Tools

| Tool | Where it falls short |
|------|---------------------|
| **Godot** | Requires understanding the scene/node architecture before anything works |
| **Unity** | Large install, complex project setup, primarily 3D-oriented UX |
| **GDevelop** | Event-based rather than timeline-based; no real scripting language |
| **Scratch** | Block-based is appropriate for young children but becomes limiting; no timeline |
| **Animate CC** | Adobe's Flash successor; $55/month subscription, professional-oriented |
| **Bitsy** | Deliberately constrained; too limited for most creative visions |
| **Pico-8** | Fantasy console constraints require programming first; no visual timeline |

Flashpoint's niche: timeline + stage model, vector drawing built in, Common
Lisp scripting with a beginner-friendly macro surface, browser-native output,
free and open-source.

---

## Implementation Plan

### Phase 0: Core Editor Shell
- Browser-based canvas editor
- Stage with basic shape tools (rectangle, ellipse, pencil, fill)
- Timeline with layers and keyframes
- Basic playback (scrub, play, stop)
- Project save/load (JSON + scripts, IndexedDB)

### Phase 1: Animation
- Tweening (motion, scale, rotation, alpha)
- Symbol system (create, place, edit symbols)
- Onion skinning
- Frame labels

### Phase 2: Scripting
- SBCL compiled to WebAssembly for the runtime
- Macro package: on-enter-frame, on-click, when-key, etc.
- Frame scripts and instance scripts
- Friendly error messages wrapping SBCL's condition system

### Phase 3: Assets
- Image import (PNG, JPEG, SVG)
- Audio import (MP3, OGG)
- Asset library panel

### Phase 4: Publish
- Single-file bundle export (index.html + runtime.wasm)
- Preview mode (test published output in the editor)

### Phase 5: Polish
- Better drawing tools (bezier curves, gradient fills, text)
- Scene management (multiple scenes in one project)
- Undo/redo
- Keyboard shortcuts

---

## Design Constraints (Non-Negotiable)

1. **Zero install to create** — the editor runs in a browser tab, no download
2. **Zero install to view** — published files run in any browser, no plugins
3. **Free and open-source** — the tool is a cultural project, not a product
4. **Single-file publish** — one file to share, one file to host
5. **Works on tablets** — the audience includes teenagers without laptops
6. **Under 5 minutes to first creation** — the approachability bar from the problem statement
7. **No invented scripting language** — Common Lisp macros suffice; don't build what you already have

---

## Open Questions

1. **SBCL → WASM toolchain:** SBCL can compile to WASM via the WASM backend or via intermediate C. Current maturity of the SBCL WASM target needs evaluation. Alternative: use WASM-optimized CL like `WebAssembly Scheme` or `BiwaScheme` if SBCL WASM is too large.

2. **Editor language:** The editor UI itself (Canvas rendering, timeline, drawing tools) should probably be in ClojureScript targeting the browser, keeping the CL family consistent. Alternatively TypeScript for broader contributor familiarity.

3. **Script evaluation model:** When a frame script runs in the editor (live preview), it needs to evaluate in a sandboxed SBCL image that has access to the stage state. The communication between the JS editor and the WASM runtime needs careful design.

4. **Drawing on tablets:** Designing the editor for touchscreen input — drawing, timeline scrubbing, symbol placement — requires specific UX work. Stylus support is particularly important for the drawing tools.

5. **Sound in the editor:** Live sound during scrubbing requires Web Audio API integration. Needs specific design to avoid audio chaos during fast scrubbing.
