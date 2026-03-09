# Visual QA Loop

## Mission

Eliminate the human visual debugging bottleneck in AI-assisted frontend development.

When an LLM writes UI code, the human currently serves as the eyes — screenshot, describe the problem, feed it back. This project closes that loop automatically: the AI writes code, sees the rendered result, evaluates it visually, and self-corrects — without a human ever looking at the screen.

## The Problem

AI coding assistants are blind. They can write CSS, generate layouts, build responsive UIs — but they have no idea what the result actually looks like. Every visual bug requires a human to:

1. Notice it
2. Screenshot or describe it
3. Relay it back to the AI
4. Wait for a fix
5. Check again

This is the slowest part of the AI-assisted frontend workflow. The code loop is fast (write, run tests, fix). The visual loop is manual and tedious.

## The Solution

An automated visual QA system that gives the AI eyes on its own output:

- **After a code change**, automatically render the result in a headless browser
- **At multiple breakpoints** (320px, 768px, 1024px, 1440px)
- **Through interactive states** (click buttons, open modals, hover, scroll, type in inputs)
- **Capture each state** and feed it to a vision model
- **Evaluate against criteria**: alignment, spacing, overflow, reflow, visual hierarchy, interaction side effects (content pushing, layout shifts)
- **Self-correct** or report findings — then loop until clean

## What This Is NOT

- Not a screenshot transporter (Claude Code can already read screenshots)
- Not a pixel-diff regression tool (Playwright/Cypress already do that)
- Not a browser automation agent (Computer Use already drives mouse/keyboard)
- Not continuous screen recording for LLMs (economically impractical, low signal-to-noise)

## What This IS

A **taste-aware visual judgment layer** that sits inside the AI coding loop. It answers questions that code analysis can't:

- "Does this *look* right?"
- "Does the layout break when resized?"
- "Does clicking this button cause ugly reflow?"
- "Is the spacing consistent with the rest of the page?"
- "Does this modal clip on mobile?"

## Key Differentiator

Existing tools catch regressions against a known baseline. This catches **subjective visual issues** — things that are technically valid CSS but look wrong to a human. The vision model provides the judgment that automated pixel-diffing can't.

## Core Architecture (Conceptual)

```
Code change (from AI or human)
  --> hot reload / rebuild
  --> Playwright headless browser
  --> scripted interaction sequences:
        resize to each breakpoint --> capture
        trigger interactive states --> capture
        scroll through page       --> capture
  --> batch screenshots to vision model
  --> evaluate against:
        - project style standards
        - before/after comparison
        - explicit criteria (no overflow, no reflow, consistent spacing)
  --> output: pass / list of visual issues with descriptions
  --> if integrated with coding AI: auto-fix and re-loop
```

## Open Questions

- **Taste calibration**: How does the AI learn what "correct" looks like for a specific project? Reference screenshots? Style guide docs? Feedback rounds?
- **Integration model**: Standalone tool? VS Code extension? Claude Code hook? MCP server?
- **Interaction scripting**: How much does the user need to define vs. how much can be inferred from the component tree?
- **Cost**: Each QA pass = multiple screenshots to a vision API. What's the per-iteration cost and how to keep it sustainable?
- **Scope**: Start with static layout QA? Or tackle animations/transitions from the start?

## Tech Stack (Likely)

- **Playwright** — headless browser, screenshot capture, interaction scripting
- **Claude Vision API** — visual evaluation and judgment
- **TypeScript/Node** — orchestration layer
- **Integration target** — Claude Code hooks, MCP server, or VS Code extension

## Success Criteria

The project succeeds if an AI coding assistant can:

1. Make a CSS/layout change
2. Automatically verify it looks correct across breakpoints and interaction states
3. Catch and fix visual bugs without a human ever opening a browser

The human should only need to intervene for genuine design decisions — not for catching a misaligned div.
