# Simple Physics in the Browser, Part 2: A Sheet That Sags, Ripples, and Takes the Mouse

> [!RECALL]
> Before reading on: write out the order of the semi-implicit Euler step for Part 1's spring — which quantity is updated first, which second — and say, in one sentence, why that order stops the wobble from growing.
>
> If you can't reconstruct it, Part 1's "The one-line fix: semi-implicit Euler" has the whole argument: the position update uses the velocity that already includes the push, and the method almost conserves the energy.

A curtain in a window, a tablecloth, a flag in the wind. The simplest computer model of all of them is a grid of dots connected by springs, and it's the family every cloth simulator starts from: "Most models of cloth are based on 'particles' of mass connected in some manner of mesh," Wikipedia notes, and the one we're building is its mass-spring branch — "The second technique treats cloth like a grid work of particles connected to each other by springs."

The surprise of Part 2 is that the surprise is small. One mass becomes 192 and one spring becomes 686, but the integrator is the one Part 1 earned, and the frame clock is the one Part 1 built. The only genuinely new code is the part that lets you grab a mass with the mouse and flick it.

## What you'll build

A single file, `sheet.html`:

- A 16 × 12 grid of masses — 192 of them — pinned along its top edge, hanging from a ceiling.
- 686 springs: horizontal and vertical (structural) plus diagonal (shear).
- The sheet sags under its own weight, ripples when you disturb it, and settles.
- A readout: mass count, spring count, the stretch of the most-stretched spring, and the simulation time.
- The mouse: press near a mass to grab it, drag it across the canvas, and on release it keeps whatever velocity your drag was giving it. Flick the bottom-right corner and the whole sheet rings.

## Prerequisites

- Part 1, all of it: Hooke's law, the semi-implicit Euler update (force → velocity → position), and the `requestAnimationFrame` clock with its clamped `dt`.
- Comfort with Canvas 2D: `beginPath`, `moveTo`, `lineTo`, `stroke`, `arc`, `fill`.
- The one new API — mouse events — gets its own section below.

## The question Part 1 left open

Part 1 ended with an open question: one spring and one mass have no ambiguity about who pushes whom. Attach a second mass and you must decide which force acts on which particle, and that decision is the entire architecture of a mass-spring system. It's two lists:

- Masses, each with a position, a velocity, and a force accumulator.
- Springs, each with a stiffness, a rest length, and the indices of exactly the two masses at its ends.

Every physics step is three passes over those lists:

1. Seed every mass's force with gravity.
2. For each spring, compute the Hooke force once and add it to both endpoint masses, equal and opposite.
3. For each mass, run the semi-implicit Euler update Part 1 earned.

Nothing else. That's the "list of springs, each pushing on exactly the two masses at its ends" Part 1 promised, and every cloth simulator you'll read about is this loop with more springs and better material laws bolted on.

The spring families follow the standard cloth reference, the Eurographics 2002 tutorial "Cloth Animation and Rendering" (Hauth et al.). It describes the classic Provot-style mesh as one "in which the particles are connected by structural springs to counteract tension, diagonal springs for shearing, and interleaving springs for bending." Our sheet uses the first two families — 356 structural springs, 330 shear springs, 686 total. Bending springs wait for Part 4, when the cloth has to drape over a table and resisting folds becomes the whole game.

## A square with no center

Before the code, a thought experiment, because it's the reason the diagonal springs exist.

Take four masses and four springs of rest length 1, forming a square. Pin the top-left mass. Where do the other three come to rest? The top-right mass has to sit exactly 1 from the pin — anywhere on the unit circle works. The bottom-left mass the same. Given those two, the bottom-right mass has to be exactly 1 from both of them, which pins it to one of two points. So the four-mass "cloth" has a two-parameter family of rest shapes: pick any two directions from the pin, and the rhombus they span is a rest shape. Every one of those rhombi has zero stretch in all four springs, so zero force on every mass. Shear the square into a rhombus and nothing resists: it has no preferred shape.

