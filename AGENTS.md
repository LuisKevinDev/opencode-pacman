# AGENTS.md

## What this repo is

Pac-Man clone built with **vanilla JS + HTML + CSS** to practice **spec-driven development**.
There is no toolchain: no `package.json`, no dependencies, no bundler, no CI, no linter, no formatter, no test runner. Don't invent or add one unless asked.

## Running and verifying

- Open `src/index.html` directly in a browser (`Start-Process src\index.html`). It works over `file://` because every script is a classic `<script>` — no modules, no `fetch`. Do **not** convert files to ES modules; that breaks `file://` (CORS) and the global-based wiring below.
- Verification is manual: there is nothing to run. Check console for errors and play a round.
- Optional static server: `npx serve src`. Node v22 is available if you want scratch scripts.

### Poor man's headless harness

Because each file ends with `window.X = X` assignments, the logic-only files load in Node in dependency order:

```powershell
node -e "globalThis.window=globalThis; require('./src/js/maze.js'); require('./src/js/game.js'); const g=window.createGame(); g.state='playing'; let n=0; while(g.state==='playing' && n<200000){ window.update(g); n++; } console.log(n, g.state, g.score, g.dotsRemaining, g.lives);"
```

Gotchas: `update()` does **not** check `game.state` itself (main.js gates it), so set `g.state='playing'` first. `render.js` is excluded — it needs a real canvas context.

## How the code is wired

- Files communicate via **globals assigned to `window`** at the bottom of each file, not imports. `src/index.html:19-22` loads scripts in a strictly required order: `maze.js` → `game.js` → `render.js` → `main.js` (render.js reads `DIRS` from game.js; main.js starts the loop at load time).
- Adding a new JS file requires adding its `<script>` tag in `index.html` in the right position. Nothing is auto-discovered.
- Loop: `main.js` `loop()` → `update(game)` (rules) → `draw(ctx, game, frame)` (rendering). State machine is `start` → `playing` → `won` / `lost`.

## Grid and coordinate conventions

- Maze lives in `src/js/maze.js` as 31 strings of 28 chars, parsed into numbers: `1` wall, `2` dot, `3` pen door (ghosts pass, Pac-Man doesn't), `0` walkable empty. Indexed `grid[y][x]`, origin top-left, x ∈ [0,27], y ∈ [0,30].
- `MAZE` is **pristine and must never be mutated**. Each new game copies it into `game.grid` (`createGame`), which is what dots are eaten from and what render reads.
- Sizes are duplicated in three places and must stay in sync: `TILE = 20` in `render.js`, `width=560 height=620` on the canvas in `index.html`, and `#game-wrap` in `style.css` (28×20 = 560, 31×20 = 620). There is no responsive scaling.
- Grid currently: 31 rows × 28 cols, 280 tiles with dot; Pac-Man's start cell is cleared, so `dotsRemaining` starts at 279.
- Row 14 is the tunnel row (`TUNNEL_ROW`): exiting either side wraps to the other (see `canMove` / `wrapTunnel`). Rows 12-17 contain the ghost pen.

## Gameplay rules that are easy to break

- Speeds are **cells per frame**, not pixels or ms: `PACMAN_SPEED = 0.125` (1/8 cell), `GHOST_SPEED = 0.1` (1/10). Turning and dot-eating only happen when the actor is cell-aligned (`aligned()` in game.js). Changing a speed can skip the aligned window and silently break turning — during edits prefer multiples that keep alignment exact.
- The loop is `requestAnimationFrame`-based and frame-, not time-based: on a 144 Hz display the game runs ~2.4× faster. Don't assume 60 fps constants.
- `update()` always runs full steps; the loop only skips it when `state !== 'playing'`.
- Two ghosts only (`GHOST_STARTS`, kinds `hunter` = greedy Manhattan chase, `random`). The hunter has **no pathfinding** — it picks the locally closest legal direction, so it gets stuck against walls. No frightened mode, power pellets, scatter, or level progression exist yet.

## Known rough edges

- The initial overlay markup in `index.html` (title + `<p>` instructions + `#action-btn`) is **replaced** by `showOverlay()` in main.js on the first win/lose, and only the rebuilt `<h1>` + button come back. If you add overlay UI, add it in both places.
- Losing a life calls `resetPositions()` but keeps the eaten dots and score; full reset of score/lives/dots only happens through `startGame()`.

## Style conventions

- JavaScript: 2-space indent and **spaces inside parentheses** — `function f( a, b )` and `if ( x )`. Match it in new code; there is no Prettier config to enforce it.
- Spanish is the repo's language: code comments, commit-facing docs (README) and all UI strings (`SCORE`, `VIDAS`, `GANASTE`, `PERDISTE`) are Spanish. Replies to the user go in whatever language they write in. Specs may be written in English or Spanish — pick one per repo and stay consistent.

## Spec-driven workflow (the point of this repo)

- Skills live in `.agents/skills/` (`spec`, `spec-impl`) and are vendored copies from `klerith/fernando-skills`, tracked with hashes in `skills-lock.json`. Don't hand-edit them; local edits desync the lock file.
- New features start with `/spec <feature>`, which writes `specs/NN-slug.md` from `.agents/skills/spec/template.md`. `specs/` does not exist yet, so the first spec is `01-…`. Number zero-padded, sequential.
- A saved spec lands in `Draft`. **Only the human flips it to `Approved`/`Aprobado`** — `/spec-impl NN-slug` refuses to run on anything else and will stop with an error. Do not "helpfully" set it to Approved yourself.
- `/spec-impl` then creates and switches to branch `spec-NN-slug` (controlled by `specs/.spec-config.yml`, `AutoCreateBranch`, defaults to true), implements one plan step at a time **pausing for review after each**, and never commits — committing is the user's decision. Implement exactly what the spec says; put disagreements in the spec, not in the code.
- Default branch is `main`. Don't push or create PRs unless asked.
