# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file IT PMO Kanban board, used as an internal demo/training tool for a fictitious bank. Everything lives in `index.html`: the markup, one `<style>` block and one `<script>` block.

There are two versions, each self-contained and deployed side by side by `.github/workflows/pages.yml`:
- `index.html`: the classic board, served at the Pages root. Leave it unchanged unless asked.
- `v2/index.html`: the redesign, served at `/v2/`. It has the same script architecture, plus a "Board health" strip of SVG ring charts. `renderSummary(tasks)` draws them from the filtered task list. It also has its own palette tokens: status hues and a priority ramp, validated for colour-blind separation. It links back to the classic board via `../index.html`.

## Running

There is no build, lint or test tooling, and no package manager. To run the app, open `index.html` directly in a browser (on Windows: `Start-Process index.html`). Node is not installed on this machine, so the JS can't be syntax-checked from the CLI. Verify changes in the browser console instead.

## Hard constraints (from the original spec — keep these)

- Vanilla HTML/CSS/JS only: no frameworks, CDNs, external fonts, image files, build step or npm. Icons are Unicode glyphs or inline SVG.
- **No persistence.** Don't use localStorage, sessionStorage, IndexedDB or cookies. A refresh resetting the board to seed data is intended, and the UI says so.
- The only network call is the FormSubmit AJAX endpoint. Never send anything elsewhere.
- Don't use UOB's real logo or trademarks, and don't imitate an official UOB system. The wordmark is plain text "IT PMO" in a corporate blue palette. Task IDs use the format `UOB-ITPM-####` as specified.
- Don't use native `alert()` or `confirm()`. Validation errors appear inline under each field, and delete is confirmed with an inline "Delete? Yes / No" toggle on the card.
- CSS uses custom properties for palette and spacing (tokens in `:root`) and no `!important`.

## Architecture (script block)

- **Single source of truth:** `state = { tasks, filters, ui, nextId }`. `state.ui` holds transient UI state: the open move menu, the pending delete confirmation, a focus target to apply after the next render, and the in-flight send flag.
- **Render from state:** `renderBoard()` rebuilds the whole board's HTML and also calls `renderSummary()`, `renderFilterStatus()` and `restoreFocus()`. Don't mutate card contents directly. Change `state`, then call `renderBoard()`. The only direct DOM tweaks allowed are the drag classes (`is-dragging`, `drop-target`) and toasts.
- **Focus management:** re-rendering destroys the focused element. Before rendering, handlers set `state.ui.focusTarget` to a CSS selector, and `restoreFocus()` focuses it afterwards. Keep doing this for any new card interaction so keyboard users stay oriented.
- **Event delegation:** clicks and native HTML5 drag-and-drop events are bound once on `#board`. Card buttons carry `data-action` values (`toggle-move`, `move-to`, `ask-delete`, `confirm-delete`, `cancel-delete`) that are handled in `handleBoardClick()`. Escape closes an open menu or confirmation.
- **XSS:** every user-supplied string must pass through `escapeHtml()` before going into template HTML.
- **Dates:** dates are stored as local `YYYY-MM-DD` strings and compared as strings (`isOverdue`, due-date validation). Seed tasks use `offsetDate(n)`, so overdue examples stay overdue whatever the current date.
- **Reference lists** (`STATUSES`, `PROJECTS`, `CATEGORIES`, `PRIORITIES`) populate both the form selects and the filter selects. Change them in one place. `FORM_FIELDS` maps each form key to its input id, and the matching error element id is `${id}-error`.
- **Add flow (optimistic):** `handleFormSubmit` validates the input, calls `addTask()` (the card renders immediately), resets the form, then awaits `notifyNewTask()`. If that call fails, a warning toast appears and the card stays. The submit button shows "Sending…" while the request is in flight.

## FormSubmit

- The `FORMSUBMIT_ENDPOINT` constant at the top of the script holds the recipient address. While it is still the placeholder `YOUR_EMAIL@example.com`, `notifyNewTask()` deliberately throws without making a request.
- A new address needs one-time activation: the first submission sends a confirmation email, and nothing is delivered until its link is clicked.
- FormSubmit can return HTTP 200 with `success: "false"`. The code treats that as a failure.
- Opened via `file://`, requests come from a `null` origin, which FormSubmit may reject. Serve the file over HTTP if email delivery matters.
