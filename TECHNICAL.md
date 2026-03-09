# AgentVisual — Technical Spec

## Integration Model

AgentVisual runs as an **MCP server** — consumed by Claude Code, Cursor, or any MCP-compatible coding agent. The agent calls AgentVisual tools during its workflow, receives visual evaluation results, and acts on them.

### MCP Tools

```
screenshot(window, options?)
  - Capture a screenshot of the target window
  - Options: resolution, format, compress
  - Returns: image file path + vision model evaluation

record(window, duration, options?)
  - Record a short video clip of the target window
  - Options: fps, resolution, format
  - Returns: video file path + vision model evaluation

evaluate(files[], prompt?)
  - Send previously captured screenshots/clips to the vision model
  - Optional custom evaluation prompt (e.g., "focus on spacing consistency")
  - Returns: structured observations
```

**TBD:** Should there be higher-level composite tools? E.g., `qa_pass(window)` that automatically runs screenshots at multiple breakpoints + interaction clips and returns a full report? Or keep it primitive and let the agent compose?

---

## System Architecture

```
AgentVisual MCP Server (TypeScript)
  │
  ├── MCP Layer (@modelcontextprotocol SDK, stdio transport)
  │     - Tool registration and request handling
  │     - Pattern follows overtooled-mcp structure
  │
  ├── Capture Engine (Rust CLI binary)
  │     - Invoked by TypeScript layer via child process
  │     - Window-targeted screenshot and video recording
  │     - Returns file paths to captured media
  │
  ├── Image Processing (sharp)
  │     - Resize/compress screenshots before vision API
  │     - Optimize for minimum token cost
  │
  └── Vision Evaluation (Gemini API)
        - Send captures with structured evaluation prompts
        - Parse and return structured observations
```

### Why This Split

- **Rust for capture** — screen recording needs native OS APIs and GPU encoding. Can't do this from Node.js.
- **TypeScript for orchestration** — MCP SDK is TypeScript, vision API calls are HTTP, prompt construction is text. No reason to do this in Rust.
- **Gemini for vision** — only major model with native video input (as of March 2026). Claude handles images well but can't process video.

---

## Capture Engine (Rust)

### Existing Infrastructure: QuickShotter

The capture engine is derived from QuickShotter (Tauri 2 + Rust), an already-built screen capture and recording tool. Relevant subsystems:

**Screenshot capture** (`capture.rs`):
- Cross-platform via `xcap` crate (GDI/DXGI on Windows, AVFoundation on macOS, X11/Wayland on Linux)
- Multi-monitor DPI-aware stitching
- Window-targeted capture

**Video recording** (`recording/`):
- GPU-accelerated H.264 encoding:
  - Windows: Media Foundation (NVENC/AMF/QuickSync auto-selected)
  - macOS: VideoToolbox (via Objective-C FFI bridge)
  - Fallback: openh264 software encoder
- Two-thread pipeline: capture thread + encoder thread with 4-frame bounded buffer
- Configurable FPS: 10/15/24/30
- Region recording
- MP4 + GIF output
- Backpressure: encoder lag causes frame drops, not stalls

**Window detection** (`window_capture.rs`):
- Persistent worker thread polls cursor position
- Returns window boundaries for targeted capture

### What Needs to Change

QuickShotter is a Tauri GUI app. AgentVisual needs a **headless CLI binary** that the TypeScript MCP server invokes.

**Extract and adapt:**
- Strip Tauri dependencies (window management, tray, overlays, hotkeys, annotation)
- Keep: `capture.rs`, `recording/pipeline.rs`, `recording/encoder*.rs`, `window_capture.rs`
- Add CLI interface:
  ```
  agentvisual-capture screenshot --window "My App" --output /tmp/shot.png
  agentvisual-capture record --window "My App" --duration 2s --fps 15 --output /tmp/clip.mp4
  agentvisual-capture list-windows  (returns JSON list of window titles + handles)
  ```
- Add: capture by window title or window handle (not just cursor position)
- Add: configurable output resolution (downscale for token efficiency)

**TBD:**
- Should the Rust binary be a long-running daemon (IPC) or invoked per-capture (CLI)? CLI is simpler, daemon avoids startup cost for rapid successive captures.
- How to handle window targeting cross-platform? Title match? PID? Window class? All of the above?

---

## Vision Evaluation

### Model Selection

| Use Case | Model | Why |
|----------|-------|-----|
| Video clips | Gemini 2.5 Pro | Only major model with native video input |
| Screenshots (cost-sensitive) | Gemini 2.5 Flash | Cheaper, still good image understanding |
| Screenshots (quality) | Claude or Gemini Pro | Strong image reasoning |

**TBD:** Should the model be user-configurable? Or should AgentVisual pick the best model per task automatically?

### Evaluation Prompts

The vision model needs structured prompts to produce actionable output. Rough structure:

**For screenshots:**
```
You are evaluating a UI screenshot for visual issues.
Breakpoint: {width}px
Context: {what changed, what page this is}

Check for:
- Alignment and spacing consistency
- Overflow or clipping
- Text readability and truncation
- Visual hierarchy — does the important content stand out?
- Responsive layout — does this look intentional at this breakpoint?

Report issues as a JSON array with: description, severity (critical/warning/info), location (top-left/center/etc.)
```

**For video clips:**
```
You are evaluating a UI interaction video for visual issues.
Interaction: {what happens in this clip}
Context: {what changed, what page this is}

Watch for:
- Layout shifts or content pushing during the interaction
- Janky or stuttered transitions
- Elements appearing/disappearing abruptly instead of smoothly
- Overflow or clipping that appears during state change
- Anything that looks visually broken during the motion

Report issues as a JSON array with: description, severity, timestamp (when in the clip it occurs)
```

