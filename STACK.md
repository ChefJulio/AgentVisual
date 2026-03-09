# AgentVisual — Stack & Foundation

Every dependency listed here is intentional. If it's not on this list, it doesn't go in the project without a reason.

---

## Two-Binary Architecture

AgentVisual is two separate programs that talk to each other:

1. **`agentvisual`** — TypeScript MCP server. Handles tool registration, orchestration, vision API calls, image optimization, and structured output. This is what coding agents connect to.

2. **`agentvisual-capture`** — Rust CLI binary. Handles screen capture and video recording. Invoked by the MCP server as a child process. Does one thing well: turn pixels on screen into files on disk.

They communicate through simple CLI invocation + file paths. No IPC protocol, no sockets, no shared memory. The TypeScript server calls the Rust binary with arguments, the binary writes a file, the server reads it.

**Why two languages:**
- Screen capture requires native OS APIs (Win32, CoreGraphics, X11) and GPU-accelerated encoding. This is Rust territory — it's already proven in QuickShotter.
- MCP protocol, HTTP API calls, JSON manipulation, and prompt construction are all TypeScript strengths. The MCP SDK is TypeScript. No reason to fight that.

---

## Rust: Capture Engine

### Language & Toolchain

- **Rust 2021 edition**
- **Cargo** for build/dependency management
- Target: standalone CLI binary (no Tauri, no GUI framework)

### Core Dependencies

Carried over from QuickShotter — these are proven and tested:

| Crate | Version | Purpose | Notes |
|-------|---------|---------|-------|
| `xcap` | 0.8 | Cross-platform screen capture | GDI/DXGI (Win), AVFoundation (Mac), X11/Wayland (Linux). Already used in QuickShotter. |
| `image` | 0.25 | Image encoding (PNG, JPEG, WebP) | Default features disabled, only enable `png`, `jpeg`, `webp`. |
| `openh264` | 0.6 | Software H.264 fallback encoder | CPU fallback when GPU encoding unavailable. |
| `crossbeam-channel` | 0.5 | Thread-safe channel for capture/encoder pipeline | Bounded buffer between capture thread and encoder thread. |
| `gif` | 0.13 | GIF encoding | For GIF output mode (lower priority). |
| `color_quant` | 1.1 | Color quantization for GIF | Palette reduction. |
| `serde` | 1 | JSON serialization | For structured CLI output (window list, capture metadata). |
| `serde_json` | 1 | JSON formatting | CLI output format. |
| `chrono` | 0.4 | Timestamps | Filename generation, metadata. |
| `thiserror` | 2 | Error types | Clean error handling. |

### Platform-Specific Dependencies

| Crate | Version | Platform | Purpose |
|-------|---------|----------|---------|
| `windows` | 0.62 | Windows | Media Foundation H.264 GPU encoding (NVENC/AMF/QuickSync). Features: `Win32_Media_MediaFoundation`, `Win32_System_Com`, `Win32_Foundation`. |

macOS VideoToolbox encoding uses Objective-C FFI bridge (compiled via `cc` build dependency) — no additional crate needed.

### New Dependencies (Not in QuickShotter)

| Crate | Purpose | Notes |
|-------|---------|-------|
| `clap` | 4.x | CLI argument parsing | QuickShotter used Tauri IPC. AgentVisual needs a proper CLI. |

### Removed from QuickShotter

These were GUI/app-specific and don't belong in a headless capture binary:

- `tauri` and all `tauri-plugin-*` crates
- `arboard` (clipboard — not needed for file output)
- `base64` (was for IPC image transfer to frontend)
- `mouse_position` (was for overlay interaction)
- `dirs` (was for user-facing save paths)

### Build Configuration

```toml
[profile.dev.package."*"]
opt-level = 2
```

Carried from QuickShotter — image encoding and video processing are unusable at debug optimization levels.

### Project Structure (Rust)

```
capture/
├── Cargo.toml
├── src/
│   ├── main.rs              CLI entry point (clap)
│   ├── capture.rs           Screenshot capture, DPI handling
│   ├── window.rs            Window detection and targeting
│   ├── recording/
│   │   ├── mod.rs
│   │   ├── pipeline.rs      Two-thread capture/encode pipeline
│   │   ├── encoder.rs       VideoEncoder trait, FallbackEncoder
│   │   ├── encoder_win.rs   Media Foundation (Windows)
│   │   ├── encoder_mac.rs   VideoToolbox (macOS)
│   │   ├── encoder_cpu.rs   openh264 fallback
│   │   ├── mp4_muxer.rs     MP4 container writer
│   │   └── gif_encoder.rs   GIF output
│   └── error.rs             Unified error types
└── build.rs                 macOS Objective-C compilation (cc)
```

### CLI Interface

```
agentvisual-capture screenshot --window <title|pid> [--output path.png] [--scale 0.5] [--format png|jpeg]
agentvisual-capture record --window <title|pid> --duration <seconds> [--fps 15] [--output path.mp4] [--scale 0.5]
agentvisual-capture list-windows [--json]
```

