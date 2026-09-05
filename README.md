# HyperSnake

A snake-in-the-box playground for the hypercubes Q2 through Q10, built to run on a phone.
It is a single-page web app with no build step and no dependencies.

**Play it:** https://hypersnakes.github.io/

## Files

- `index.html` — the whole app (board, rules, search worker, settings).
- `manifest.webmanifest`, `sw.js`, `icon*.png`, `icon.svg` — what turns it into an installable, offline-capable app.
- `HANDOFF.md` — developer notes: rules as implemented, interaction model, layout, search, known rough edges.
- `README.md` — this file.

## Run it

**Quickest:** open `index.html` in any modern browser. Everything works, including the search, but "Add to Home Screen" only gives you a bookmark and it will not work offline.

**As an app on your phone:** open the hosted URL above (or any HTTPS host serving these files).

- iPhone (Safari): Share → Add to Home Screen.
- Android (Chrome): menu → Install app, or Add to Home screen.

After that it launches full-screen from its own icon and works without a connection.
Your board and settings are saved on the device between sessions.

If you change `index.html` later, bump the `CACHE` name in `sw.js` (for example from `hypersnake-v2` to `hypersnake-v3`) so installed copies pick up the new version.

## Using it

- Tap a vertex to extend a snake to it. If more than one snake end could reach it, you choose which.
- Tap a snake end to select it: its legal moves are listed as chips and highlighted on the board. Tap a chip or a highlighted vertex to keep growing. The board recenters on the new end when you use the chips, which is the practical way to play in Q7 and above.
- Tap a snake vertex to cut the snake there or delete it. Tap an empty vertex to start a new snake when a slot is free. Reset in the bottom bar clears the board (Undo brings it back). The app starts with one snake; the number of snakes is set in Settings.
- Shading is the number of snake vertices a vertex touches; an amber ring marks vertices some end can still reach; coral edges join two different snakes (only possible when "Snakes may touch" is on).
- "Extend snakes" runs an exhaustive depth-first search from the current position in a background thread, growing only the snakes already on the board. It stops at the budget chosen in Settings (or when you press Stop) and applies the best configuration it found. Exhaustive results are realistic up to Q6 and for well-advanced positions in Q7; beyond that the search is a lower bound.
- Copy position / Load position in Settings exports and imports a small JSON string, so you can save interesting configurations or move them between devices.
