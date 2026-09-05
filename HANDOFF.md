# HyperSnake — handoff notes

## What this is

A snake-in-the-box playground for hypercubes Q2–Q20, built as a single-file PWA so it runs on a phone. A *snake* is an induced path in the hypercube: no two vertices on it are adjacent unless they are consecutive. The app lets you grow up to 10 snakes at once by tapping vertices, shows a "burn map" of how many snake vertices each vertex touches, and can run an exhaustive search that extends the snakes already on the board.

It grew out of an interactive widget in a chat; this folder is the standalone version. No build step, no dependencies, vanilla JS.

## Files

- `index.html` — the entire app: CSS, markup, logic, and the search worker (as an inline source string turned into a Blob worker).
- `manifest.webmanifest`, `sw.js`, `icon.svg`, `icon-192.png`, `icon-512.png`, `icon-512-maskable.png` — PWA install and offline support. `sw.js` uses a network-first cache named `hypersnake-v4`; bump the name when `index.html` changes.
- `README.md` — user-facing usage and deploy instructions (GitHub Pages → Add to Home Screen).

## Rules as implemented

- `k` snake slots (1–10), default 1. Each snake is an ordered vertex array; index 0 is the tail, last is the head. A single-vertex snake is allowed and has length 0 (length = edges = vertices − 1).
- **Extending:** vertex `v` may be appended to snake `i` at end `e` iff `v` is unoccupied, `v` has exactly one neighbor in snake `i` (which must be `e`), and — when **Snakes may touch** is off — `v` has no neighbors in any other snake. See `canJoin(v,i,o)`.
- **Starting:** an unoccupied vertex with a free slot; when touch is off it must also have zero snake neighbors. Tapping a completely empty board starts snake 1 immediately; otherwise you get a "Start snake N here" button.
- **Touch mode on:** the only cross-snake rule is vertex-disjointness. Edges joining two different snakes are drawn coral and counted as "contacts".
- Turning touch off with contacts present doesn't alter the board; those edges just stop being legal moves.

## Interaction model (tap vertices, not edges)

`tapVertex(v)` in `index.html`:
1. If an end is selected and `v` is a legal move from it → extend, keep the new end selected (chain taps).
2. If `v` is on a snake → select it: ends become an end-selection (legal moves highlighted in blue, listed as chips); interior vertices show "Drop head side (n) / Drop tail side (n) / Delete snake".
3. If `v` is free with exactly one `(snake, end)` that can reach it → extend immediately. Multiple → choose via buttons. None → info panel with neighbor chips (a local view: tapping a chip focuses that neighbor and recenters the board).

The panel with no selection lists every snake's ends with their move counts; tapping one selects it and calls `centerOn(v)` (zoom ≥3 at n≥7). This chip-driven flow is the intended way to play Q7–Q10.

Desktop mouse hover highlights a vertex's neighborhood; touch devices get the same on tap.

## Layout

- `buildLayout()`: for n ≤ 4 with layout `proj`, a parallel projection with axis vectors `AX` chosen by a random search so no two vertices or non-incident edges overlap (the classic tesseract drawing had collisions). Otherwise a Gray-code torus grid: rows = low ⌊n/2⌋ bits, columns = high ⌈n/2⌉ bits, both in Gray order, so every edge is within a row or column. Non-adjacent same-row/column pairs (wraparounds and the extra Gray-code chords once a side has ≥3 bits) are drawn as quadratic arcs bulging up/left. Apex height is interpolated by gap between `LG.lo` (just clears the vertex circles the arc passes over) and `LG.hi` (just short of the next row/column), see `gapRange`/`arcH`; the earlier version scaled height by gap without a ceiling and the widest arcs ran straight through the row above.
- Vertex spacing/radius: 120/17 (n ≤ 4), 72/15 (5–6), 56/13 (≥7). Labels binary for n ≤ 4, decimal otherwise (setting: auto/bin/dec/none).
- `edgesMode` 'all' vs 'snakes' (only snake, contact, and highlight edges). Auto-switches to 'snakes' at n ≥ 7 and is forced there for n ≥ 11.
- Vertex positions are not stored; `P(v)` computes them from the Gray-code row/column (`projPos` holds the n ≤ 4 projection). `vAt(row,col)` is the inverse.
- **Big boards (n ≥ 11, `CULL()`):** `viewWindow()` works out the visible cell range from the screen rect, `renderBoard()` draws only those cells (via `vAt`) and only snake/contact edges whose row/column span crosses the window (`edgeVisible`). If the window holds more than `VCAP` (4096) cells it switches to an overview: grid outline (`.extent`), snake edges with non-scaling strokes, snake vertices only, taps disabled. `applyView()` schedules a `renderBoard()` on the next frame whenever the window changes, so panning re-draws; `render()` (state changes) still does the global `status()` and move count, about 150 ms at Q20. `resetView()` opens big boards at vertex level anchored at the grid's corner. Zoom cap comes from `zoomFor()` so it scales with the board.
- Pan/zoom: `Z, TX, TY` applied as a transform on `<g id="view">` inside a fixed viewBox; pointer events handle drag, pinch, wheel; `nearest()` does hit-testing in board coordinates with a tolerance that scales with zoom (O(1) on the grid by rounding to a cell, a scan for the projection).

