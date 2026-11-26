---
name: notebook-infographics
description: 'Create NotebookLM-style SVG infographics for notebooks, READMEs and docs. Use when asked to create, redesign or replace a diagram, flowchart, architecture picture, concept overview or "infographic"; when a Mermaid diagram renders badly in light or dark themes; or when a notebook needs a visual summary. Produces a theme-independent, hand-authored SVG in docs/images/ and embeds it with a relative Markdown image link.'
---

# Notebook infographics

Hand-authored SVG infographics in the style of NotebookLM: a short visual story told with numbered cards, soft colour, simple icons and very little text. SVG renders identically in GitHub, VS Code, Jupyter and docs sites, diffs cleanly in git and never depends on the reader's theme.

## When to use which format

| Need | Use |
|---|---|
| Concept overview, pipeline, "how it works" | This skill (SVG infographic) |
| Tiny flow (≤ 5 boxes) that changes often | Mermaid, but give **every** node an explicit `fill`, `stroke` and `color` |
| Data plots | matplotlib in a code cell, not an infographic |

## Workflow

1. **Find the story.** Read the notebook's intro and steps. Write one title (≤ 7 words), one subtitle sentence, and 3–6 steps of *heading (1–3 words) + body (≤ 3 short lines)*. Add one concrete example strip if it helps (real names from the notebook's data).
2. **Lay it out** on a fixed canvas using the grid below. Left-to-right for pipelines, top-to-bottom for decision flows.
3. **Write the SVG** to `docs/images/<notebook-topic>.svg` using the style guide. No external fonts, images or scripts.
4. **Embed it** in the notebook's first Markdown cell with a relative path, replacing the Mermaid block:
   `![<alt text describing the whole flow>](../docs/images/<name>.svg)`
5. **Check contrast** (see Validation) before handing back.

## Style guide

**Canvas:** `viewBox="0 0 1280 H"` (H usually 720–800). Draw a full-size background `rect` with `rx="28"` and fill `#F7F8FC`. That panel is what makes the image theme-independent, so never omit it.

**Grid:** 64 px outer margin, 32 px gutters. Four cards across = 264 px wide each.

**Type:** `font-family="Segoe UI, Inter, Helvetica, Arial, sans-serif"`.

| Role | Size | Weight | Colour |
|---|---|---|---|
| Title | 34 | 700 | `#1F2430` |
| Subtitle | 18 | 400 | `#5B6272` |
| Card heading | 22 | 700 | `#1F2430` |
| Body | 15–16 | 400 | `#5B6272` |
| Labels on arrows | 13 | 600 | `#5B6272` |

Keep body lines ≤ 26 characters on a 264 px card; break lines manually with `<tspan x=… dy="24">`.

**Palette** (accent / soft fill): pick one accent per concept and reuse it consistently across all infographics in the repo.

| Name | Accent | Soft |
|---|---|---|
| Blue (data, indexes) | `#3B6FD8` | `#E8F0FE` |
| Amber (models, processing) | `#D98E1C` | `#FFF4E0` |
| Green (agents, answers) | `#2E9E6A` | `#E6F6EE` |
| Purple (queries, code) | `#7B5BD6` | `#F0EBFD` |
| Coral (places, warnings) | `#D9534F` | `#FDECEC` |

**Cards:** white `rect`, `rx="20"`, 2 px stroke in the soft-fill colour darkened (or the accent at 40 % opacity). Numbered badge top-left (circle r=18, accent fill, white number); icon top-right (circle r=24, soft fill, icon drawn with the accent as a 2.5 px stroke, `stroke-linecap="round"`).

**Icons:** draw simple line icons inline (document, database, graph, sparkle, chat, code brackets, loop). Fit them in a 24 × 24 box. No emoji, no icon fonts.

**Connectors:** `#9AA3B5`, 2.5 px, with a single shared arrow `marker`. Label only when the label adds meaning.

**Callouts:** for a key insight, use a dashed band in the concept's soft fill with accent stroke, one sentence.

**Example strip:** "pills" (rounded rects, soft fill, accent text) joined by labelled arrows, e.g. `Yuri Gagarin —crewOf→ Vostok 1`.

## Validation

- Render the SVG on **both** a white and a near-black page (an HTML file with two `<img>` tags, opened with the browser tools) and take a screenshot. Every word must stay readable and no element may vanish into the background.
- Check that no text overflows its card, that arrows touch their cards, and that the alt text describes the full flow for screen-reader users.
- Keep the file small (no embedded raster images) and readable (indented, grouped with `<g>` per card, short comments naming each section).
