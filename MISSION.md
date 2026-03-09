# AgentVisual

## Mission

Give AI coding agents video-based visual perception of their own UI output — not screenshots, not pixel diffs, but actual video of the interface in motion — so they can catch the problems that only reveal themselves through interaction and movement.

## The Problem

AI coding assistants are blind. They write CSS, build layouts, wire up animations — and have zero idea what any of it actually looks like. The current workaround is the human: you look at the screen, notice something's off, screenshot it, describe it, feed it back. You are the visual cortex.

Screenshots help, but they miss an entire class of bugs that only exist in motion:

- Layout shifting when a button is clicked
- Content pushing down on state change
- Janky resize behavior between breakpoints
- Hover/focus transitions that feel wrong
- Scroll-triggered layout breaks
- Modal open/close causing reflow
- Animation timing that looks off

These are **temporal, interaction-driven problems**. A static image can't capture them. You need to see the thing *move*.

## The Solution

Short, targeted video recordings of UI interactions, fed to a vision-capable LLM for evaluation.

Not continuous screen recording. Not a stream. **Precise clips** — record only the window of time needed to capture the interaction being tested:

- Resize from 1440px down to 320px over 3 seconds — record that
- Click a button that triggers a state change — record the 1-2 seconds around it
- Open a modal, interact, close it — record that sequence
- Scroll through a page — record the scroll

Trim it tight, keep resolution and framerate sane (720p, 15-30fps is plenty for UI evaluation), and send the clip to a vision model that can watch it and say: "At 1.2 seconds, the sidebar collapses and pushes the main content down by 40px before snapping back. That's a layout shift bug."

## What This Is NOT

- Not a screenshot tool (Claude Code already does that)
- Not a pixel-diff regression framework (Playwright/Cypress do that)
- Not continuous screen recording (too expensive, too noisy)
- Not a browser automation agent (Computer Use drives the mouse — this watches the result)

## What This IS

A **video-based visual QA layer** for AI coding loops. The AI writes code, the UI renders, AgentVisual records targeted interaction clips, and the vision model evaluates what it sees — catching the temporal, motion-based problems that no amount of code reading or screenshot analysis can detect.

## Core Concept

```
Code change (AI or human)
  --> UI hot reloads
  --> AgentVisual triggers targeted recordings:
        record: resize from desktop to mobile (3s clip)
        record: click primary action button (1s clip)
        record: open/close modal (2s clip)
        record: scroll full page (2s clip)
        record: hover interactive elements (1s clip each)
  --> trim clips tight, encode at 720p / 15-30fps
  --> send to vision model with context:
        "Watch this clip. Evaluate for layout shifts, reflow,
         animation jank, overflow, clipping, or anything that
         looks visually broken during the interaction."
  --> model returns timestamped observations
  --> feed back into coding loop for fixes
```

## Cost Control

Video tokens are expensive. The entire design is built around minimizing them:

- **Short clips only** — 1-3 seconds per interaction, not minutes of footage
- **Lower resolution** — 720p is sufficient for UI evaluation, maybe even 480p
- **Reduced framerate** — 15fps captures motion without waste. No need for 60fps
- **Targeted recording** — only record during the specific interaction being tested. Don't record idle time, typing, or reading
- **Interaction-driven** — the system knows exactly when to start and stop recording based on what's being tested

The goal is clips measured in seconds and megabytes, not minutes and gigabytes.

## Open Questions

1. **Taste calibration** — How does the model learn what "correct" looks like for a specific project? Reference clips of good behavior? Style guide? Feedback rounds?
2. **Interaction scripting** — What triggers the recordings? Can it infer meaningful interactions from the component tree, or does the user define test sequences?
3. **Temporal understanding** — Can current vision models reliably spot a 200ms layout shift in a video clip? How good is their frame-by-frame analysis?
4. **Integration model** — Where does this live? MCP server? Claude Code hook? VS Code extension? Standalone tool?
5. **Feedback format** — How does the model report issues back? Timestamped descriptions? Annotated frame grabs from the video? Directly mapped to source code?

## Success Criteria

An AI coding agent can:

1. Make a UI change
2. AgentVisual automatically records targeted interaction clips
3. The vision model watches the clips and identifies motion-based visual bugs
4. The agent fixes them without a human ever opening a browser
5. The human only intervenes for design decisions — not for catching a div that pushes content around when clicked

## Tech Stack (TBD)

- **Screen/window recording** — targeted capture of browser window during interactions
- **Video encoding** — trim, compress, optimize for token efficiency
- **Vision model API** — Claude, Gemini, or whatever handles video input best
- **Orchestration** — trigger recordings, manage clips, feed to model, return results
- **Integration** — hook into existing AI coding workflows