The same mode exists in the sheet. Shear every row sideways by an amount proportional to its row number, and every cell of the grid becomes a rhombus with all its sides unchanged — no structural spring stretches at all. In the simulation, a 10° global shear of the structural-only sheet leaves a max stretch of 2 × 10⁻¹⁵: numerically zero, the mode is exact. Add the diagonals and the mode costs real energy: the same 10° shear stretches them by 9.1%.

That's what the shear springs are for, and it sets their stiffness. Wikipedia's material model has an energy term for exactly this deformation: "The energy of trellising describes the shearing of the fabric (distortion within the plane of the fabric)." The diagonals are the trellising. And they're deliberately weaker than the structural springs, the way real fabric resists shearing less than it resists stretching: the tutorial puts "very large constants" on the structural springs, "whereas for the bend and shear forces the springs have small values." So `K_STRUCT = 32` and `K_SHEAR = 16`. The sheet can still shear a little — fabric does — but now it costs force, and that's the difference between cloth and a lattice of hinges.

## The sheet, standing still

The first version of the file draws the sheet without moving it: the grid, the springs, the ceiling, and one `draw()` call. Save this as `sheet.html` and open it.

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Sheet, Part 2</title>
<style>
  body { margin: 0; background: #101418; }
  canvas { display: block; margin: 20px auto; background: #161c22; }
</style>
</head>
<body>
<canvas id="c" width="800" height="600"></canvas>
<script type="module">
const canvas = document.getElementById("c");
const ctx = canvas.getContext("2d");

const PX_PER_M = 50;
const G = 9.81;

const COLS = 16;
const ROWS = 12;
const SPACING = 0.6;   // m, distance between neighbors
const MASS = 0.1;      // kg, each mass
const K_STRUCT = 32;   // N/m, horizontal and vertical springs
const K_SHEAR = 16;    // N/m, diagonal springs
const C = 0.5;         // N·s/m, viscous damping per mass

const X0 = 3.5;   // m, left edge of the sheet (175 px)
const Y0 = 1.6;   // m, pinned row (80 px, on the ceiling)

const N = COLS * ROWS;
const x = new Float64Array(N);
const y = new Float64Array(N);
const vx = new Float64Array(N);
const vy = new Float64Array(N);
const pinned = new Array(N).fill(false);
const fx = new Float64Array(N);
const fy = new Float64Array(N);
const springs = [];

function addSpring(a, b, k) {
  springs.push({ a, b, k, rest: Math.hypot(x[b] - x[a], y[b] - y[a]) });
}

function idx(col, row) { return row * COLS + col; }

// The grid: every position is assigned before any spring is built.
for (let row = 0; row < ROWS; row++) {
  for (let col = 0; col < COLS; col++) {
    const i = idx(col, row);
    x[i] = X0 + col * SPACING;
    y[i] = Y0 + row * SPACING;
    if (row === 0) pinned[i] = true;
  }
}

// Structural springs: right and down from each mass.
for (let row = 0; row < ROWS; row++) {
  for (let col = 0; col < COLS; col++) {
    const i = idx(col, row);
    if (col + 1 < COLS) addSpring(i, idx(col + 1, row), K_STRUCT);
    if (row + 1 < ROWS) addSpring(i, idx(col, row + 1), K_STRUCT);
  }
}

// Shear springs: both diagonals of every cell.
for (let row = 0; row < ROWS - 1; row++) {
  for (let col = 0; col < COLS - 1; col++) {
    addSpring(idx(col, row), idx(col + 1, row + 1), K_SHEAR);
    addSpring(idx(col + 1, row), idx(col, row + 1), K_SHEAR);
  }
}

function drawCeiling(x, yTop, width) {
  ctx.beginPath();
  ctx.moveTo(x - width / 2, yTop);
  ctx.lineTo(x + width / 2, yTop);
  ctx.stroke();
  const marks = Math.floor(width / 16);
  for (let i = 0; i < marks; i++) {
    const hx = x - width / 2 + 10 + i * 16;
    ctx.beginPath();
    ctx.moveTo(hx, yTop);
    ctx.lineTo(hx - 8, yTop - 10);
    ctx.stroke();
  }
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.lineJoin = "round";

  drawCeiling(400, Y0 * PX_PER_M, 480);

  for (const s of springs) {
    ctx.strokeStyle = s.k === K_SHEAR ? "#3a4552" : "#8fb3ff";
    ctx.lineWidth = s.k === K_SHEAR ? 1 : 1.5;
    ctx.beginPath();
    ctx.moveTo(x[s.a] * PX_PER_M, y[s.a] * PX_PER_M);
    ctx.lineTo(x[s.b] * PX_PER_M, y[s.b] * PX_PER_M);
    ctx.stroke();
  }

  ctx.fillStyle = "#e8ecf0";
  for (let i = 0; i < N; i++) {
    ctx.beginPath();
    ctx.arc(x[i] * PX_PER_M, y[i] * PX_PER_M, 3, 0, Math.PI * 2);
    ctx.fill();
  }
}

draw();
</script>
</body>
</html>
```

A few notes on the constants, because the constants are the material:

- `PX_PER_M = 50` — the same scale as Part 1: 1 meter is 50 canvas pixels.
- `COLS = 16`, `ROWS = 12`, `SPACING = 0.6` — the sheet is 9 m × 6.6 m, or 450 × 330 px on the canvas.
- `MASS = 0.1` per node, `K_STRUCT = 32`, `K_SHEAR = 16` — the values from the last section.
- `C = 0.5` — half of Part 1's damping, and both values measured: at `C = 1.0` the sheet takes about 7.5 s to settle to millimeter stillness, at `C = 0.5` about 3.4 s. `0.5` leaves a ripple you can actually see decay.
- The top row is pinned at `Y0 = 1.6 m` (80 px, the same ceiling line as Part 1).

Two things to notice in the construction code. First, the grid loop assigns every position before any spring is built. `addSpring` computes the rest length from the current positions, so a spring built before its endpoint has a position gets a rest length measured from `(0, 0)` — a silent bug that shows up as a sheet that tears itself apart the moment physics starts. Second, the pinned masses stay in the grid: springs attach to them and pull on them, but the integrator skips them. The ceiling is a force, not a boundary.

What you should see: a bright blue grid with darker diagonals under a hatched ceiling line. No readout, no clock, no motion — the sheet, standing still.

## Forcing 192 masses

Now the physics. Insert this function between `draw()` and the final `draw();` call:

```js
// The physics step: forces, then semi-implicit Euler.
function step(dt) {
  // Every mass starts the step with gravity on it; the springs add on top.
  for (let i = 0; i < N; i++) { fx[i] = 0; fy[i] = MASS * G; }

  // Each spring acts on exactly the two masses at its ends.
  for (const s of springs) {
    const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a];
    const dist = Math.hypot(dx, dy);
    const f = s.k * (dist - s.rest);   // Hooke: stretch pulls, compression pushes
    const ux = dx / dist, uy = dy / dist;
    fx[s.a] += f * ux;  fy[s.a] += f * uy;
    fx[s.b] -= f * ux;  fy[s.b] -= f * uy;
  }

  // Semi-implicit Euler, the order Part 1 earned: velocity first, then position.
  for (let i = 0; i < N; i++) {
    if (pinned[i]) { vx[i] = 0; vy[i] = 0; continue; }
    vx[i] += (fx[i] - C * vx[i]) / MASS * dt;
    vy[i] += (fy[i] - C * vy[i]) / MASS * dt;
    x[i] += vx[i] * dt;
    y[i] += vy[i] * dt;
  }
}
```

Read pass two slowly, because it is the architecture: each spring computes one force and adds it to exactly the two masses at its ends, equal and opposite. Each spring pushes and pulls equally at its two ends, so the spring forces cancel in pairs — they can never accelerate the sheet as a whole. Only gravity, and the ceiling through the pinned row, can.

> [!PREDICT]
> Before the clock arrives, work this out: how far should the sheet sag? Look at one column. The topmost vertical spring of the column carries 11 masses below it, the next carries 10, and so on down to 1. Sum the loads, apply Hooke's law with `k = K_STRUCT`, and you have that column's sag. What does it say the bottom row is at?

The arithmetic: the column's total drop is `11 + 10 + … + 1 = 66` copies of `MASS · G / K_STRUCT` — `0.1 × 9.81 / 32 × 66 ≈ 2.02 m`, about 101 px — so the column calculation puts the bottom row at `80 + 330 + 101 ≈ 511` px. Its topmost spring carries `11 × 0.1 × 9.81 ≈ 10.8 N`, stretches `10.8 / 32 ≈ 0.34 m`, and reads 56% of its 0.6 m rest length.

Don't run it yet — `step()` isn't called from anywhere, so `sheet.html` still draws the sheet standing still. The column calculation is a prediction: 511 px for the bottom row, 56% at the top springs. The mouse section is next, the clock comes after it, and that's when the prediction gets checked.

**The accumulator callback.** Part 1's design note predicted the fixed-timestep accumulator would come back "when this series gets there" — at hundreds of masses. There are 192 now, so here's the measurement: one physics step takes about 24 µs on this machine, about 0.1% of a 16.7 ms frame. Two catch-up steps after a hiccup: 0.2%. Part 1's note was wrong by about three orders of magnitude — 192 masses are cheap, because each one is a few multiplications over flat arrays. The accumulator can wait. It comes back in Part 4, when the cloth meets the table and contact checks make every step expensive enough to matter.

**The compression cheat, revisited.** Part 1 named the linear-spring cheat and promised to revisit it "when springs start pulling on each other." That's this part. A hanging sheet doesn't fold, but a dragged one does: drag the bottom-right corner across toward the left and the sheet buckles, some springs compress, and the compressed linear springs push back like thin rods, so the fold resists more than real fabric would. Keep the drags modest and the stretch stays small relative to rest lengths, the same discipline Part 1 kept; Part 4 will do it properly with tension-only springs when the cloth drapes.

## Grab it with the mouse

Part 1's last exercise becomes the core interaction. Insert this block after `step()`, still before the final `draw();`:

```js
// --- Grab a mass and drag it ---
let grabbed = -1;
const mouse = { x: 0, y: 0, px: 0, py: 0 };
const MAX_FLICK = 40;   // m/s, a hard flick, not a rocket launch

// e.clientX / e.clientY are viewport coordinates (measured from the
// window's corner). The rect is where the canvas actually sits; the
// difference is canvas coordinates in pixels, and PX_PER_M makes them meters.
function canvasPos(e) {
  const rect = canvas.getBoundingClientRect();
  return { x: (e.clientX - rect.left) / PX_PER_M,
           y: (e.clientY - rect.top) / PX_PER_M };
}

canvas.addEventListener("mousedown", (e) => {
  const p = canvasPos(e);
  let best = -1, bestD = 0.4;   // 0.4 m = 20 px grab radius
  for (let i = 0; i < N; i++) {
    if (pinned[i]) continue;
    const d = Math.hypot(x[i] - p.x, y[i] - p.y);
    if (d < bestD) { bestD = d; best = i; }
  }
  grabbed = best;
  mouse.x = mouse.px = p.x;
  mouse.y = mouse.py = p.y;
});

canvas.addEventListener("mousemove", (e) => {
  const p = canvasPos(e);
  mouse.x = p.x;
  mouse.y = p.y;
});

canvas.addEventListener("mouseup", () => { grabbed = -1; });
```

The one new API is `e.clientX` and `e.clientY`: MDN defines them as "The X coordinate of the mouse pointer in viewport coordinates" (and the Y analog). Viewport coordinates are measured from the window's corner, not the canvas's, and the canvas sits somewhere in the window. `canvas.getBoundingClientRect()` is how we learn where: it "returns a DOMRect object providing information about the size of an element and its position relative to the viewport." Subtract the rect's offset from the event, and you have canvas coordinates in pixels; divide by `PX_PER_M`, and you have the simulation's meters. (The MDN canvas tutorial notes "By default, one unit on the canvas is exactly one pixel," so pixels and simulation units are the same number.)

> [!PREDICT]
> Say you skip the `getBoundingClientRect()` step and use `e.clientX` directly. Your window is 1920 px wide, the canvas is 800 px and centered. You press on a mass in the middle of the sheet. What does the grab test actually see, and what happens?

The canvas's left edge is at `(1920 − 800) / 2 = 560` px. Without the subtraction, the grab test sees a point 560 px to the right of your cursor — past the sheet's right edge, where there is no mass — and the press does nothing. Press anywhere else on the sheet and it stays that way: the test point is always beyond the sheet, and the sheet looks dead to the mouse. The subtraction is the whole coordinate fix; the rest of the block is distance math.

Two design choices to notice:

- **Grab radius 0.4 m (20 px).** You don't have to land exactly on a 3-px dot; the nearest free mass within 20 px wins. Pinned masses are skipped — you can drag the sheet, but you can't move the ceiling.
- **The release velocity comes from the drag, not the physics.** Each frame, the grabbed mass's velocity is set to the mouse's movement that frame, `(mouse.x − mouse.px) / dt`, and on `mouseup` the mass keeps it. That's the flick: the last motion of your hand is the first motion of the physics. `MAX_FLICK` caps that release velocity at 40 m/s. Position is not clamped — the grabbed mass sits wherever the cursor is, and a fast cursor jump teleports it — so the cap is a plausibility guard, not a stability one. A hard 80 px/frame yank (96 m/s) peaks the readout near 750% without the cap and around 600% with it; the numerics survive either way, because this is a linear system and its stability depends on step size, not amplitude. The guard for step size is the `dt` clamp in the next section, and the checkpoint shows what happens without it.

Still nothing moves: `step()` has no clock yet. Next section.

## The clock starts the machine

The last two blocks. First, the clock itself, replacing the final `draw();`:

```js
let last = null;
let t = 0;

function frame(now) {
  if (last === null) last = now;
  let dt = (now - last) / 1000;   // rAF hands us milliseconds; physics wants seconds
  last = now;
  dt = Math.min(dt, 1 / 30);      // never take a step bigger than 1/30 s

  t += dt;
  step(dt);
  if (grabbed >= 0) {
    vx[grabbed] = Math.max(-MAX_FLICK, Math.min(MAX_FLICK, (mouse.x - mouse.px) / dt));
    vy[grabbed] = Math.max(-MAX_FLICK, Math.min(MAX_FLICK, (mouse.y - mouse.py) / dt));
    x[grabbed] = mouse.x;
    y[grabbed] = mouse.y;
  }
  mouse.px = mouse.x;
  mouse.py = mouse.y;
  draw();
  requestAnimationFrame(frame);
}

requestAnimationFrame(frame);
```

The frame is Part 1's frame with one addition: after `step(dt)`, the grabbed mass is snapped to the cursor and given the drag's velocity. The order is the series' order: physics first, then this frame's user input, then the draw. While a mass is held, the springs still pull on it as part of the physics, but its position and velocity are overwritten by the mouse every frame, so the drag tracks the cursor one-to-one instead of turning into a springy tug-of-war.

Run it: the sheet falls, sags, ripples, and settles. Grab a corner, drag it around, flick it, and watch it ring.

Now the readout, at the end of `draw()`, after the mass dots:

```js
  // Readout: the most stretched spring in the whole sheet.
  let maxStretch = 0;
  for (const s of springs) {
    const d = Math.hypot(x[s.b] - x[s.a], y[s.b] - y[s.a]) / s.rest - 1;
    if (d > maxStretch) maxStretch = d;
  }
  ctx.fillStyle = "#9aa4b2";
  ctx.font = "14px monospace";
  ctx.fillText(`${N} masses · ${springs.length} springs · max stretch ${(maxStretch * 100).toFixed(1)}%   t = ${t.toFixed(1)} s`, 12, 28);
```

It walks all 686 springs and reports the most-stretched one — the number the PREDICTs in this part were asking for.

Run it again, with the readout. The sheet falls, sags, ripples, and settles; the readout climbs from 0.0% and holds. The column calculation's answer lands here: the bottom row comes to rest at about 479 px — 32 px higher than its 511 — and the readout settles at 42.4%, not the column calculation's 56%. By about 2 s the sheet is under 1 cm/s; by about 3.4 s it's still to a millimeter.

The column calculation assumed each column hangs on its own. It doesn't: the diagonal springs share load between neighboring columns, and the tutorial flags exactly this — "The diagonal shear springs, for instance, also lead to additional tension and transversal contraction." Every vertical spring stretches less than a lonely column would. Remove the diagonals and the sheet reproduces the column calculation to the pixel — 101 px of sag, 56% at every top spring — which is Exercise 3.

There's a second number hiding in the 42.4%: that's the edge columns. The leftmost and rightmost columns have the fewest diagonal neighbors, so their top springs carry the most load: 42.4% at the edges, 35.0% for the interior columns. The sheet's rest shape is a compromise among all 686 springs, not 16 independent columns.

Then the numbers for a deliberate drag. Grab the bottom-right corner and drag it 100 px (2 m) to the right: the readout spikes to about 100% — about 150% for a brisk drag — the sheet rings at about 4 hertz, and after release it's still again in about 5 s. Drag a corner in a circle and the sheet ripples the way fabric does, because that's all it is: 192 masses, 686 springs, and the one-line fix.

## Checkpoint

> [!PREDICT]
> With the file closed, from memory: where does the bottom row rest, what does the readout say at rest, and how long does settling take?

**Run this to verify your work so far:** build the file from the blocks in the sections above, then open `sheet.html` in a browser (double-click it, or `python3 -m http.server` from its folder and visit the page).

Expected: the bottom row rests at about 479 px from the top of the canvas — a 69 px sag below its starting 410 px — and the readout holds at `192 masses · 686 springs · max stretch 42.4%`. The sheet is under 1 cm/s within about 2 s of loading and under 1 mm/s by about 3.4 s. Drag the bottom-right corner 100 px (2 m) to the right: the readout spikes to about 100% (about 150% for a brisk drag), the sheet rings at about 4 hertz, and it's still again about 5 s after release. The readout's `t` keeps counting after the sheet has settled: the simulation runs even when nothing visible moves, and the readout is the proof. The PREDICT answer: bottom row about 479 px, readout `42.4%`, settled in about 2–3.4 s.

**Likely errors:**
- If the sag is 101 px and the readout rests at 56%, the diagonal springs are missing — `springs.length` reads 356, not 686. That's the column calculation's prediction, and it's the sheet's honest behavior without shear resistance; "A square with no center" has the deeper reason.
- If the bottom row is off the canvas, `K_STRUCT` has a decimal slip: 3.2 instead of 32 makes the sag ten times deeper — about 690 px — and the bottom row sits near 1100 px on a 600 px canvas.
- If the sheet ignores the mouse, `getBoundingClientRect()` is missing or misused: the grab test looks 560 px to the right of the cursor (a 1920 px window with the 800 px canvas centered), which is always past the sheet's right edge, so no press finds a mass.
- If you switch tabs while the sheet is still falling and it comes back settled in the wrong shape — the bottom row about 35 px higher, and it stays there — the `dt` clamp is gone. The 10-second gap becomes one 10-second step, 300× the 1/30 s step this file is built around, and the sheet lands in a different resting shape. With the clamp, a tab switch is harmless: the sheet resumes exactly where it was.

## What's next

Part 3 leaves the 2D canvas. The same masses and springs go into 3D, rendered with Three.js, as a bouncy jelly cube you can grab and throw. The physics is nearly identical — same two lists, same three passes — and the part is about what changes when "draw" stops meaning "lines on a canvas."

Part 4 is the one you came for: a square cloth draped over a round table. It needs the three ingredients this part has been saving: bending springs (the "interleaving springs" from the Eurographics tutorial — a draped cloth has to fold, not crumple into a lattice), tension-only springs (the end of Part 1's compression cheat: cloth resists stretching, not compression), and contact (the table pushes back, and contact checks are what make steps expensive enough to need the fixed-timestep accumulator Part 1 parked).

One more method is on the horizon, and it's worth naming before Part 4: it breaks the link between stiffness and wobble. This series uses force-based integration: springs compute forces, and the integrator turns them into motion. That choice carries a trade-off — a sheet stiff enough to sag only 69 px rings at about 4 hertz, and you can't have both a stiff, flat sheet and a slow, lazy one with these springs. Position-based dynamics breaks that link by treating springs as constraints on positions rather than forces; Part 4 reaches for it when the tablecloth needs to be both stiff and calm. (This part keeps the force-based integrator on purpose: it's the one Part 1 earned, and it's the one that makes the readout's percentages mean something.)

## Exercises

- [ ] **Pin the corners instead.** Replace the top-row pinning with `pinned[i] = true` for the four corner masses only. The sheet hangs from four points like a hammock: the deepest point drops to about 579 px (a 169 px sag) and the readout settles at about 159%. How does the wobble feel different from the top-pinned sheet?
- [ ] **Halve the stiffness.** Set `K_STRUCT = 16` and `K_SHEAR = 8`. Predict the sag before running (hint: sag scales as 1/k — the diagonals halve too and still share a little load, so the real sag lands a bit under your estimate). Then check: the bottom center drops to about 541 px, a 131 px sag, and the readout settles at about 79%. The wobble slows by about √2 — flick a corner, count the rings, and confirm.
- [ ] **Remove the shear springs.** Delete the two diagonal `addSpring` calls; `springs.length` should read 356. The sag goes from about 69 px to the column calculation's 101 px, and every top spring stretches 56% — no edge/interior difference, because nothing shares load anymore. Then grab the bottom-right corner, drag it about 1 m (50 px) to the right, and hold it a moment: the readout peaks at about 62%, against about 45% with the diagonals, and the sheet takes about 9 s to settle against about 6. The deeper reason is in "A square with no center": without diagonals, the sheet can be sheared into a grid of rhombi with no spring stretched at all, so a sheared sheet has no force pushing it back.
- [ ] **Add bending springs.** For every mass that has a neighbor two steps to its right, add a spring of `k = 4` between them; do the same two steps down. `springs.length` becomes 1014, the deepest point rises from about 479 px to about 463 px, and the readout drops from 42.4% to about 36%. Bending stiffness is what a real tablecloth has and this sheet doesn't; Part 4 will need it.

## Sources

Cloth and spring systems:

1. [Cloth simulation — Wikipedia](https://en.wikipedia.org/wiki/Cloth_simulation) — the particle-mesh and mass-spring model families; the bending and trellising energy terms.
2. [Hauth, Etzmuss, Eberhardt, Klein, Sarlette, Sattler, Daubert, Kautz, "Cloth Animation and Rendering," Eurographics 2002 Tutorial T3](http://www.mirkosattler.de/publications/2002_tutorial_eg.pdf) — structural, shear, and bending spring families; the stiffness hierarchy; diagonal load sharing.

Browser APIs:

3. [MouseEvent — MDN](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent) — `clientX` / `clientY`, viewport coordinates.
4. [Element.getBoundingClientRect() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect) — the element's size and position relative to the viewport.
5. [Canvas Transformations — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations) — one canvas unit is one pixel by default.