All output defaults to a temp directory. JSON output on stdout for metadata (dimensions, file path, duration, etc.).

**Capture output format:** The Rust binary always captures at full native resolution (DPI-aware). Downscaling for the vision API happens in the TypeScript layer via sharp — this keeps the Rust binary simple and lets us keep a full-resolution copy on disk while sending an optimized version to Gemini. The `--scale` flag is for cases where even the disk copy should be smaller.

---

## TypeScript: MCP Server

### Runtime & Toolchain

- **Node.js >= 18** (LTS baseline, matches overtooled-mcp)
- **TypeScript 5.5+** (strict mode, ES2022 target)
- **ESM only** (`"type": "module"` in package.json)
- **Vitest** for testing

### Core Dependencies

| Package | Version | Purpose | Notes |
|---------|---------|---------|-------|
| `@modelcontextprotocol/sdk` | ^1.27.0 | MCP server framework | Stdio transport. Same version as overtooled-mcp. |
| `zod` | ^3.23.0 | Input validation | Schema validation for tool parameters. |
| `sharp` | ^0.33.0 | Image processing | Resize/compress screenshots before sending to vision API. Proven in overtooled-mcp. See "Image Optimization Pipeline" below. |

### Vision API Client

| Package | Version | Purpose | Notes |
|---------|---------|---------|-------|
| `@google/generative-ai` | ^0.24.0 | Gemini API client | Official Google SDK for Gemini. Supports inline image data and File API for video upload. |

