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

**Decision: Keep it primitive.** Let the agent compose `screenshot()` and `record()` calls as needed. A composite `qa_pass()` tool bakes in assumptions about what "a full QA pass" means — which varies by project, by change, and by context. The agent is better positioned to decide what to check. If a common pattern emerges from real usage, a composite tool can be added later without breaking the primitives.

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

**Decision: CLI, not daemon.** Invoked per-capture. Startup cost is negligible for a native binary (< 50ms), and it eliminates an entire class of problems: process lifecycle, IPC protocol, crash recovery, zombie processes. If startup cost ever becomes a bottleneck (unlikely — the vision API call dominates latency), a daemon can be added later without changing the TypeScript bridge interface.

**Window targeting:** Title substring match is the primary method (cross-platform, human-readable). PID as secondary for disambiguation when multiple windows share a title. Window class/handle as platform-specific fallback. `list-windows` returns all three so the agent can pick the right identifier.

---

## Vision Evaluation

### Model Selection

| Use Case | Model | Why |
|----------|-------|-----|
| Video clips | Gemini 2.5 Pro | Only major model with native video input |
| Screenshots (cost-sensitive) | Gemini 2.5 Flash | Cheaper, still good image understanding |
| Screenshots (quality) | Claude or Gemini Pro | Strong image reasoning |

**Decision: Automatic by default, overridable.** The server picks the best model per task (Flash for screenshots, Pro for video) unless the user overrides via `AGENTVISUAL_MODEL` env var or `.agentvisual.json` config (Phase 3). The vision client interface abstracts the provider so adding new models (Claude for images, future video-capable models) is a client implementation swap, not an architecture change.

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

