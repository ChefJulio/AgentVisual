# AgentVisual

## Mission

Give AI coding agents visual perception of their own output — so they can see what they build, judge whether it looks right, and fix what doesn't — without a human ever needing to look at the screen.

## The Problem

AI coding assistants are blind. They write CSS, build layouts, wire up animations — and have zero idea what any of it actually looks like. The current workaround is the human: you look at the screen, notice something's off, screenshot it, describe it, feed it back. You are the visual cortex.

This is the slowest part of AI-assisted development. The code loop is fast — write, test, fix. The visual loop is manual and tedious. Every visual bug requires a human to notice it, capture it, relay it, and verify the fix. The AI can't close this loop on its own.

## What AgentVisual Does

AgentVisual gives the AI eyes. It captures what the UI actually looks like — both as screenshots and as video — and feeds it to a vision model that can evaluate what it sees.

**Screenshots** catch static issues: broken layouts, alignment problems, overflow, spacing at different screen sizes. They're cheap, fast, and effective for the basics.

**Video** catches everything else — the problems that only exist in motion:

- Layout shifting when a button is clicked
- Content pushing down on state change
- Janky resize behavior between breakpoints
- Hover/focus transitions that feel wrong
- Scroll-triggered layout breaks
- Modal open/close causing reflow
- Animation timing that looks off

These are temporal, interaction-driven problems. A static image can't capture them. You need to see the thing move.

The natural workflow is escalation: run the cheap screenshot pass first, and if something looks off or the change involves interaction/animation, trigger targeted video clips to evaluate the motion.

## Not Just Browsers

This works at the screen level, not the browser level. Any visual application benefits:

- Web apps (React, Vue, Svelte — via dev server)
- Desktop apps (Tauri, Electron, native)
- Mobile emulators
- Terminal UIs
- Design tools
- Game engines

Playwright-based tools are limited to headless browser capture. AgentVisual captures the actual window on screen — whatever the application, whatever the framework.

## What This Is NOT

- **Not a screenshot relay** — Claude Code can already read screenshots you paste. AgentVisual is automated and agent-driven, not human-triggered.
- **Not a pixel-diff regression tool** — Playwright/Cypress compare against known baselines. AgentVisual makes subjective visual judgments about whether something looks right.
- **Not continuous screen recording** — too expensive, too noisy. AgentVisual records precise, short clips around specific interactions.
- **Not a browser automation agent** — Computer Use drives the mouse. AgentVisual watches the result.

## What This IS

A visual QA layer for AI coding loops. The AI writes code, the UI renders, AgentVisual captures what it needs to — stills or clips — and a vision model evaluates what it sees, catching problems that code analysis alone can never detect.

The key insight: existing tools catch regressions against a known baseline. AgentVisual catches subjective visual issues — things that are technically valid code but look wrong to a human eye. The vision model provides the judgment that automated pixel-diffing can't.

## The Vision

An AI coding agent can:

1. Make a UI change
2. AgentVisual automatically captures screenshots and/or targeted video clips
3. A vision model evaluates stills for static issues, video for motion-based bugs
4. The agent fixes what it finds without a human ever opening a browser
5. The human only intervenes for genuine design decisions — not for catching a div that pushes content around when clicked

## Cost Philosophy

Video tokens are expensive. The entire design is built around getting maximum visual insight for minimum token cost:

- **Short clips only** — 1-3 seconds per interaction, not minutes of footage
- **Lower resolution** — 720p or even 480p is sufficient for UI evaluation
- **Reduced framerate** — 15fps captures motion without waste
- **Targeted recording** — only record during the specific interaction being tested
- **Screenshots first** — use the cheap pass to triage, escalate to video only when needed

The goal is clips measured in seconds and megabytes, not minutes and gigabytes.

## Open Questions

1. **Taste calibration** — How does the model learn what "correct" looks like for a specific project? Reference clips of good behavior? Style guide documents? Feedback rounds?
2. **Interaction scripting** — What triggers the recordings? Can the system infer meaningful interactions, or does the user/agent define test sequences?
3. **Temporal understanding** — How reliably can current vision models spot a 200ms layout shift in a video clip? What's the minimum framerate/resolution needed?
4. **Feedback format** — How does the model report issues back? Timestamped descriptions? Annotated frame grabs? Directly mapped to source code locations?
