# CLAUDE.md — Slam Bracket

Source of truth for this codebase. Read before touching any code.

---

## 1. App Overview

Slam Bracket is a single-page web app for running a Grand Slam tennis bracket pick pool. A user uploads a TNNS Live draw PDF, reviews the parsed Round 1 matches, makes picks for every match before the tournament starts, then locks the bracket. During the tournament they confirm match results (✓/✗) and make backup picks when their original picks are eliminated. Multiple draws (Men's/Women's) for multiple slams can coexist in one session, accessible via a dropdown and segmented M/W control. The app is fully client-side with no backend — persistence is localStorage (autosave) plus a scoped JSON download/upload. A future multiplayer version using Supabase is planned.

---

## 2. Tech Stack

- **Single file:** all HTML, CSS, and JS live in `index.html`. No build step, no bundler, no modules.
- **PDF.js 3.11.174** — loaded from `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js` (worker from same CDN path). Used only in `extractPdfText()`.
- **Google Fonts** — Playfair Display (ital,wght 400/600), DM Mono (400/500), DM Sans (300/400/500/600). Loaded via `<link>` in `<head>`.
- **localStorage** — autosave under key `slam_bracket`. Storage warning threshold at 4 MB.
- **No other runtime dependencies.** No React, no framework, no npm packages.
- **PWA:** `manifest.json` + `icon.png` / `icon-192.png` / `icon-512.png`.
- **Print target:** A3 portrait, generated as a new window document via `buildPrintHTML()`.

---

## 3. Module Map

Everything is in `index.html`. Logical sections in order:

**CSS / Design tokens (lines ~17–290).** Owns all visual styles. Defines CSS custom properties under `:root` and slam theme overrides as `body.theme-AO` etc. Must not contain any logic. Never hardcode colors in JS — use CSS variables. Exception: `buildPrintHTML()` uses inline hex colors because it generates a standalone document.

**State & utility helpers (lines ~408–426).** Owns `state` (the single mutable object), `pdfFile`, `pendingDraw`, the `$()` shorthand, `showScreen()`, `activeDraw()`, `applyTheme()`, `drawLabel()`, `drawShortLabel()`. These are the primitive foundation — never duplicate them.

**Match helpers & scoring (lines ~428–547).** Owns `isBackupPick()`, `isOrigPick()`, `ROUND_CONFIG`, `numericSeed()`, `calcUpsetBonus()`, `calcMatchScore()`, `calcStatsAsOf()`, `calcStats()`. Pure functions — no DOM, no state mutation. `statsRoundFilter` (the round snapshot selector) lives here as a module-level variable.

**Stats rendering (lines ~549–679).** Owns `renderStats()`. Reads state + `statsRoundFilter`. Writes only to `#stats-strip` and `#stat-desc-bar`. Must not touch bracket DOM.

**Lock / unlock (lines ~681–749).** Owns `lockDraw()`, `doUnlock()`, `unlockDraw()`, `updateLockBtn()`. Lock snapshots `originalPick` on every match. Unlock with results clears winners, resets picks to `originalPick`, and re-cascades. Must always call `saveToStorage()` → `renderStats()` → `renderBracket()` on completion.

**Pick cascade (lines ~751–838).** Owns `findSeed()`, `getNextSlot()`, `placePickInNextRound()`, `placePickAllRounds()`, `clearPickForward()`, `handlePickClick()`. Pre-lock picks cascade forward through subsequent rounds. Post-lock picks are backup-only (no cascade) unless `editedAfterLock` is true (withdrawal repick). Must not be called on locked, non-withdrawal matches.

**Winner application (lines ~840–928).** Owns `applyWinner()`, `undoWinner()`, `markLoserForward()`, `unmarkLoserForward()`. Confirms match results post-lock. `markLoserForward` walks downstream and sets `elim:true` on player slots (visual only, no scoring impact). `undoWinner` reverses all of this. Must always call `saveToStorage()` → `renderStats()` → `renderBracket()`.

**Modals (lines ~930–1041).** Owns `openWinnerPicker()`, `closeModal()`, `editCtx`, `openEditPlayerModal()`, `confirmEditPlayer()`, `withdrawalClearForward()`, `updatePlayerNameForward()`. The single `#edit-player-modal` DOM element is reused for both winner-picking and player editing. `closeModal()` resets the modal to player-edit state. Withdrawal post-lock sets `editedAfterLock:true` on the match and walks forward clearing the withdrawn player.

