# Simple Physics in the Browser, Part 1: A Spring That Stretches, Bounces, and Settles

In 1678, Robert Hooke published the solution to a Latin anagram he had been holding back since 1676: *ut tensio, sic vis* — "as the extension, so the force." The force a spring exerts is proportional to how far you stretch it. That is the entire law, and it is about 350 years old.

Everything in this series is built from it: a stretchy bouncy sheet, a wobbling jelly cube, a square of cloth that drapes over a round table. Each one is a pile of springs pretending to be matter, and every spring in every one of them obeys that proportionality.

You can simulate a believable spring in under 40 lines of JavaScript, in a single HTML file with no dependencies. And along the way you will meet the one place the whole enterprise goes wrong: the most obvious way to step the physics forward does not just lose a little accuracy — it makes the spring pump itself up forever, until the mass leaves the canvas. By the end of this part you will know the two-line fix, and why it works.

## What you'll build

One HTML file, `spring.html`. It draws an 800×600 canvas with a ceiling bar, a spring hanging from it, and a 1 kg ball. The ball starts with the spring already stretched 1.2 meters past its natural length; you release it, and it wobbles up and down with a period of about 1.4 seconds before settling at a rest position 24.5 pixels below the natural length. About 40 of the roughly 100 lines of the file are physics; the rest is drawing and bookkeeping.

## Prerequisites

- Basic JavaScript: variables, functions, `for` loops, template literals.
- Some canvas experience — you have drawn on a 2D canvas before.
- Enough physics to know that force changes velocity, and that gravity pulls at 9.81 m/s² near Earth's surface.
- A current browser. Nothing to install; the file works by opening it directly.

## One spring, one mass, one law