**TBD:**
- How to inject project-specific style standards into prompts?
- Should there be a "reference" mode where you show a known-good capture alongside the new one?
- What's the right output format? Structured JSON? Free text? Both?

---

## Cost Estimates

Based on Gemini 2.5 pricing (March 2026):

| Capture Type | Estimated Tokens | Estimated Cost |
|---|---|---|
| Single screenshot (720p, compressed) | ~1,500-3,000 | ~$0.002-0.004 |
| 4 breakpoint screenshots | ~6,000-12,000 | ~$0.008-0.015 |
| 2-second video clip (720p, 15fps) | ~15,000-30,000 | ~$0.02-0.04 |
| Full QA pass (4 screenshots + 3 clips) | ~60,000-100,000 | ~$0.08-0.13 |

**TBD:** These are rough estimates. Need to verify actual token counts for video input with Gemini's API.

---

## User Workflow

### Agent-Driven (Primary Use Case)

The coding agent (Claude Code, Cursor, etc.) calls AgentVisual as part of its own workflow. No human involvement for visual QA.

```
1. Agent makes a UI change
2. Agent calls: screenshot("My App - localhost:5173", breakpoints: [320, 768, 1024, 1440])
3. AgentVisual captures, evaluates, returns: "overflow at 320px, sidebar clips main content"
4. Agent fixes the overflow
5. Agent calls: record("My App - localhost:5173", duration: "2s", interaction_note: "just added a modal toggle button")
6. AgentVisual records, evaluates, returns: "clean, no layout shift on interaction"
7. Agent moves on
```

### Human-Triggered (Secondary)

The developer explicitly asks for a visual check during a conversation:

```
Developer: "Check how this page looks"
Agent calls AgentVisual tools, reports findings
Developer: "The spacing on mobile is fine, ignore that"
Agent adjusts and continues
```

**TBD:**
- Does the agent need to know the dev server URL, or can AgentVisual auto-detect running dev servers?
- For non-browser apps, how does the agent specify which window to capture? By title? By process name?
- Should there be a `.agentvisual.json` config file in the project root with defaults (window title patterns, breakpoints, known-good reference captures)?

---

## Input / Output

### Inputs (from coding agent to AgentVisual)

- **Window identifier** — title, process name, or handle
- **Capture type** — screenshot or video
- **Duration** (video only) — how long to record
- **Options** — FPS, resolution, breakpoints, format
- **Context** (optional) — what changed, what to focus on, custom evaluation prompt

### Outputs (from AgentVisual back to coding agent)

- **File paths** — captured screenshots/video clips saved to temp directory
- **Evaluation results** — structured JSON with:
  - `status`: "pass" | "issues_found"
  - `issues[]`: array of findings
    - `description`: what's wrong
    - `severity`: "critical" | "warning" | "info"
    - `timestamp` (video only): when in the clip
    - `location`: where on screen (rough quadrant or coordinates)
  - `summary`: one-line overall assessment

**TBD:**
- Should AgentVisual clean up temp files automatically, or let the agent manage them?
- Should captured media be base64-encoded in the MCP response, or returned as file paths?
- Max file size / clip duration limits?

---

## Existing Code to Extract

### From QuickShotter (`/BosDev/quickshotter/`)

| Source | What to Extract | Adaptation Needed |
|---|---|---|
| `src-tauri/src/capture.rs` | Screenshot capture, DPI stitching, clipboard | Strip clipboard, add CLI output, add resolution downscale |
| `src-tauri/src/recording/pipeline.rs` | Two-thread capture/encode pipeline | Strip Tauri state, add CLI start/stop control |
| `src-tauri/src/recording/encoder.rs` | VideoEncoder trait, FallbackEncoder | Keep as-is |
| `src-tauri/src/recording/encoder_win.rs` | Media Foundation H.264 | Keep as-is |
| `src-tauri/src/recording/encoder_mac.rs` | VideoToolbox encoding | Keep as-is |
| `src-tauri/src/recording/encoder_cpu.rs` | openh264 fallback | Keep as-is |
| `src-tauri/src/recording/mp4_muxer.rs` | MP4 container writer | Keep as-is |
| `src-tauri/src/window_capture.rs` | Window detection | Adapt: find by title/PID, not cursor |

### From overtooled-mcp (`/BosDev/overtooled-mcp/`)

| Source | What to Extract | Adaptation Needed |
|---|---|---|
| `bin/overtooled-mcp.ts` | MCP server entry point | Rename, swap tool registrations |
| `src/server.ts` | Tool registration pattern | New tools (screenshot, record, evaluate) |
| `src/tools/*.ts` | Tool handler structure | New implementations |
| `src/engines/engine-image.ts` | Sharp image processing | Reuse for screenshot compression |

---

## Development Phases

### Phase 1: Screenshot MVP
- Rust CLI: `screenshot --window "title" --output path.png`
- TypeScript MCP server with `screenshot()` tool
- Gemini Flash evaluation of screenshots
- Returns structured findings to agent

### Phase 2: Video Recording
- Rust CLI: `record --window "title" --duration 2s --fps 15 --output path.mp4`
- MCP `record()` tool
- Gemini Pro evaluation of video clips
- Timestamped findings

### Phase 3: Refinement
- `evaluate()` tool for re-evaluating existing captures with different prompts
- Project config file (`.agentvisual.json`)
- Reference/comparison mode (before vs. after)
- Prompt tuning based on real-world usage

**TBD:** Timeline, priority of features within each phase, what counts as "done" for Phase 1.