**Storage (lines ~1043–1108).** Owns `saveToStorage()`, `triggerDownload()`, `mergeDraws()`, `migrateState()`. `saveToStorage()` writes all of `state` to localStorage. `triggerDownload()` scopes the export to just the active slam+year. `mergeDraws()` replaces matching draws (same slam+draw+year) and adds new ones. `migrateState()` is the canonical place to handle legacy field shapes — add defensive defaults here, not inline.

**PDF parser (lines ~1110–1158).** Owns `extractPdfText()`, `parseTnnsText()`, `buildInitialRounds()`. Regex-based; expects TNNS Live positional format. `parseTnnsText` extracts 128 player slots by position, pairs them into 64 R1 matches. Currently stores names as `"Lastname F."` (first initial only) — full first names are the target format. `buildInitialRounds` constructs the full 7-round structure from R1 matches.

**Navigation (lines ~1160–1272).** Owns slam dropdown, segmented M/W control, `slamKey()`, `slamLabel()`, `drawsForSlam()`, `uniqueSlams()`, `activeSlamKey()`, `renderHeader()`, `renderSlamDropdown()`, `switchTab()`, `removeDrawTab()`. `switchTab()` resets `statsRoundFilter` to null. Draws are grouped by `slamKey = slam + '_' + year`.

**Search (lines ~1274–1391).** Owns `allMatchesForSearch()`, `runSearch()`, `closeSearch()`, and keyboard nav (↑/↓/Enter/Escape, ⌘F). Searches only within the active slam+year. Scrolls to and highlights the matched card after switching tabs.

**Chalk scoring (lines ~1393–1422).** Owns `calcChalkScore()`. Computes a "seed-order" reference bracket score for comparison; shown as a diff in the Score stat pill.

**Bracket render (lines ~1424–1542).** Owns `renderBracket()`, geometry constants (`CW=180`, `CH=62`, `GAP=20`, `COL=200`), SVG connector drawing, round-labels-bar population, section labels, and quarter-separator lines. Clears and rebuilds `#bracket-body` from scratch on every call. Must not persist any DOM references across renders.

**Card render (lines ~1544–1674).** Owns `placeCard()` and inner `makeRow()`. Constructs individual match cards with player rows, pick state CSS classes, the score input, ✓/✗ buttons, high-confidence star, and player edit button. All event listeners are attached here at render time (no delegation). Must not read or write `state` directly — receives `d`, `m`, `ri`, `mi` as arguments.

**Edit screen (lines ~1676–1711).** Owns `renderEditScreen()`, `getEditedMatches()`, and the add-match and confirm-draw handlers. Stores the raw parsed match array in `dataset.raw` on the container. `confirm-draw-btn` writes the new draw to `state.draws` (replacing if existing) and navigates to the bracket.

**Upload screen / screen transitions (lines ~1713–1798).** Owns `showUploadScreen()`, `renderDrawPills()`, `showBracketScreen()`, `checkReady()`, all upload-form event handlers, and the load-JSON handler. `mergeDraws()` is called on load.

**Print (lines ~1800–1964).** Owns `buildPrintHTML()`. Generates a fully self-contained HTML document targeting A3 portrait (277 mm usable, 420 mm tall, 10/10/8 mm padding). Two pages: top half (R1 matches 0–31) and bottom half (32–63). Six visible columns R1–SF, each with inline bracket arms. Champion shown in top-half header. All styling is inline — this section must remain self-contained.

**Init (lines ~1966–1983).** `DOMContentLoaded` wires scroll sync between `#bracket-body` and `#round-labels-inner`. `window.load` restores from localStorage; handles both legacy single-draw format (`raw.rounds`) and current multi-draw format (`raw.draws`).

---

## 4. Data Model

All state lives in one object persisted to localStorage under `slam_bracket`.

