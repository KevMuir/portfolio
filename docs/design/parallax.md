# Hero Parallax — Design Notes

## What it does
A `<canvas>` element sits absolutely behind the hero section. It renders three independent layers of floating nodes (dots) connected by faint lines, creating a shifting network graph effect. Each layer drifts at a different speed relative to the mouse cursor position, producing depth without heavy visual noise.

## Why it's built this way

**Canvas over CSS transforms**: The node-to-node connecting lines require knowing every other node's position at every frame. CSS transforms on individual elements can't express inter-element relationships, so canvas is the right tool. A CSS-only approach (e.g., animated pseudo-elements) couldn't produce the organic network graph look.

**Three layers at different depths**: A single layer moving uniformly looks like a texture sliding around. Three layers at `0.012x`, `0.028x`, and `0.048x` mouse delta create genuine parallax — the eye reads depth because elements at different visual distances move at different rates. Keeping the multipliers small (max ~0.05) means the effect is noticeable but never distracting.

**Lerped offsets (not direct assignment)**: Mouse position sets a *target* offset; the actual draw offset moves 7% toward the target each frame (`offset += (target - offset) * 0.07`). This easing makes the response feel physical rather than mechanical. 0.07 was chosen empirically — lower feels sluggish, higher loses the smooth-follow quality.

**Sinusoidal autonomous drift**: Each node has unique `phaseX`, `phaseY`, `ampX`, `ampY`, and `speed` values, so nodes drift independently with `sin(t * speed + phase) * amp`. Without this, the canvas is static when the mouse isn't moving, which kills the "alive" feeling. With it, the patterns slowly reconfigure over time even without interaction.

**Mouse listener on `window`, not `canvas`**: The canvas fills only the hero section. If a user moves the mouse above the fold before the hero is scrolled into view, or approaches from outside the hero boundary, attaching to `window` ensures the parallax is always responsive.

## Key patterns

### Layer data structure
```js
const LAYERS = [
  { depth: 0.012, nodes: [], color: 'rgba(6,182,212,',  count: 28, sizeRange: [1.5, 3.0] },
  { depth: 0.028, nodes: [], color: 'rgba(139,92,246,', count: 18, sizeRange: [1.0, 2.0] },
  { depth: 0.048, nodes: [], color: 'rgba(6,182,212,',  count: 12, sizeRange: [0.8, 1.6] },
];
```
`depth` is the mouse-delta multiplier. Farther layers (lower depth) have more nodes and larger dots. Closer layers (higher depth) have fewer, smaller dots — matching how distant objects appear denser.

### Line alpha falloff
```js
const alpha = (1 - dist / LINK_DIST) * 0.18;
ctx.strokeStyle = layer.color + alpha + ')';
```
Lines are invisible at `LINK_DIST` (200px) and peak at `0.18` alpha when nodes are coincident. This means lines only appear between nearby nodes and fade gracefully — no hard-edge appearance/disappearance.

### Lerped offset easing
```js
offsets[i].x += (targetOffsets[i].x - offsets[i].x) * 0.07;
```
Runs every `requestAnimationFrame`. The 0.07 factor means the offset closes ~50% of the remaining gap every ~10 frames (at 60fps, ~160ms to 95% of target). Feels natural, not laggy.

## Gotchas

- **`canvas.width/height` vs CSS size**: Setting `canvas.width = canvas.offsetWidth` (not a fixed value) is critical on resize. If you set the CSS size via Tailwind but don't update the canvas resolution to match, drawing coordinates are wrong and the canvas looks blurry/stretched.
- **`buildNodes()` on resize**: Nodes store absolute x/y positions based on the canvas dimensions at creation time. If you resize without rebuilding, all nodes cluster in the old canvas bounds. The resize handler calls `buildNodes()` to redistribute.
- **Node positions wrap visually but aren't clamped**: Nodes near edges can drift off-canvas during mouse parallax or sinusoidal drift. This is intentional — it avoids "bouncing" nodes at the boundary which would look jittery. Slightly off-canvas nodes just don't render, which is a clean falloff.
- **Touch support**: `touchmove` listener uses `e.touches[0]` and passes `{ passive: true }` to avoid scroll blocking. Remove the passive flag only if you need `preventDefault()` on touch.
- **Performance**: `O(n²)` line drawing per layer. At the current node counts (28/18/12) this is ~1800 distance checks per frame — trivial. If you increase `count` significantly (>60/layer), profile first.

## Related
- Hero section markup: `index.html` — `<section id="home">`
- Canvas element: `<canvas id="parallax-canvas">`
- Script: inline `<script>` IIFE at bottom of `index.html`
