# Word Wall Breaker — CLAUDE.md

This file is named **`CLAUDE.md`** so that Claude Code auto-loads it whenever a session opens this folder. Do not rename it — renaming breaks auto-loading.

## Files & location

Everything for this game lives in one folder:

```
C:\Users\User\.claude\Code Sessions\Word Wall Breaker\
├── index.html   ← the entire game (HTML + CSS + JS, ~1090 lines)
└── CLAUDE.md    ← this file — auto-loaded by Claude Code (do not rename)
```

There are no other files, assets, or dependencies — the game is fully self-contained in `index.html`.

## What this is

A single-file browser game: **Word Wall Breaker**, a Breakout/Arkanoid-style brain-training game built around words. The entire app — HTML, CSS, and JS — lives in [index.html](index.html). There is no build step, no dependencies, no package manager, and no test suite.

## Running & verifying

- **Run it:** open `index.html` directly in a browser (`file://`). On Windows: `Start-Process "C:\Users\User\.claude\Code Sessions\Word Wall Breaker\index.html"`.
- **No local web runtime:** this machine has no working Node/Python/npx, so a dev server (and the Preview MCP that needs one) cannot be started for this static file. Verify changes by careful code reading plus opening the file in a browser.
- **Debugging "nothing happens":** there are no tests, so the browser DevTools console (F12) is the primary diagnostic. A `$("id")` lookup against a non-existent element returns `null`; the subsequent property access throws a `TypeError` that **silently aborts the click/key handler mid-way** (and can leave `game.state` stranded, e.g. in `PAUSED`). This class of bug is invisible except in the console.

## Architecture (all in `index.html`)

The `<script>` is organized into labeled banner-comment sections (Word data → Persistence → Canvas → Audio → Helpers → Game state → Level build → Trays → Input → Game logic → Update/Physics → Render → HUD → Flow → UI wiring → Main loop). Key cross-cutting concepts:

- **Single `game` object + string state machine.** `game.state` is one of `MENU / MODES / SETTINGS / STATS / PLAYING / PAUSED / MEMORIZE / LEVEL / OVER` plus reveal/guess overlays. `update(dt)` early-returns unless `state==="PLAYING"`, so pausing/menus simply stop physics. `showOverlay(key)` / `hideOverlays()` toggle the matching DOM overlay; every overlay must be registered in the `overlays` map.

- **Canvas + DOM split.** The playfield (bricks, ball, paddle, particles) is drawn on `<canvas>` in the render loop. All menus, the HUD, the answer trays, and the word-reveal/guess dialogs are DOM elements layered on top via `position:absolute` + `z-index`. **Gotcha:** `#hud` has `pointer-events:none`; any interactive HUD control needs an explicit `pointer-events:auto` CSS rule (inline styles on children are unreliable here).

- **Four modes, one engine.** `game.mode` ∈ `unscramble / memory / category / time`. `buildLevel()` branches on mode but reuses the same ball/paddle/collision core. Mode differences live in: brick rendering (`drawBricks`), the trays, the hit handlers, and the win check.

- **The unscramble letter model (the core mechanic).** Each word is `{answer, order, live, solved, accent}`. `order` maps each visual column → an index into `answer` (the scramble). `live` is the count of letters already cleared, so the *only* legal next target is the brick whose `ansIdx === word.live`. Hitting it = `correctHit` (advance `live`, score, combo); hitting any other letter brick = `wrongHit` (combo reset, no break). A word is solved when `live === answer.length`.

- **`layout()` rebuilds `game.bricks` from scratch** on every level build, resize, and after each break. For letter bricks it **skips any brick with `ansIdx < word.live`** — that single condition is what keeps already-broken letters gone. If you change how letters are cleared, preserve this invariant or broken bricks will reappear. Category bricks are skipped when their backing `ref.broken` is true.

- **Collision** (`handleBrickCollision`) picks the single closest overlapping brick per sub-step (the update loop sub-steps the ball to avoid tunneling at high speed), reflects off it, then dispatches to the mode-specific hit handler.

- **Word reveal** (`showWordReveal(nextFn, isWin)`) runs only in unscramble mode and wraps both `levelComplete()` and `gameOver()` — it animates the answers in, then calls `nextFn` to show the actual result screen (auto-advances on a timer, or via the Continue button).

- **Persistence:** `cfg` (settings) and `stats` (lifetime totals) are plain objects mirrored to `localStorage` under `wwb_cfg_v1` / `wwb_stats_v1` via `saveCfg()` / `saveStats()`. Stats are committed in `gameOver()`.

- **Colors:** theming uses CSS variables; `css(var)` reads a resolved value and `refreshColors()` caches them in `C` (called on load and theme change). Canvas `fillStyle` cannot resolve `var(--x)` strings — pass a resolved color (e.g. `C.good`, a hex, or `hexA(hex, alpha)`), not a CSS-variable string.

- **Audio** is generated with the Web Audio API (`beep()` + named helpers); gated by `cfg.sound`. No external assets.

## Conventions

- Keep everything self-contained in `index.html` — no external files, CDNs, or build tooling.
- Match the existing terse, single-letter-helper style (`$`, `rr`, `rnd`, `pick`, `clamp`, `hexA`).
- Mode-specific behavior belongs in the relevant branch of `buildLevel`/`drawBricks`/hit-handlers/win-check, not in new top-level globals.