## Rendering

`render()` recomputes `ST=status()` and the move count, then `renderBoard()` rebuilds `#view` innerHTML for the current view (whole board up to Q10, visible window above). Groups in draw order: `edead`, `eav`, `econ` (contacts), `ov` (neighborhood/candidate overlay), `esnk` (snake edges), `vg` (vertices). `applyHighlights()` only toggles classes on cached vertex elements and rewrites `#ov`, so hover/selection changes are cheap.

Vertex fill: snake color if occupied, else the red ramp `REDS[cnt-1]` (white for 0). Amber ring = some end can reach it; thick dark ring = snake end; blue rings = focused vertex's neighbors; blue dashed edge + thick blue ring = legal move from the selected end.

## Search (Web Worker)

`workerSrc` string → `new Worker(blobURL)`. Exhaustive DFS that **only extends snakes already on the board** (never starts new ones). It enumerates move sequences whose snake index never decreases: a frame for snake `j` first hands over to snake `j+1` (recording the state), then tries each extension of snake `j` (head end, then tail end) and recurses on `j`. This is complete because any final configuration can be built by finishing snake 0, then snake 1, etc. (the same configuration can still be reached by different head/tail orderings, so it is not duplicate-free). The stack is explicit (`stack` of `{j,k,mv}` frames) because the recursive version overflowed the call stack from Q15 up; node counts and results are identical to the recursive one. The best state is copied lazily: a push always adds one edge, so the best state is the current one until the next pop, and `keep()` snapshots it there or when reporting. Objective = total edges across all snakes. Posts `{nodes, bestT, bestS}` every 2^20 states; Stop terminates the worker and applies the last progress message's best; `worker.onerror` does the same with a toast. Budget options 10M–1B or 0 = until stopped.

Best-known table (`MAXSNAKE`) carries published lower bounds through Q13 and shows `?` beyond.

Measured: Q4 from one edge → 7 in 1,759 states. Q5 from one edge → 13 in 5.9M states (exhaustive, ~0.7 s in headless Chrome). Q6 from one edge → 26 within 10M states but not proven. Q7 from one edge → 43/50 after 7M states. Beyond that it's a lower bound. In 300k states from one edge the greedy descent reaches 1461 at Q13, 5237 at Q15, 36k at Q18 and 135k at Q20 (~2 s).

Test harness: extract `workerSrc` and run it with `new Function('postMessage','data', src + ';onmessage({data});')` (works in a page or in Node).

## Persistence and interchange

`S` (`{n,k,touch,snakes,layout,edgesMode,labels,budget}`) is saved to `localStorage['hypersnake']` on every commit. First load (no saved state) opens the How to play dialog. Settings → Copy/Load position uses `{"n":4,"touch":false,"snakes":[["0000","0001",...],[...]]}` with binary strings (integers also accepted on load).

## What's been tested

Playwright + headless Chromium at 390×844 (2x) and 1200×800, light and dark: start/extend/chain taps, second snake via button, cutting, undo/redo, touch-mode search, position copy/load round-trip, reload persistence, Q6 and Q10 rendering, Q7 search with Stop, manifest/sw served. Not yet tested on a real iOS/Android device.

## Known rough edges

- Code style is dense (written to fit one file); reformatting into readable sections or ES modules would help before larger changes.
- `removeEndSimple` is the only end-removal function (an earlier `removeEnd` was deleted); rename freely.
- Legend and zoom buttons can crowd each other when the legend wraps (n ≥ 7 on narrow screens).
- Dimension/count changes use native `confirm()`. Reset in the bottom bar clears without confirming; it's undoable.
- Search has no pruning and only one objective; from an empty-ish board in Q8+ it mostly burns budget.
- Chord arcs in 'all' edge mode at n ≥ 7 are visually noisy (hence the default).

## Ideas for next steps

1. Smarter search: upper-bound pruning (free vertices reachable from each end), a heuristic/beam mode with random restarts, live "best so far" applied incrementally, and selectable objectives (total, maximin shortest snake, all slots used).
2. Share positions via URL hash instead of copy/paste JSON.
3. Keyboard support on desktop (arrow through candidates, Enter to extend, Backspace to remove).
4. A proper test file (Playwright) mirroring the smoke tests above, plus unit tests for `canJoin`/`status` and the worker.
5. Optional: wrap with Capacitor if a store listing is ever wanted; the PWA route already covers home-screen install and offline.

## Deploy

Push the folder to a GitHub repo root, enable Pages, open the URL on the phone, Add to Home Screen. Remember to bump `CACHE` in `sw.js` on each release.
