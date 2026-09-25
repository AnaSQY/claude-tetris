# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla JS Tetris — HTML5 Canvas + CSS, no frameworks, no build step, no dependencies (no `package.json`).

## Running

Open `index.html` directly, or serve statically:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
npx serve .
```

There is no build, lint, or test command — the project has none configured.

## Architecture

Three files, single global scope, no modules:

- `index.html` — DOM shell: `#board` canvas (300×600, 10×20 grid at `BLOCK=30`px), `#next-canvas` preview, HUD spans (`#score`, `#lines`, `#level`), and `#overlay` for pause/game-over.
- `style.css` — dark/retro arcade theme.
- `game.js` — all logic, procedural style, driven by module-level mutable state (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.) rather than a class or state container.

### Core flow

`init()` builds an empty board, seeds `next` via `randomPiece()`, calls `spawn()`, and starts `loop()` via `requestAnimationFrame`. `loop(ts)` accumulates elapsed time and drops the current piece one row once `dropAccum >= dropInterval`; otherwise it calls `lockPiece()`, which merges the piece into the board, clears completed lines, and spawns the next one. `spawn()` promotes `next` to `current`, generates a new `next`, and calls `endGame()` if the new piece immediately collides.

Key mechanics, all in `game.js`:
- **Pieces**: `PIECES` holds the 7 tetrominoes as square matrices, each cell value doubling as a color index into `COLORS`.
- **Rotation**: `rotateCW()` transposes + reverses rows; `tryRotate()` applies it then attempts wall kicks at offsets `[0, -1, 1, -2, 2]`, discarding the rotation if all collide.
- **Collision**: `collide(shape, ox, oy)` checks board bounds and overlap with locked cells.
- **Line clears**: `clearLines()` scans bottom-up, splices out full rows, unshifts empty ones, and re-checks the same row index after a splice.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop adds 2 pts/row, soft drop 1 pt/row.
- **Leveling/speed**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Ghost piece**: `ghostY()` projects the landing row; drawn via `drawBlock(..., alpha=0.2)`.
- **Highscores/combo**: `combo` counts consecutive `lockPiece()` calls whose `clearLines()` cleared at least one line, resetting to 0 on a clear-less lock; `comboMax` tracks the run's peak. Top-5 scores persist to `localStorage` (`tetris-highscores`, array of `{name, score, lines, combo, date}`); best combo/max lines persist to `tetris-stats` (`{bestCombo, maxLines}`). `endGame()` checks `qualifiesForHighscore()` to show the name-entry form (`#name-entry`); `renderHighscores()` always uses `textContent` (never `innerHTML`) to render names. A `#start-overlay` screen (shown before the first `init()`, via the `#play-btn` handler) and the game-over `#overlay` both render the table and expose a reset button (`resetHighscores()`, gated by `confirm()`). The global `keydown` listener bails out early when `document.activeElement` is an `INPUT`/`TEXTAREA`, so typing a name never triggers game controls.

Input is a single `keydown` listener switching on `e.code` (arrows, `KeyX` to rotate, `Space` for hard drop, `KeyP` to pause).

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS×BLOCK` by `ROWS×BLOCK`).
