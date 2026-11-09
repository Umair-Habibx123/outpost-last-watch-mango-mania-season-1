# Character atlas pipeline

How the sprite atlases in `public/assets/art/characters/` are made, and the two
rules that break them if you get them wrong.

## Why a tool exists

The renderer slices atlases by pure arithmetic — `naturalWidth / cols` — so a
sheet has to be exactly right. Generated sheets never are: rows wrap, cells
drift, the requested facing order is ignored, and the background comes back flat
white or grey instead of transparent. Prompting harder does not fix it.

What generation IS reliably good at is drawing several **consistent poses of one
character** in a single image. So `tools/atlas-build.mjs` throws away the source
layout entirely: it finds each sprite, cuts it out, and rebuilds the grid.

```bash
node tools/atlas-build.mjs     # then open http://localhost:3998
```

Put source sheets in `tools/src/`. In the page console:

```js
await loadSheet('/sheet-warden.png', { tol: 66, minArea: 4000 });
showDetection();                       // numbered boxes — read the indices off this
buildAtlas({ cols:6, rows:8, cellW:256, cellH:288, pad:0.04,
             refH:504, refW:492, cells:[ {col:0,row:0,sprite:3,flip:true}, ... ] });
await save('warden-directional-atlas.webp', 'image/webp', 0.95);
```

## Rule 1 — the foot anchor is 78%, and it eats your cell

Every consumer draws a cell as:

```js
drawImage(..., x - w/2, y - h*0.78, w, h)
```

so the world position sits **78% down the cell**. Two consequences:

1. Sprite feet must land on 78% of cell height, or characters hover or sink.
2. **Only 78% of the cell is usable for the body.** Scaling a sprite to the full
   cell height pushes its head above the cell top, where the per-cell clip
   decapitates it. This is exactly the bug that shipped in the first warden
   atlas — heads sliced flat.

`buildAtlas` now derives its scale from `cellH * FOOT_ANCHOR - cellH * pad`.
The knock-on effect: figures occupy ~74% of the cell rather than filling it, so
**draw widths in the consumers had to grow** to keep on-screen size
(backup guard 59 → 82, war beast 75 → 105 / 84 → 118 in `arsenal.js`). If you
rebuild an atlas and characters suddenly look small, this is why.

## Rule 2 — sprite indices are not stable

Detection indices depend on the keying tolerance. Raising `tol` from 44 to 66 on
the warden sheet renumbered everything and silently produced a wrong atlas (row 0
aiming left, row 4 firing in all six frames). It also **separated** two wardens
that had been merged by their shared muzzle flash, making a hand-crop redundant.

Always re-run `showDetection()` after changing `tol`, and read the indices off
the preview rather than reusing old ones.

## Facing rows

`atlasDirectionRow(angle) = round(angle / (PI/4)) mod 8`, canvas angles, +y down:

| row | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| facing | E | SE | **S (front)** | SW | W | NW | **N (back)** | NE |

Generation gives 3-4 genuinely distinct angles (side, front, back, three-quarter).
Fill the rest with `flip:true` to mirror. That is standard 2D practice and reads
fine in motion, but it is a compromise — true 8-angle art needs identity-locked
image-to-image, which the free plan cannot reach.

## The tool's other pieces

- **Background keying** floods inward from the border only, so light areas
  *inside* a sprite (pale fur, bone horns) survive. A second soft pass ramps
  alpha on near-background pixels — without it a fire breath's glow stays opaque
  and cuts out as a pale rectangle.
- **`crop(x,y,w,h,{fadeRight})`** carves regions by hand; detection merges
  anything physically touching. `fadeRight` ramps alpha out so a flame cut
  mid-stream dissolves instead of ending in a hard edge.
- **`refH` / `refW`** hold one scale across every cell. Per-cell fitting makes a
  character pulse in animation, because shorter poses get scaled up more.
- **Per-cell clipping** stops a deliberately-overflowing frame bleeding into its
  neighbour.

## Current atlas specs

| file | grid | cell | frames |
|---|---|---|---|
| `warden-directional-atlas.webp` | 6×8 | 256×288 | idle, aim, fire, fire, aim, idle |
| `backup-guard-atlas.webp` | 5×8 | 224×256 | consumer uses 1 idle, 2-3 fire, 4 recover |
| `war-beast-atlas.webp` | 6×8 | 256×192 | 0-3 run cycle, 4-5 pounce |
| `dragon-atlas.webp` | 8×1 | 420×260 | 0-3 fly, 4 dive, 5-7 breath (flame grows) |

Legacy art is preserved in `public/assets-old/`.
