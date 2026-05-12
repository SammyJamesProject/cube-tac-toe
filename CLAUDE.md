# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file browser game (`index.html`) — no build step, no dependencies to install. Open directly in any modern browser. Three.js r158 is loaded via ES module importmap from unpkg CDN.

## Architecture

Everything lives in `index.html`: HTML structure, CSS, and a single `<script type="module">`.

### Face / sticker model

Six faces are indexed `F=0, B=1, L=2, R=3, U=4, D=5` (matching standard Rubik's notation viewed from outside). The flat `stickers[54]` array is the single source of truth for game logic; index = `faceIdx*9 + row*3 + col`. Each entry holds `.mark` (`null | 'X' | 'O'`) and `.mesh` (the Three.js plane that renders it).

`faceRC(fi, gx, gy, gz)` converts a cubie's integer grid coordinates (0–2) to `[row, col]` on its exposed face — this is the mapping that connects the 3D cube to the logical sticker array.

### 3D scene

26 `THREE.Group` cubies (all positions except the invisible centre) live inside `cubieGroup`. Each cubie stores its current grid position in `userData.gx/gy/gz`. Sticker planes are children of their cubie, so they travel with it automatically during Three.js rotations.

### Rotation pipeline

1. `startRotation(name, cw, onDone)` — collects the 9 cubies on the face using the grid-axis filter, reparents them onto a temporary `pivot` group via `pivot.attach()` (preserves world transform), then stores animation state.
2. The render loop tweens `pivot.quaternion` over 320 ms with an ease-in-out quad.
3. `finishAnim()` — reparents cubies back to `cubieGroup` via `cubieGroup.attach()`, snaps positions to integers to prevent float drift, calls `updateCoord()` to update `userData.gx/gy/gz`, then calls `rotateFace()` + `cycleStrips()` to permute the logical sticker state.

`rotateFace` and `cycleStrips` permute both `.mark` and `.mesh` together and update `mesh.userData.si` so the raycast→sticker lookup stays correct.

### Move definitions

`MOVES` stores the 6 base moves. Each entry has `fi` (face index) and `s` (4 strips of 3 sticker indices each). CW cycle: `s[0]→s[1]→s[2]→s[3]→s[0]`. Primed (CCW) moves reuse the same definition with the direction flipped. `CW_PERM` / `CCW_PERM` handle the rotation of the face's own 9 stickers.

Strip helpers: `r(f,i)` = row i of face f, `rr` = reversed row, `c` = column, `cr` = reversed column.

### Layer preview

When a rotation button is hovered, `showPreview(moveName)` makes a semi-transparent highlight plane visible on the relevant face and shows one of the 12 pre-built `arrowGroups` (cyan = CW, orange = CCW). `clearPreview()` hides both. Highlights and arrows are built once at startup and toggled with `.visible` / `.material.opacity`.

### AI opponent

The AI runs in two `setTimeout`-chained phases when `aiEnabled && player === 'O'`:

1. **`aiDoPlace`** — picks a sticker via `aiChooseSticker()` (win → block → centre → random) and calls `goRotate()`.
2. **`aiDoRotate`** — picks a rotation via `aiChooseRotation()` using `simulateRotation()` (immediate win → safe + best threat score → damage control) and calls `startRotation()`.

`simulateRotation(marks, moveName)` operates on a plain JS array copy — no Three.js involvement — making it fast enough to evaluate all 12 moves synchronously.

### Game state machine

```
STATE_PLACE  → click sticker (or AI auto-places) → STATE_ROTATE
STATE_ROTATE → click button (or AI auto-rotates) → check win → STATE_PLACE | STATE_OVER
STATE_OVER   → Play Again → full rebuild via buildCube() → STATE_PLACE
```

`goOver()` is only called from `endTurn()`, which is only called after a rotation completes — wins are never checked at placement time.
