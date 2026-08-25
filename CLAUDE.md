# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running & verifying

There is no build, no dependencies, no tests, and no linter — no `package.json` at all. Do not add
tooling unless asked.

- Run it: `open index.html` (macOS), or serve statically: `python3 -m http.server 8000` /
  `npx serve .`, then open `http://localhost:8000`.
- Verification is manual, in a browser: load the page and confirm pieces spawn, move, rotate, soft/
  hard drop, lock, clear lines, and that SCORE/LINES/LEVEL update correctly. There is no automated
  check to run.

## Architecture

The game is one flat, un-modularized script (`game.js`, ~300 lines, `'use strict'`, no classes/
modules/build step) plus `index.html` and `style.css`. The cross-file details that matter when
editing:

- **Single global-state block.** All mutable game state lives in one `let` declaration
  (`game.js:43`): `board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`,
  `lastTime`, `dropAccum`, `dropInterval`, `animId`. Every function reads/mutates these directly.
  `init()` (`game.js:259`) is both the bootstrap and the restart handler (wired to `restartBtn`) —
  any new state must be reset there or it will leak across games.

- **Piece type index is overloaded three ways.** The integer `1–7` is simultaneously the index into
  `PIECES`, the index into `COLORS`, and the value stored in an occupied board cell (`0` = empty).
  Each shape matrix in `PIECES` (`game.js:18`) is filled with its own type number (e.g. the T piece
  is made of `3`s). A new piece type must stay in sync across all three, and `randomPiece()`
  (`game.js:49`) hardcodes the `1–7` range via `Math.random() * 7`.

- **Canvas size is duplicated across files.** `COLS`/`ROWS`/`BLOCK` (`game.js:3-5`, 10/20/30) must
  match the hardcoded `width="300" height="600"` on `<canvas id="board">` in `index.html:12`.
  Similarly, `drawNext()` (`game.js:210`) assumes a 4×4 cell grid at 30px against
  `<canvas id="next-canvas">`'s 120×120 (`index.html:30`). Changing board/block geometry means
  updating both files.

- **DOM lookups are unguarded.** `game.js:31-41` grabs every element by `id` at load time with no
  null checks, so renaming/removing an id in `index.html` breaks the script at startup. The
  `<script>` tag sits at the end of `<body>` (`index.html:55`), which is what makes those lookups
  safe to run at load time.

- **Game loop & pause.** `loop()` (`game.js:243`) is a `requestAnimationFrame` loop that accumulates
  `dt` into `dropAccum` and advances the piece once it exceeds `dropInterval`. Both pause and game
  over call `cancelAnimationFrame(animId)`. `togglePause()` (`game.js:229`) resets `lastTime` before
  resuming — skipping that would make the first `dt` after unpause span the whole paused duration.

- **Rotation.** No SRS/rotation-state tracking. `rotateCW()` (`game.js:68`) transposes + reverses
  the shape matrix fresh each call. `tryRotate()` (`game.js:77`) does naive wall kicks by trying
  column offsets `[0, -1, 1, -2, 2]` in order and keeping the first that doesn't collide.

- **Difficulty curve is derived, not stored.** Computed in `clearLines()` (`game.js:96`) whenever
  lines clear: `level = floor(lines / 10) + 1`, `dropInterval = max(100, 1000 - (level - 1) * 90)`.

- **Rendering.** `draw()` (`game.js:188`) redraws everything every frame, in order: grid → locked
  board → ghost piece (via `drawBlock(..., alpha=0.2)`, position from `ghostY()`) → active piece.
  `ghostY()` is called from both `draw()` and `hardDrop()`. `drawNext()` only runs on `spawn()`, not
  every frame.

## Conventions

- Vanilla ES6+ (`const`/`let`, arrow functions, spread, `Array.from`, template literals). No
  framework, no transpilation — match this plain-script style for any new code.
- User-facing text is Spanish (`index.html` has `lang="es"`; overlay strings are `PAUSA` /
  `Puntuación:` / `Reiniciar`). Keep new UI copy in Spanish. Code identifiers/comments in `game.js`
  are English — preserve that split.