**Gemini SDK notes:**
- Images: send as inline base64 data (no File API needed for optimized screenshots — they're small after Sharp processing).
- Video: upload via File API first (`fileManager.uploadFile()`), then reference in the prompt. Required for files > 20MB or for video input.
- Max video: 1 hour or 2GB per file via File API. Our clips are 1-3 seconds — well within limits.
- Streaming: `generateContentStream()` available. Worth using for video evaluation where response may be longer. Not needed for screenshot evaluation (responses are small).

**Multi-model support:** Abstract behind a `VisionClient` interface from day one. Start with Gemini implementation only, but the interface makes adding Claude (images) or future video-capable models a new implementation, not a refactor. The interface is simple: `evaluate(media: Buffer | string, prompt: string) => EvaluationResult`.

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.5.0 | Compiler |
| `vitest` | ^2.0.0 | Testing |
| `@types/node` | ^20 | Node.js type definitions |

### Image Optimization Pipeline

`server/src/processing/optimize.ts` — the critical layer between capture and vision API.

Gemini tokenizes images by tiling them into 768x768 chunks at 258 tokens per tile. A raw 1440p screenshot costs ~3,096 tokens (12 tiles). Downscaled to fit within 768x768, it costs 258 tokens (1 tile) — **92% cheaper** with no meaningful loss for UI evaluation.

**Processing steps (sharp):**
1. **Read** the full-resolution PNG from the Rust binary
2. **Resize** to fit within 768x768 (maintain aspect ratio, `sharp.resize({ width: 768, height: 768, fit: 'inside' })`)
3. **Convert** to JPEG quality 80 (smaller file, faster upload — lossless PNG is wasted on vision API input)
4. **Output** the optimized buffer for API upload

The full-resolution original stays on disk untouched. Only the API payload is optimized.

**For video:** No image optimization needed. Gemini tokenizes video at a flat 263 tokens/second regardless of resolution. Send 720p for best evaluation quality — it costs the same as 480p. Only clip duration affects cost.

### Intentionally NOT Including

| Package | Why Not |
|---------|---------|
| `ffmpeg-static` / `ffprobe-static` | Video recording handled by Rust binary, not Node. |
| `puppeteer` / `playwright` | Not a browser automation tool. Screen-level capture via Rust. |
| `canvas` | No server-side rendering. We capture what's already on screen. |
| `express` / `fastify` | MCP uses stdio transport, no HTTP server needed. |
| `pdfkit` / `pdfjs-dist` | No document processing. |

### Project Structure (TypeScript)

```
server/
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── bin/
│   └── agentvisual.ts          MCP server entry point
├── src/
│   ├── index.ts                Public exports
│   ├── server.ts               Tool registration
│   ├── tools/
│   │   ├── screenshot.ts       screenshot() tool handler
│   │   ├── record.ts           record() tool handler
│   │   └── evaluate.ts         evaluate() tool handler
│   ├── capture/
│   │   └── bridge.ts           Spawns Rust binary, parses output
│   ├── vision/
│   │   ├── client.ts           Gemini API client wrapper
│   │   ├── prompts.ts          Evaluation prompt templates
│   │   └── types.ts            EvaluationResult, Issue, etc.
│   ├── processing/
│   │   └── optimize.ts         Sharp-based image optimization before API
│   └── config.ts               Runtime config, env vars, defaults
└── __tests__/
```

---

## Monorepo Structure

Both components live in one repository:

```
AgentVisual/
├── MISSION.md
├── TECHNICAL.md
├── STACK.md                  (this file)
├── capture/                  Rust CLI binary
│   ├── Cargo.toml
│   └── src/
└── server/                   TypeScript MCP server
    ├── package.json
    └── src/
```

**Decision: Independent builds, no monorepo tooling.** `cd capture && cargo build` / `cd server && npm run build`. A workspace-level build script adds complexity for zero benefit when there are only two components. The TypeScript server locates the Rust binary via: (1) `AGENTVISUAL_CAPTURE_BIN` env var if set, (2) `agentvisual-capture` on PATH otherwise. No bundling — the binary is a separate install.

---

## Configuration

### Environment Variables

```
GEMINI_API_KEY          — Required. Gemini API authentication.
AGENTVISUAL_CAPTURE_BIN — Optional. Path to Rust binary. Defaults to looking on PATH.
AGENTVISUAL_TEMP_DIR    — Optional. Where to save captures. Defaults to OS temp directory.
AGENTVISUAL_MODEL       — Optional. Override vision model. Defaults to "gemini-2.5-flash".
AGENTVISUAL_LOG_LEVEL   — Optional. "debug" | "info" | "warn" | "error". Defaults to "info".
```

### Project-Level Config (`.agentvisual.json`)

Optional file in project root for project-specific defaults:

```json
{
  "window": "My App - localhost:5173",
  "breakpoints": [320, 768, 1024, 1440],
  "video": {
    "fps": 15,
    "maxDuration": 5,
    "scale": 0.5
  },
  "model": "gemini-2.5-pro"
}
```

**Phase 3 refinement.** Not needed for MVP. Defaults are sufficient until real usage reveals what needs to be configurable.

---

## Build & Distribution

### Rust Binary

```bash
cd capture
cargo build --release
```

Produces a single static binary. No runtime dependencies (openh264 is compiled in, GPU encoders use OS-provided libraries).

**Cross-compilation targets:**
- `x86_64-pc-windows-msvc` (Windows)
- `aarch64-apple-darwin` (macOS Apple Silicon)
- `x86_64-apple-darwin` (macOS Intel)
- `x86_64-unknown-linux-gnu` (Linux)

### TypeScript MCP Server

```bash
cd server
npm install
npm run build
```

**Distribution: GitHub Releases + npm postinstall download (Option 3).**

npm package published as `agentvisual`. On `npm install` (or `npx`), a postinstall script downloads the correct prebuilt Rust binary from the matching GitHub Release for the user's platform/arch. Same pattern as `esbuild` but simpler — one binary, not a platform-specific npm package per target.

Why not Option 1 (platform-specific npm packages): Complex to set up and maintain. Not worth it until there's significant adoption.
Why not Option 2 (build from source): Too much friction. Users shouldn't need a Rust toolchain to use an MCP server.

For development: build locally with `cargo build --release`. The postinstall download is only for published releases.

---

## Testing Strategy

### Rust

- Unit tests for image encoding, CLI argument parsing, JSON output format
- Integration tests that capture a known window and verify output file
- **No GPU encoder tests in CI** — GPU availability varies. Test the fallback (openh264) in CI, GPU encoding manually.

### TypeScript

- Unit tests for prompt construction, output parsing, image optimization
- Mock the Rust binary for tool handler tests
- Mock the Gemini API for evaluation tests
- Integration tests that run the full MCP server with mock capture + mock vision

### What NOT to Test

- Don't test `xcap` internals (OS screen capture APIs)
- Don't test `sharp` internals (image processing library)
- Don't test Gemini API behavior (external service)
- Test the glue, the prompts, the output parsing, and the orchestration

---

## Foundation Priority

What to build first, in order. Each layer is usable before the next one starts:

### Layer 1: Rust CLI — Screenshots Only
- `agentvisual-capture screenshot --window "title" --output path.png`
- `agentvisual-capture list-windows --json`
- Window targeting by title (exact and substring match)
- PNG output with optional scale factor
- JSON metadata on stdout

### Layer 2: TypeScript MCP Server — Screenshot Tool
- MCP server with `screenshot()` tool
- Spawns Rust binary, collects full-resolution PNG
- Sharp optimization: resize to 768x768 max, JPEG quality 80 (258 tokens per image vs ~3,000 raw)
- Gemini Flash API call with structured evaluation prompt + JSON response schema
- Before/after comparison mode when reference capture exists (516 tokens for the pair)
- Returns structured JSON findings to agent

### Layer 3: Rust CLI — Video Recording
- `agentvisual-capture record --window "title" --duration 2s --fps 15 --output path.mp4`
- GPU-accelerated H.264 with CPU fallback
- Two-thread pipeline from QuickShotter

### Layer 4: TypeScript MCP Server — Video Tool
- MCP `record()` tool
- Send 720p clips directly to Gemini Pro (resolution doesn't affect token cost for video — only duration does, at 263 tokens/sec)
- Video evaluation prompt with JSON response schema + timestamped observations

### Layer 5: Evaluate Tool & Refinement
- `evaluate()` tool for re-running vision on existing captures
- Project config file support
- Prompt tuning based on real usage
