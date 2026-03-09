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

The vision model needs structured prompts to produce actionable output. Two principles drive prompt quality:

**1. Tell it what changed.** Without context, the model evaluates everything generically and produces vague findings. With context about what the developer modified, it focuses on the area most likely to be broken.

**2. Force structured JSON output.** Free-text evaluations are unreliable to parse and act on programmatically. JSON schema ensures the agent can reliably read and respond to findings.

**For screenshots:**
```
You are evaluating a UI screenshot for visual issues.
Breakpoint: {width}px
Context: {what the developer changed, what page this is}

Evaluate ONLY for:
1. Does any content overflow or clip beyond the viewport edge?
2. Is any text truncated or overlapping other elements?
3. Are interactive elements (buttons, inputs) at least 44px tall for touch targets?
4. Is the vertical spacing between sections consistent?
5. Does the layout look intentional at this breakpoint, or broken?

For each issue, report:
- WHAT: specific element description (e.g., "the Save button in the bottom-right")
- WHERE: quadrant of the screen (top-left, center, bottom-right, etc.)
- SEVERITY: critical (broken/unusable), warning (ugly but functional), info (minor polish)
- SUGGESTION: one-line fix recommendation

Response format (JSON):
{
  "status": "pass" | "fail",
  "issues": [
    {
      "what": "string",
      "where": "string",
      "severity": "critical" | "warning" | "info",
      "suggestion": "string"
    }
  ],
  "summary": "one-line overall assessment"
}
```

**For video clips:**
```
You are evaluating a UI interaction video for visual issues.
Interaction: {what happens in this clip}
Context: {what the developer changed, what page this is}

Watch for:
- Layout shifts or content pushing during the interaction
- Janky or stuttered transitions
- Elements appearing/disappearing abruptly instead of smoothly
- Overflow or clipping that appears during state change
- Anything that looks visually broken during the motion

Response format (JSON):
{
  "status": "pass" | "fail",
  "issues": [
    {
      "what": "string",
      "where": "string",
      "severity": "critical" | "warning" | "info",
      "suggestion": "string",
      "timestamp": "1.2s"
    }
  ],
  "summary": "one-line overall assessment"
}
```

### Before/After Comparison Mode

Sending two screenshots (before and after a change) dramatically improves issue detection. The model is much better at spotting regressions when it has both states to compare:

```
Image 1: the page BEFORE the change (reference).
Image 2: the page AFTER the change (current).
Identify any visual regressions introduced by the change.
```

At 258 tokens per optimized image, a before/after pair costs 516 tokens — negligible. This should be the default mode for screenshot evaluation when a reference capture exists.

**TBD:**
- How to inject project-specific style standards into prompts?
- When/how to capture and store reference screenshots for comparison?
- Gemini 3 models support per-part `media_resolution` — should reference images use LOW and the current image use HIGH?

---

## Gemini Tokenization & Cost Optimization

Understanding how Gemini counts tokens is critical for keeping costs low.

### How Gemini Tokenizes Media

**Images:**
- Both dimensions <= 384px → **258 tokens** (flat rate, single tile)
- Larger images → cropped into **768x768 tiles**, each tile = **258 tokens**
- A raw 1440x900 screenshot ≈ 4 tiles ≈ 1,032 tokens
- A 768x768 image = 1 tile = 258 tokens (the sweet spot)

**Video:**
- **263 tokens per second** (fixed rate — resolution does NOT affect token count)
- A 2-second clip = 526 tokens regardless of whether it's 480p or 1080p
- Only duration matters for video cost

**Gemini 3 models** add a `media_resolution` parameter (low/medium/high/ultra_high) that can be set per-part within a single request.

### Optimization Rules (Implemented in `server/src/processing/optimize.ts`)

**Screenshots — downscale to single-tile sweet spot:**

| Strategy | Resolution | Tokens | vs. Raw |
|---|---|---|---|
| Raw 1440p capture | ~2560x1440 | ~3,096 (12 tiles) | baseline |
| Downscale to 768px wide | 768x~432 | 258 (1 tile) | **92% cheaper** |
| Downscale to 1024x768 | 1024x768 | 516 (2 tiles) | 83% cheaper |

Rule: **always downscale screenshots to fit within 768x768 before sending to Gemini.** For UI evaluation (layout, spacing, alignment), this resolution is sufficient. Capture at full resolution for the file on disk, downscale only for the API call.

**Video — keep clips short, don't reduce resolution:**
- Resolution has zero effect on token cost for video
- Send 720p for better evaluation quality — it's free
- The only lever is duration: every second = 263 tokens
- 1-3 second clips are the target

**Format:**
- **JPEG quality 80 for screenshots** — smaller upload, faster transfer, no perceptible quality loss for vision evaluation. Never send PNG to the API (lossless is wasted on vision input).
- **MP4 H.264 for video** — standard, well-supported. Low bitrate is fine for UI content (low motion complexity).

### Actual Cost Estimates

Based on Gemini 2.5 pricing ($1.25/M input tokens for Pro, $0.30/M for Flash):

| Capture Type | Tokens (Optimized) | Cost (Flash) | Cost (Pro) |
|---|---|---|---|
| Single screenshot (768px, JPEG) | 258 | $0.00008 | $0.0003 |
| 4 breakpoint screenshots | 1,032 | $0.0003 | $0.001 |
| Before/after screenshot pair | 516 | $0.00015 | $0.0006 |
| 2-second video clip | 526 | $0.00016 | $0.0007 |
| 3-second video clip | 789 | $0.00024 | $0.001 |
| Full QA pass (4 screenshots + 3 video clips + before/after) | ~3,100 | $0.001 | $0.004 |

A full QA pass costs under a penny. During development, even aggressive testing (50 full passes in a session) stays under $0.20.

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