The setup is the classic one: a spring with a mass on its lower end, hanging from a fixed anchor. Hooke's law says the spring's restoring force is "proportional to the extension" — stretch the spring by `x` and it pulls back with force `k·x`, where `k` is the spring constant, [measured in newtons per meter (N/m)](https://en.wikipedia.org/wiki/Hooke%27s_law). In the [restoring-force convention](https://en.wikipedia.org/wiki/Hooke%27s_law), the force the spring exerts "is opposite to that of the displacement": pull it down, it pulls up.

Two other forces join in. Gravity pulls the mass down with `m·g`. And real springs lose energy to friction; the standard model for that friction is a force "proportional to the velocity" of the mass, `c·v`, always opposing the motion, where `c` is the [viscous damping coefficient](https://en.wikipedia.org/wiki/Damped_harmonic_oscillator). Here are the three, on the mass:

```svg
<svg viewBox="0 0 360 200" role="img" aria-label="Free-body diagram: spring force up, gravity down, damping opposing velocity">
  <polyline points="180,8 188,20 172,32 188,44 172,56 188,68 172,80 180,92" fill="none" stroke="currentColor" stroke-width="2"/>
  <rect x="150" y="92" width="60" height="44" fill="none" stroke="currentColor" stroke-width="2"/>
  <text x="180" y="120" font-size="13" font-family="sans-serif" fill="currentColor" text-anchor="middle">m</text>
  <line x1="180" y1="136" x2="180" y2="182" stroke="#e07850" stroke-width="2"/>
  <polyline points="174,174 180,182 186,174" fill="none" stroke="#e07850" stroke-width="2"/>
  <text x="192" y="176" font-size="13" font-family="sans-serif" fill="#e07850">mg</text>
  <line x1="244" y1="114" x2="244" y2="58" stroke="#7a66ff" stroke-width="2"/>
  <polyline points="238,66 244,58 250,66" fill="none" stroke="#7a66ff" stroke-width="2"/>
  <text x="256" y="84" font-size="13" font-family="sans-serif" fill="#7a66ff">k(y − L0)</text>
  <line x1="116" y1="114" x2="116" y2="70" stroke="currentColor" stroke-width="2" stroke-dasharray="5 4"/>
  <polyline points="110,78 116,70 122,78" fill="none" stroke="currentColor" stroke-width="2"/>
  <text x="104" y="84" font-size="13" font-family="sans-serif" fill="currentColor" text-anchor="end">cv (opposes v)</text>
</svg>
```

Measure everything from the anchor, downward, in meters: `y` is the distance from anchor to mass, and `L0` is the spring's natural length — the length it has when unstretched. The extension is `y − L0`, so the spring's force (downward-positive) is `−K·(y − L0)`. Newton's second law adds the three forces:

```
m·a = m·g − K·(y − L0) − C·v
```

which gives the acceleration our simulation will actually use:

```
a = G − (K / M)·(y − L0) − (C / M)·v
```

One known cheat is worth naming now, the way you'd specify behavior on weird input. This is a *linear* spring: the law keeps holding if `y − L0` goes negative (compression) all the way to zero and below. A real coil binds at some minimum length and stops pulling. In this series we never let the simulation get anywhere near that — the amplitudes stay small relative to `L0` — and Part 2 will revisit it when springs start pulling on each other.

The constants, in SI units, chosen so the motion is slow enough to watch:

```js
const L0 = 2.0;   // natural length, m
const M  = 1.0;   // mass, kg
const K  = 20;    // spring constant, N/m
const C  = 0;     // damping coefficient, N·s/m — added in a later section
const G  = 9.81;  // gravity, m/s²
const PX_PER_M = 50;  // 1 meter = 50 pixels on the canvas
```

Two consequences of these numbers are worth knowing before you run anything, because both come straight from the formulas.

**Where it settles.** At rest, acceleration and velocity are both zero, so `G = (K/M)·(y − L0)` and the mass hangs at

```
y_eq = L0 + M·G/K = 2.0 + 9.81/20 = 2.4905 m
```

The spring is stretched `M·G/K = 0.4905 m` at rest — 24.5 pixels. Gravity does not make the spring longer by an amount you get to choose; it is fixed by the ratio of weight to stiffness.

**How fast it wobbles.** A mass on a spring executes [simple harmonic motion](https://en.wikipedia.org/wiki/Simple_harmonic_motion): "a special type of periodic motion an object experiences by means of a restoring force whose magnitude is directly proportional to the distance of the object from an equilibrium position." Its period is

```
T = 2π·√(M/K) = 2π·√(1/20) ≈ 1.4 s
```

and the motion is [isochronous](https://en.wikipedia.org/wiki/Simple_harmonic_motion) — "the period and frequency are independent of the amplitude and the initial phase of the motion." Pull it back 10 cm or 1 m; the period is the same 1.4 s.

We start the mass stretched 1.2 m past natural length, at rest: `y = L0 + 1.2 = 3.2 m`, `v = 0`. That is 0.71 m (35.5 px) below the equilibrium point, so the wobble will swing roughly ±35 px around a rest position of `80 + 2.4905·50 ≈ 205` pixels from the top of the canvas.

## A page and a static spring

The file is one HTML document with one inline module script. Two facts about module scripts make this layout work. [Modules use strict mode automatically](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules), so you get `const`-discipline for free, and [they are deferred automatically](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) — they run after the HTML is parsed, which means `document.getElementById("c")` already works. And because this script imports *nothing*, the browser fetches nothing, and the file runs when you open it straight from disk. The moment you `import` a separate file, module scripts are [fetched with CORS](https://jakearchibald.com/2017/es-modules-in-browsers/), and `file://` stops cooperating — Part 3, where Three.js comes in, deals with that.

Save this as `spring.html`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Spring, Part 1</title>
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

// Everything is simulated in meters; the canvas is in pixels.
// PX_PER_M is the bridge between the two.
const PX_PER_M = 50;

// The spring hangs from this point (pixels).
const ANCHOR = { x: 400, y: 80 };

// Physics, in SI units.
const L0 = 2.0;   // natural length, m
const M  = 1.0;   // mass, kg
const K  = 20;    // spring constant, N/m
const C  = 0;     // damping coefficient, N·s/m — added later
const G  = 9.81;  // gravity, m/s²

// State. y is the distance from anchor to mass (m), positive downward.
// Start with the spring already stretched 1.2 m past natural length.
let y = L0 + 1.2;
let v = 0;

function drawCeiling(x, yTop, width) {
  ctx.beginPath();
  ctx.moveTo(x - width / 2, yTop);
  ctx.lineTo(x + width / 2, yTop);
  ctx.stroke();
  for (let i = 0; i < 6; i++) {
    const hx = x - width / 2 + 10 + i * 16;
    ctx.beginPath();
    ctx.moveTo(hx, yTop);
    ctx.lineTo(hx - 8, yTop - 10);
    ctx.stroke();
  }
}

function drawSpring(x, yTop, yBottom, coils, halfWidth) {
  const lead = 10;
  const yStart = yTop + lead, yEnd = yBottom - lead;
  const step = (yEnd - yStart) / (coils * 2);
  ctx.beginPath();
  ctx.moveTo(x, yTop);
  ctx.lineTo(x, yStart);
  let sign = 1;
  for (let i = 0; i < coils * 2; i++) {
    ctx.lineTo(x + sign * halfWidth, yStart + step * (i + 0.5));
    sign *= -1;
  }
  ctx.lineTo(x, yBottom);
  ctx.stroke();
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.strokeStyle = "#e8ecf0";
  ctx.lineWidth = 2;
  ctx.lineJoin = "round";

  drawCeiling(ANCHOR.x, ANCHOR.y, 120);

  const massY = ANCHOR.y + y * PX_PER_M;
  drawSpring(ANCHOR.x, ANCHOR.y, massY, 8, 12);

  ctx.fillStyle = "#8fb3ff";
  ctx.beginPath();
  ctx.arc(ANCHOR.x, massY, 14, 0, Math.PI * 2);
  ctx.fill();
}

draw();
</script>
</body>
</html>
```

Open it in a browser. You should see a hatched ceiling bar, an 8-zigzag coil, and a blue ball about 205 pixels from the top. The spring's rendered length is `y·PX_PER_M = 160` pixels, so the zigzag is drawn at full stretch — watch what it looks like in the compressed state before anything moves.

Two canvas details in there are worth a sentence each. `lineJoin = "round"` [rounds off the corners](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/lineJoin) where the zigzag segments meet — the default `"miter"` makes sharp spikes at every peak, which looks like a saw, not a coil. And `lineWidth = 2` sets the stroke thickness; [the default is 1.0](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/lineWidth), which reads as faint at this scale.

## The frame clock

Physics needs time. The browser's animation clock is `requestAnimationFrame`: it "requests the browser to call a user-supplied callback function before the next repaint," and [the call frequency "will generally match the display refresh rate"](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) — 60 Hz on most monitors, "though 75hz, 120hz, and 144hz are also widely used." The callback receives one argument: a timestamp in milliseconds, which is how you measure how long the previous frame took.

Replace the final `draw();` line with this:

```js
let last = null;
let t = 0;

function frame(now) {
  if (last === null) last = now;
  let dt = (now - last) / 1000;   // rAF hands us milliseconds; physics wants seconds
  last = now;
  dt = Math.min(dt, 1 / 30);      // never take a step bigger than 1/30 s

  t += dt;
  draw();
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

The loop is the standard one: each frame, ask how much time passed since the last frame (`dt`), then re-render. Notice that nothing runs yet — `dt` is measured and thrown away. The page looks exactly as it did, only now it is being redrawn 60 times a second.

That `Math.min` line is doing real work. [requestAnimationFrame calls are paused in background tabs](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame): switch away for ten minutes, come back, and the first `dt` is ten minutes. Without the clamp, one update step with `dt = 600` would hurl the mass across the page. Capping the step at 1/30 s means the worst case is a small slow-motion hiccup instead of a catastrophe.

> [!DESIGN-NOTE]
> **The rigorous version: fixed timestep with an accumulator.**
>
> Passing the frame's actual `dt` to the physics is what this series does, and it is fine while one mass is the whole simulation. The stricter pattern, described in Glenn Fiedler's [Fix Your Timestep](https://gafferonggames.com/post/fix_your_timestep/), is to give the simulation a *fixed* `dt` — say 1/60 s — and bank leftover frame time in an accumulator:
>
> ```c
> accumulator += frameTime;
> while (accumulator >= dt) {
>   integrate(state, t, dt);
>   accumulator -= dt;
> }
> ```
>
> Fiedler's argument is that "it's much more realistic to say that your simulation is well behaved only if delta time is less than or equal to some maximum value," and this loop guarantees that. It also means a slow machine and a fast one run identical physics, at different render rates. The cost is that a frame that falls behind has to catch up with several steps at once, which — if the physics is the expensive part — can produce the "spiral of death," where the simulation "can't keep up with the steps it's asked to take." One mass never comes close to that. Hundreds of masses will. When this series gets there, the accumulator comes back.

Add one more thing to `draw()`, just before the ceiling, so you can watch the state while you work:

```js
  ctx.fillStyle = "#9aa4b2";
  ctx.font = "14px monospace";
  ctx.fillText(`y = ${y.toFixed(3)} m   v = ${v.toFixed(3)} m/s   t = ${t.toFixed(1)} s`, 12, 28);
```

`fillText` [draws a string at the given coordinates in the current fill style](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/fillText). Refresh: the clock now ticks — `t` climbs by about 0.016 per frame at 60 Hz — but the mass is still frozen, because nothing has computed an acceleration.

## The obvious first step, and the bug it ships with

You have `a` as a function of `(y, v)`, and you know that velocity is the rate of change of position and acceleration is the rate of change of velocity. The most natural step forward uses the values you have on hand:

```js
  // Forward Euler, inside frame(), before draw():
  const a = G - (K / M) * (y - L0) - (C / M) * v;
  y += v * dt;
  v += a * dt;
```

This is the [Euler method](https://en.wikipedia.org/wiki/Euler_method) — "a first-order numerical procedure for solving ordinary differential equations" — and it is the first thing anyone writes. Open the page and watch for ten seconds.

The wobble starts at the predicted size, and then it *grows*. At 60 frames per second the amplitude climbs by roughly a fifth per second: doubled in about 4 seconds, several times the original by 10 seconds, and the mass leaves the top of the canvas somewhere in there. The readout stops being a meter and starts being a stock ticker. Nothing in the code is a typo. The integrator is adding energy to the system every single step.

Why, in one paragraph? The true motion of an undamped spring is a closed loop in the plane of `(position, velocity)`. Forward Euler approximates that loop with straight segments, and because each position update uses the velocity from *before* the push, the polygonal path lands just outside the true loop at every step. The orbit spirals outward: "the energy increases steadily when the standard Euler method is applied," in the words of the [semi-implicit Euler method](https://en.wikipedia.org/wiki/Semi-implicit_Euler_method) article — and the [Euler method](https://en.wikipedia.org/wiki/Euler_method) article puts the same thing in its instability section: "the numerical solution grows very large for equations where the exact solution does not." Your spring is the textbook case.

> [!PREDICT]
> Before the fix: with `C = 0` and the naive ordering, is there any `dt` small enough that the wobble stops growing — or does it grow for every positive `dt`?

The second half of the answer is "it grows for every positive `dt`; making `dt` smaller only makes the growth slower." The first half is the fix, and it is smaller than you'd expect.

## The one-line fix: semi-implicit Euler

In `frame()`, swap the order of the two update lines:

```js
  // Semi-implicit (symplectic) Euler:
  // compute the force from the current state, update the velocity,
  // then move with the *new* velocity.
  const a = G - (K / M) * (y - L0) - (C / M) * v;
  v += a * dt;
  y += v * dt;
```

That is the entire difference. The [semi-implicit Euler method](https://en.wikipedia.org/wiki/Semi-implicit_Euler_method) — "also called symplectic Euler, semi-explicit Euler, Euler–Cromer, and Newton–Størmer–Verlet," depending on who you meet at a conference — is still first order, still one force evaluation per step, and still as cheap as the naive version. But the position update now uses the velocity that already includes the push. The polygonal path in `(y, v)` no longer systematically lands outside the true loop; the method is a [symplectic integrator](https://en.wikipedia.org/wiki/Symplectic_integrator), and "as a consequence, the semi-implicit Euler method almost conserves the energy" of a system whose true energy is constant.

> [!ASIDE]
> The method "has been discovered and forgotten many times, dating back to Newton's Principiae," as recalled by Richard Feynman in his *Feynman Lectures* (Vol. 1, Sec. 9.6). Newton, in other words, got this right before anyone had a word for it.

Run it with `C = 0` still in place. The mass wobbles at the predicted 35.5 px around 205 pixels, with a period of about 1.4 s — and it *keeps* wobbling. After 30 seconds, more than twenty full periods, the amplitude is still 71.0 cm, to the centimeter, against a starting value of 71.0 cm. No growth, no decay: the integrator is doing what the physics says, and the physics says the energy is conserved.

Now the answer to the prediction: there is no `dt` that fixes forward Euler, because the energy gain is structural, not a size problem. Semi-implicit Euler is stable at this `dt` (and at any `dt` up to a limit that depends on `ω = √(K/M)`), and its small per-step error is of the kind that orbits *around* the true energy instead of climbing away from it.

## Making it settle: damping

A frictionless wobble forever is a nice demonstration and a poor toy. Set the constant that has been waiting:

```js
const C  = 1.0;   // damping coefficient, N·s/m
```

The force term `−(C/M)·v` is already in the acceleration — it has been computing zero this whole time, because `C` was zero. With `C = 1.0`, the damping ratio — the single number that sorts damped oscillators into families, `ζ = C / (2·√(M·K))` — is

```
ζ = 1 / (2·√20) ≈ 0.11
```

and the system is [underdamped](https://en.wikipedia.org/wiki/Damped_harmonic_oscillator): it will "oscillate with a frequency lower than in the undamped case, and an amplitude decreasing with time." Refresh, and the wobble now dies away. The mass is within about a pixel of its rest position after roughly 7 seconds; the last millimeter takes about 13.

The damping ratio is worth keeping as a dial, because it has named settings. With our `M` and `K`:

| `C` | `ζ` | What you see |
|---|---|---|
| 0 | 0 | Wobbles forever, constant 71 cm amplitude |
| 1.0 | 0.11 | Wobbles, dies out in about 7 s |
| 8.94 | 1.0 | No wobble — falls straight to rest in about 1.5 s |
| 18 | 2.0 | No wobble — sinks in about 4 s, slower than the critical case |

The middle row is the [critically damped](https://en.wikipedia.org/wiki/Damped_harmonic_oscillator) case — "the boundary solution between an underdamped oscillator and an overdamped oscillator." It is the fastest way to get the mass to rest without a single oscillation, and it is the default for door closers, syringe plungers, and anything an engineer does not want to see bounce. The last row is overdamped: no wobble, but it takes the long way there. Try all four values of `C` before moving on; the table is the prediction, and the canvas is the check.

## Checkpoint

> [!PREDICT]
> Before you run: the mass settles where spring force balances gravity. Compute the rest position in pixels — the anchor is at 80 px, `PX_PER_M` is 50, and `y_eq = L0 + M·G/K`. What number should the readout's `y` be converging to?

**Run this to verify your work so far:** open `spring.html` in a browser (double-click it, or `python3 -m http.server` from its folder and visit the page).

Expected: the ball starts at `y = 3.200 m` and wobbles about ±35 px around `y ≈ 2.490 m` (about 205 px on the canvas), one cycle every ~1.4 s, damping out until the readout sits still at `y = 2.490 m, v = 0.000 m/s` after about 7–13 seconds. The PREDICT answer: `80 + 2.4905·50 ≈ 205` px.

**Likely errors:**
- If the mass falls straight through the floor and never comes back, your spring force points the wrong way: it must be `- (K / M) * (y - L0)` — pulling *back toward* the natural length — not `+`. The restoring force's sign is the most common slip in this entire series.
- If nothing moves and only `t` ticks, the physics step is not running: it belongs in `frame()`, before `draw()`, not inside `draw()`.
- If the wobble still grows after you swapped the two update lines, you swapped them in a copy that isn't the one running — or you moved the `const a = ...` line below the `v += a * dt` line, so the force is computed from the *new* state. The force must be computed first, from the state at the start of the step.
- If you see `Uncaught ReferenceError: Cannot access 't' before initialization`, the original `draw();` call is still at the bottom of the script and is now running before `let t` exists. Delete it — `requestAnimationFrame(frame)` is what starts the loop.

## What's next

One spring is a law, not a system. Attach a second mass to a second spring and you immediately have a decision the single-mass page never faced: *which force goes on which particle* — and the answer (a list of springs, each pushing on exactly the two masses at its ends) is the whole architecture of Part 2, where this becomes a grid of masses and springs that behaves like a stretchy, bouncy sheet you can drag with the mouse.

Before you move on: in two sentences, why does updating the velocity *before* the position — the only difference between the two integrators — stop the wobble from growing? Write the answer that would satisfy a sceptical colleague.

## Exercises

- [ ] **Critical damping.** Set `C = 2 * Math.sqrt(K * M)` (8.944). The mass should drop to rest without a single oscillation, within a centimeter in about 1.5 s. Now set `C = 18` and confirm it takes visibly longer to get there.
- [ ] **Stiffer spring, predicted first.** Set `K = 80`. Before refreshing, compute the new period (`T = 2π·√(M/K)`) and the new rest position in pixels from `y_eq = L0 + M·G/K`. Run it and compare both. (The period should drop to about 0.7 s; the rest position rises to about 186 px.)
- [ ] **The conservation check.** Restore `C = 0` and run for 30 seconds. The amplitude should still be about 71 cm — that is the "almost conserves the energy" property, and it is why Part 2 can trust this integrator with hundreds of springs. Temporarily put the naive ordering back in to see the contrast one more time.
- [ ] **Grab the mass.** On `mousedown` near the ball, drag it: set `y` from the cursor's position (and `v = 0`) while the button is down; on `mouse up`, release it and let the physics take over. Part 2 makes this its core interaction.

## Sources

Physics references:

1. [Hooke's law — Wikipedia](https://en.wikipedia.org/wiki/Hooke%27s_law) — the force-extension proportionality, the restoring-force sign, and the N/m unit of `k`.
2. [Simple harmonic motion — Wikipedia](https://en.wikipedia.org/wiki/Simple_harmonic_motion) — the period `T = 2π√(M/K)` and isochronism: period independent of amplitude.
3. [Damped harmonic oscillator — Wikipedia](https://en.wikipedia.org/wiki/Damped_harmonic_oscillator) — the `−C·v` friction model, the damping ratio `ζ = C/(2√(M·K))`, and the underdamped/critical/overdamped families.
4. [Euler method — Wikipedia](https://en.wikipedia.org/wiki/Euler_method) — the naive first-order integrator and its instability: the numerical solution "grows very large" where the exact one does not.
5. [Semi-implicit Euler method — Wikipedia](https://en.wikipedia.org/wiki/Semi-implicit_Euler_method) — the one-line fix, its alias list, and the energy-conservation property it buys.

Browser APIs and deep-dives:

6. [Window.requestAnimationFrame — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) — callback timing, refresh-rate matching, background-tab pausing, the timestamp argument.
7. [CanvasRenderingContext2D.lineWidth — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/lineWidth) — stroke thickness and its 1.0 default.
8. [CanvasRenderingContext2D.lineJoin — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/lineJoin) — `"round"` vs `"miter"` joins for the zigzag coil.
9. [CanvasRenderingContext2D.fillText — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/fillText) — the state readout text.
10. [JavaScript modules — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) — modules run in strict mode and are deferred automatically.
11. [ES modules in browsers — Jake Archibald](https://jakearchibald.com/2017/es-modules-in-browsers/) — inline modules and the CORS rules that make `file://` work only while nothing is imported.
12. [Fix Your Timestep — Glenn Fiedler (Gaffer on Games)](https://gafferonggames.com/post/fix_your_timestep/) — the fixed-timestep accumulator and the spiral of death, for when the particle count grows.