```
state = {
  draws:     Draw[],
  activeTab: number        // index into draws[]
}

Draw = {
  slam:   'AO' | 'RG' | 'WIM' | 'USO',
  draw:   'MS' | 'WS',
  year:   string,          // e.g. "2026"
  locked: boolean,
  rounds: Round[]          // indices 0–6: R1, R2, R3, R4, QF, SF, F
}

Round = {
  label:   'R1'|'R2'|'R3'|'R4'|'QF'|'SF'|'F',
  matches: Match[]
}

Match = {
  id:              string,           // e.g. "r1_0", "R2_3"
  p1:              Player,
  p2:              Player,
  pick:            string | null,    // current active pick (player name)
  originalPick:    string | null,    // snapshotted at lock; never mutated after lock
  winner:          string | null,    // confirmed winner name
  result:          'correct' | 'wrong' | null,
  score:           string,           // freeform score text
  editedAfterLock: boolean,          // true = withdrawal; re-pick allowed post-lock
  highConfidence:  boolean           // star flag on the active pick row
}

Player = {
  name: string,
  seed: string,    // '1'–'32', 'Q', 'WC', 'LL', 'PR', or ''
  elim?: boolean   // transient; set by markLoserForward — visual only, never persisted intent
}
```

**Bracket topology.** R1 has 64 matches. Each round `ri` has `ceil(prev.length / 2)` matches. Match `mi` in round `ri` is fed by matches `mi*2` and `mi*2+1` of round `ri-1`. `getNextSlot(d, ri, mi)` returns `{ nri: ri+1, nmi: floor(mi/2), side: mi%2===0 ? 'p1' : 'p2' }`.

**Slam key.** `slamKey(d) = d.slam + '_' + d.year`. This is the grouping key for multi-draw operations (save scoping, dropdown grouping, search scope).

**Scoring constants.** `ROUND_CONFIG[ri].base` = `round(1.78^ri)` → `[1, 2, 3, 6, 10, 18, 32]` for ri 0–6. Upset bonus = `numericSeed(winner) - numericSeed(loser)`, floored at 0. Unseeded/Q/WC/LL/PR all map to seed 33. Unseeded vs. unseeded special-cases to 0.5 flat. Only correct original picks score; backup picks track accuracy only.

**JSON save format.** The downloaded file has shape `{ draws: Draw[], activeTab: 0, _slamKey: string }`. Legacy format (single draw at root) is handled in `migrateState()`.

---

## 5. Rules & Conventions

**Single-file architecture.** All code stays in `index.html`. Do not create separate `.js`, `.css`, or component files unless Ben explicitly asks.

**Never rewrite whole files.** Make surgical edits. If a section needs large changes, edit it in discrete, reviewable chunks.

**State mutation protocol.** After any state change: call `saveToStorage()`, then `renderStats()`, then `renderBracket()` — in that order, always. Missing any step leaves the UI stale or data unsaved.

**`originalPick` is sacred post-lock.** Once locked, `originalPick` on a match must never be changed except by `withdrawalClearForward()` (which nulls it as part of a withdrawal). It is the source of truth for scoring and draw accuracy.

**Pick semantics change at lock boundary.**
- Pre-lock: picks cascade forward via `placePickAllRounds()`. Changing a pick clears the old one forward via `clearPickForward()`.
- Post-lock (normal): picks are backup — no cascade, purple styling. Only available when the match has no result yet.
- Post-lock (`editedAfterLock: true`): withdrawal repick — cascades like pre-lock, re-snapshots `originalPick`, and clears `editedAfterLock`.

**`$()` shorthand.** `function $(id){return document.getElementById(id)}`. Never redefine or shadow it. Never use `querySelector` with IDs that already have a `$()` alias.

**Stable DOM IDs.** The following IDs are referenced in JS and must not be renamed or removed: `screen-upload`, `screen-edit`, `screen-bracket`, `bracket-body`, `stats-strip`, `stat-desc-bar`, `stat-desc-label`, `stat-desc-text`, `hdr-row1`, `hdr-row2`, `round-labels-bar`, `round-labels-inner`, `slam-dropdown-btn`, `slam-dropdown-menu`, `seg-control`, `lock-switch`, `lock-track`, `lock-label`, `search-input`, `search-results`, `search-wrap`, `search-clear`, `save-btn`, `storage-warn`, `print-btn`, `add-draw-btn`, `parse-btn`, `parse-status`, `file-loaded`, `pdf-input`, `slam-select`, `draw-select`, `year-input`, `go-bracket-btn`, `load-saved-btn`, `load-json-input`, `edit-player-modal`, `epm-title`, `epm-subtitle`, `epm-seed`, `epm-name`, `epm-confirm`, `epm-cancel`, `epm-btn-area`, `epm-inputs`, `existing-draws`, `draw-pills`, `edit-title-label`, `edit-matches-container`.

