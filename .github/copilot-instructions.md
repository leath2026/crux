# Copilot instructions for crux

## Project overview
- This repo is a static HTML snapshot of the CRUX Handbook (no app code, build system, or tests).
- The primary artifact is [CRUX _ Main _ Handbook3-8.html](CRUX%20_%20Main%20_%20Handbook3-8.html); treat it as source-of-truth content.

## Editing conventions
- Keep the existing PmWiki-generated structure intact (e.g., header blocks, toc markup, anchors like `#ntoc*`).
- Prefer minimal, localized edits; do not reflow or re-indent large sections of HTML.
- Preserve external references (CSS/JS/CDN links to crux.nu) unless explicitly asked to update them.
- Maintain inline formatting choices already present (e.g., `<span style='color: #007a00;'>`, `<pre>` blocks, and inline `<code>`).

## Files to know
- [CRUX _ Main _ Handbook3-8.html](CRUX%20_%20Main%20_%20Handbook3-8.html): full handbook content.
- [README.md](README.md): currently minimal, no extra workflows.

## Workflows
- No build, test, or lint commands are defined in this repo.
- Edits are manual; validate by visual inspection if needed.