**Decided:**
- Project-specific style standards injected via Layer 3 of the prompt architecture (see "Prompt Architecture" section below). Loaded from `.agentvisual.json` `styleGuide` field.
- Reference screenshots are automatic — the server tracks `lastCapture` per window in session state. No explicit "store reference" step needed. The previous capture IS the reference.
- Gemini 3 `media_resolution`: use HIGH for the current image (what we're evaluating) and LOW for the reference (just needs enough detail for comparison). This keeps the pair cost-efficient while maximizing evaluation quality on the thing that matters.

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

**Decided:**
- AgentVisual captures windows, not URLs. The agent specifies a window title (or PID). It doesn't need to know how the app is served — just what the window is called. `list-windows` lets the agent discover available windows.
- For non-browser apps, same mechanism: window title or PID. All applications have a window title.
- `.agentvisual.json` is Phase 3. Not needed for MVP. The defaults (768px optimization, JPEG quality 80, Gemini Flash for screenshots) are good enough. Project config adds value once users want custom breakpoints, style guide injection, or model overrides.

---

## Cross-Process Contracts

The boundary between the Rust binary and the TypeScript server is the hardest thing to change once both sides are built. These schemas are the API — treat them as load-bearing.

### Rust CLI → TypeScript: Stdout JSON Schemas

Every command outputs exactly one JSON object on stdout. Stderr is reserved for logs/errors.

**`screenshot` output:**
```json
{
  "command": "screenshot",
  "success": true,
  "file": "/tmp/agentvisual/shot-2026-03-09T17-23-01.png",
  "window": {
    "title": "My App - localhost:5173",
    "pid": 12345,
    "width": 1920,
    "height": 1080
  },
  "capture": {
    "width": 1920,
    "height": 1080,
    "scale": 1.0,
    "format": "png",
    "bytes": 245760
  }
}
```

**`record` output:**
```json
{
  "command": "record",
  "success": true,
  "file": "/tmp/agentvisual/clip-2026-03-09T17-23-01.mp4",
  "window": {
    "title": "My App - localhost:5173",
    "pid": 12345,
    "width": 1920,
    "height": 1080
  },
  "recording": {
    "width": 1920,
    "height": 1080,
    "scale": 1.0,
    "fps": 15,
    "duration_ms": 2000,
    "frames": 30,
    "encoder": "media_foundation" | "videotoolbox" | "openh264",
    "format": "mp4",
    "bytes": 512000
  }
}
```

**`list-windows` output:**
```json
{
  "command": "list-windows",
  "success": true,
  "windows": [
    {
      "title": "My App - localhost:5173",
      "pid": 12345,
      "width": 1920,
      "height": 1080,
      "visible": true
    }
  ]
}
```

**Error output (any command):**
```json
{
  "command": "screenshot",
  "success": false,
  "error": {
    "code": "window_not_found",
    "message": "No window matching 'My App' found"
  }
}
```

### Error Taxonomy

Errors are categorized so the TypeScript server (and ultimately the calling agent) can make decisions based on the error type, not parse human-readable strings.

**Rust binary error codes:**

| Code | Meaning | Agent can retry? |
|------|---------|-----------------|
| `window_not_found` | No window matches the given title/PID | No — agent should check window title or call `list-windows` |
| `window_minimized` | Window exists but is minimized/hidden — can't capture pixels | No — agent should note this and skip or ask user |
| `capture_failed` | OS screen capture API returned an error | Maybe — transient OS issue, one retry reasonable |
| `encoding_failed` | Video encoder crashed or produced invalid output | Maybe — will fall back to CPU encoder automatically, but if CPU also fails, no |
| `io_error` | Couldn't write output file (disk full, permissions) | No — system issue |
| `invalid_args` | Bad CLI arguments | No — caller bug |

**TypeScript server error codes (returned to agent via MCP):**

| Code | Meaning |
|------|---------|
| `binary_not_found` | Rust binary not on PATH and `AGENTVISUAL_CAPTURE_BIN` not set |
| `binary_crashed` | Rust process exited non-zero with unparseable output |
| `window_not_found` | Passthrough from Rust binary |
| `window_minimized` | Passthrough from Rust binary |
| `capture_failed` | Passthrough from Rust binary |
| `optimization_failed` | Sharp image processing failed (corrupt image?) |
| `vision_api_error` | Gemini API returned an error (auth, rate limit, malformed response) |
| `vision_api_timeout` | Gemini API didn't respond within timeout |

Every MCP error response includes `code` + human-readable `message`. The agent can branch on `code`; the `message` is for logging.

---

## State Management

### Decision: Session-Scoped State

The MCP server maintains lightweight state for the duration of the MCP connection (one agent session). This enables before/after comparison without the agent managing file paths.

**What's tracked per session:**
- `lastCapture: { file: string, window: string, timestamp: number }` — the most recent screenshot or recording. Enables "compare with previous" mode automatically.
- `captures: Map<string, CaptureRecord>` — all captures in this session, keyed by auto-generated ID. Enables `evaluate()` to reference previous captures by ID.

**What's NOT tracked:**
- Nothing persists across MCP connections. When the agent disconnects, session state is gone.
- No database, no files for state. Pure in-memory.

**Why session-scoped:**
- Stateless (agent manages paths) is simple but makes before/after comparison clunky — the agent has to track and pass file paths for every comparison.
- Persistent state (across sessions) adds complexity with no clear benefit — captures from a previous session are stale.
- Session-scoped hits the sweet spot: automatic before/after within a working session, no cleanup problem, no persistence overhead.

**How before/after works:**
1. Agent calls `screenshot("My App")` — server captures, stores as `lastCapture`, evaluates, returns results.
2. Agent makes a code change.
3. Agent calls `screenshot("My App")` again — server captures new screenshot, sends both previous + current to vision model in comparison mode, updates `lastCapture`.
4. Agent gets results that highlight regressions between the two states.

The agent doesn't need to know this is happening. It just calls `screenshot()` and gets smarter results when a previous capture exists for the same window.

**Override:** The agent can pass `compare: false` to skip comparison, or `compare_with: "<capture_id>"` to compare against a specific earlier capture instead of the most recent one.

---

## File Lifecycle

### Decision: Session-Scoped with Automatic Cleanup

Capture files live in a session-specific temp directory and are cleaned up when the MCP connection closes.

**Structure:**
```
$AGENTVISUAL_TEMP_DIR/          (default: OS temp dir)
  └── session-<uuid>/
      ├── shot-001.png          Full-resolution original
      ├── shot-001-opt.jpg      Optimized version sent to vision API
      ├── clip-002.mp4          Video recording
      └── ...
```

**Lifecycle:**
1. **On MCP connection open** — create `session-<uuid>/` directory.
2. **On each capture** — Rust binary writes to session directory. TypeScript writes optimized copies alongside.
3. **On MCP connection close** — delete the entire `session-<uuid>/` directory.
4. **Crash safety** — on server startup, scan for orphaned `session-*` directories older than 1 hour and delete them.

**Why not let the agent manage cleanup:**
- Agents crash. Files leak. Temp directories fill up.
- The agent has no reason to keep captures after the session — they're stale.
- Automatic cleanup means zero maintenance.

**Escape hatch:** If the agent wants to keep a capture, it can copy the file to a permanent location. The MCP response includes the file path. But by default, captures are ephemeral.

---

## Prompt Architecture

### Decision: Layered and File-Loaded

Evaluation prompts are the core value of the project. They'll be iterated constantly. The architecture must support rapid iteration without rebuilds, while still allowing programmatic composition.

**Three layers, composed at evaluation time:**

```
┌─────────────────────────────────┐
│  Layer 1: Base prompt           │  Built-in. Defines the evaluation framework,
│  (what to look for, how to      │  output JSON schema, severity definitions.
│  report it)                     │  Changes rarely. Lives in source code.
├─────────────────────────────────┤
│  Layer 2: Capture context       │  Auto-generated per call. Breakpoint width,
│  (what was captured, what       │  window title, what the agent said it changed,
│  changed)                       │  comparison mode (before/after).
├─────────────────────────────────┤
│  Layer 3: Project style guide   │  Optional. Loaded from `.agentvisual.json`
│  (project-specific standards)   │  if present. E.g., "this project uses 8px
│                                 │  spacing grid" or "dark theme, no white bg".
└─────────────────────────────────┘
```

**Layer 1 (base prompts)** lives in `server/src/vision/prompts/` as `.txt` files, loaded at startup:
- `screenshot-eval.txt` — base screenshot evaluation prompt
- `video-eval.txt` — base video evaluation prompt
- `comparison-eval.txt` — before/after comparison prompt

Why `.txt` files instead of hardcoded template literals:
- Editable without recompiling TypeScript
- Diffable in git — prompt changes are visible in commit history
- No risk of breaking code syntax while editing prose

**Layer 2 (capture context)** is injected programmatically by the tool handlers. Template variables in the base prompts (`{window_title}`, `{breakpoint}`, `{change_context}`) are replaced at call time.

**Layer 3 (project style guide)** is optional text from `.agentvisual.json`:
```json
{
  "styleGuide": "8px spacing grid. Dark theme only. Minimum touch target 48px. Font sizes: 14px body, 18px headings."
}
```

If present, this text is appended to the base prompt. If absent, the prompt works without it.

**Why not a single hardcoded template:**
- Hardcoded prompts require a rebuild to iterate. During early development, prompt tuning will be the most frequent change.
- But fully dynamic prompts (loaded from user config) lose version control. The layered approach keeps the base prompts versioned in source while allowing project-specific additions.

---

## Input / Output

### Inputs (from coding agent to AgentVisual)

- **Window identifier** — title (substring match), process name, or PID
- **Capture type** — screenshot or video
- **Duration** (video only) — how long to record
- **Options** — FPS, resolution, breakpoints, format
- **Context** (optional) — what changed, what to focus on, custom evaluation prompt
- **Comparison** (optional) — `compare: false` to skip auto-comparison, `compare_with: "<id>"` to compare against a specific earlier capture

### Outputs (from AgentVisual back to coding agent)

- **Capture ID** — session-scoped identifier for referencing this capture later
- **File path** — full path to captured file (in session temp directory)
- **Evaluation results** — structured JSON:
  ```json
  {
    "status": "pass" | "fail",
    "issues": [
      {
        "what": "description of the issue",
        "where": "location on screen (quadrant or description)",
        "severity": "critical" | "warning" | "info",
        "suggestion": "one-line fix recommendation",
        "timestamp": "1.2s"
      }
    ],
    "summary": "one-line overall assessment",
    "comparison": {
      "reference_id": "capture-001",
      "regressions": 2,
      "improvements": 1
    }
  }
  ```
- **Error** (on failure) — `{ "code": "...", "message": "..." }` using the error taxonomy above

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

**Phase 1 is "done" when:** An agent can call `screenshot("window title")` via MCP, get back a structured JSON evaluation with actionable findings, and the full pipeline works end-to-end (Rust capture -> Sharp optimization -> Gemini evaluation -> MCP response). Before/after comparison working. Error handling returns structured codes, not crashes.