**CSS variables for colors.** All colors in component styles must use `var(--token)`. The slam theme tokens (`--accent`, `--accent-dim`, etc.) are overridden per-slam on `body.theme-AO` etc. Exceptions: print HTML (inline styles, hardcoded hex) and SVG connector lines (hardcoded `#c8c4bb`).

**Typography contract.** Playfair Display for player names, headings, and the champion box. DM Mono for seeds, labels, stats, scores, and monospaced indicators. DM Sans for body text, buttons, and UI chrome. Never substitute fonts without discussion.

**`migrateState()` is the migration gate.** Any new field added to Match or Draw must have a defensive default in `migrateState()`. Do not add inline defaults scattered through rendering code.

**`renderBracket()` is destructive.** It clears `#bracket-body` and rebuilds from scratch. Never cache DOM references to bracket cards across renders. Event listeners are re-attached on every render inside `placeCard()`.

**Print is standalone.** `buildPrintHTML()` produces a self-contained document. It must not reference any external state or DOM other than the `Draw` argument passed to it. Keep all its styles inline.

**Responsive breakpoints are ordered.** The cascade is: ≤600px (hide lock label), ≤720px (shorten seg/buttons), ≤580px (shorten + Draw button). Don't add breakpoints that conflict with this cascade without reviewing all three stages.

---

## 6. Feature Status

**Complete and working:**
- PDF upload + drag-drop, TNNS Live text extraction (pdf.js)
- Regex parser for 128-player draws; handles seeded, Q, WC, LL, PR entries
- Edit screen for reviewing/correcting R1 before confirming
- Full bracket render: absolute-positioned cards, SVG connector lines, quarter separators, section labels
- Pre-lock pick selection with full forward cascade
- Lock with `originalPick` snapshotting; unlock with confirmation dialog (clears results, restores picks)
- Post-lock backup picks (purple, no cascade)
- Winner confirmation (✓/✗ buttons) and undo; winner picker modal for unpicked matches
- Player name/seed edit (pre- and post-lock); withdrawal handling post-lock
- `markLoserForward` / `unmarkLoserForward` for immediate downstream visual feedback
- Scoring: base points + upset bonus; chalk reference score; "vs chalk" diff
- Stats bar: Score, Draw Accuracy, Match Accuracy, Draw Health pills with hover descriptions
- Round filter on stats bar (snapshot mode for completed rounds)
- Multiple draws per session: slam dropdown + segmented M/W control
- Player search across active slam draws; keyboard navigation (↑↓ Enter Esc); scroll-to-card with pulse
- ⌘F / Ctrl+F shortcut to focus search
- Save (scoped JSON download), load (merge into state), localStorage autosave with 4 MB warning
- Print: A3 portrait, 2 pages (top/bottom half), 6 columns R1–SF, inline pick-state indicators
- PWA: manifest + icons, standalone display
- Per-slam color themes (AO blue, RG red/terracotta, WIM green, USO blue)
- Responsive header: 3-stage cascade at 600/720/580 px
- High-confidence star flag on active pre-result picks (both halves confirmed)
- `migrateState()` handles legacy field names (backup, backupSide, p1Backup, p2Backup, watch)

**Not yet built / known gaps:**
- Multiplayer / shared draw pools (planned: Supabase backend, admin-controlled result entry)
- No server-side component of any kind
- No automated tests
- PDF parser is regex-based and may fail on non-standard TNNS Live formats or unusual name characters
- Player names are stored as `"Lastname F."` (first initial only) — should be full names; requires updating the `parseTnnsText` regex capture in the parser and anywhere names are propagated forward
- `statsRoundFilter` resets to null on tab switch (intentional, acceptable)
- No mobile-optimized layout (app is designed for desktop/tablet use)
