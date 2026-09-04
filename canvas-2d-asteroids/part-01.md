# Asteroids in One HTML File, Part 1: The Page, the Canvas, and the Ship

In 1979, Atari put a white triangle on a black background in an arcade cabinet and sold the right to move it. The game was [Asteroids](https://en.wikipedia.org/wiki/Asteroids_(video_game)) — a multidirectional shooter in which you pilot a triangular ship through an asteroid field, break rocks into smaller, faster rocks for points, and dodge the flying saucers that keep crossing the field. [The ship had five buttons](https://www.museumofplay.org/games/asteroids/): rotate left, rotate right, thrust, fire, and hyperspace.

The whole game is triangles and lines on a pixel grid. That is exactly what a browser's Canvas 2D API draws. Across four parts, we will rebuild an Asteroids clone as a single HTML file — no framework, no build step, no dependencies. This part gets the first triangle on screen, and along the way it introduces the modern JavaScript — ES modules — that will carry the rest of the game.

By the end of this part, you'll have one file, `game.html`, that opens in a browser and shows a black 800×600 field with a white triangular ship parked at its center — (400, 300) — its position stored in a plain object that the drawing code reads. The ship won't move yet. That starts in Part 2.

## What you'll build

The end state of this part is a single file you can open by double-clicking. It contains a page with one 800×600 `<canvas>`, one inline module script, and nothing else. The script paints the field black, draws the ship — a triangle with its nose at (400, 285) and its back corners at (388, 312) and (412, 312) — and logs the ship's state to the console. The ship is a drawing, not an object: nothing moves, nothing reacts to input. Part 2 adds the animation loop and the ship's rotation and thrust. Part 3 adds asteroids, bullets, and collisions. Part 4 assembles the full game: waves, scoring, lives, and the saucer.

## Prerequisites

- A modern browser — Chrome, Firefox, Safari, or Edge from the last several years. Canvas 2D and ES module support have been stable across all of them for a long time ([MDN notes canvas support in "recent versions of all major browsers"](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial)). There is no version to pin: the code in this series behaves the same in each.
- Basic JavaScript: variables, functions, `for` loops, objects. If `function add(a, b) { return a + b; }` reads fine, you're ready. The modern idioms you may not know yet — modules, `const`/`let`, arrow functions — will be introduced the first time they appear.
- A text editor.
- No installation. No npm, no framework, no build. The entire toolchain is your browser. (If you later want to split the game across files, any static file server works — `python3 -m http.server` or `npx serve` — but nothing in this part needs one.)

## A page with a box in it

The constraint for this series: one HTML file. The game, its code, and its styles all live in a single document you can open by double-clicking. That is a real constraint, not just nostalgia — it keeps the whole game shareable as one file, and it shapes how we use JavaScript, as the next section shows.

Here is the entire page. Save it as `game.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Asteroids</title>
  <style>
    body {
      margin: 0;
      min-height: 100vh;
      display: grid;
      place-items: center;
      background: #222222;
    }
    canvas {
      border: 1px solid #555555;
    }
  </style>
</head>
<body>
  <canvas id="game" width="800" height="600">
    Your browser does not support the canvas element — the game needs it.
  </canvas>

  <script type="module">
    const canvas = document.getElementById("game");
  </script>
</body>
</html>
```

Open it: a gray page with a bordered box in the middle. The box is the [canvas element](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Basic_usage) — a rectangle that, in the [spec's words](https://html.spec.whatwg.org/multipage/canvas.html), "provides scripts with a resolution-dependent bitmap canvas." A bitmap: a fixed grid of pixels that scripts can paint. Unlike `<img>`, it has no `src`; its only meaningful attributes are `width` and `height`, and those size the *coordinate space*, not the CSS display. Omit them and you get the spec's defaults, 300×150 — the size you may have seen the first time you made a canvas and wondered where those numbers came from.

The text inside `<canvas>…</canvas>` is fallback content: screen readers and browsers without canvas support show it instead of the bitmap. It costs nothing, and it is the accessible way to ship a canvas.

That's the whole page. The box is transparent right now — the spec says a fresh canvas bitmap "must be transparent black" — so you're seeing the gray page through it.

## The module script: deferred, strict, private

The script in that file is ordinary JavaScript with one attribute: `type="module"`. After the `canvas` line inside the script, add a log that proves the module ran:

```js
console.log("module ran, canvas is", canvas.width, "x", canvas.height);
```

Reload the page and open the DevTools console (F12, or right-click → Inspect → Console). You should see:

```
module ran, canvas is 800 x 600
```

Three things about that attribute matter for this series, per [MDN's module guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules):

1. **It runs late, on purpose.** Module scripts are deferred automatically — the browser finishes parsing the HTML, then runs the module. Inline module scripts are [deferred even when they import nothing](https://jakearchibald.com/2017/es-modules-in-browsers/). That's why `document.getElementById("game")` finds the canvas with no `load` event or `DOMContentLoaded` listener. A classic script sitting in `<head>` would run before the canvas element exists and get `null`.
2. **It is strict mode, automatically.** Modules use strict mode by default: sloppy patterns, like assigning to a variable you never declared, throw instead of quietly creating a global.
3. **It is private.** Top-level `const`, `let`, and function declarations in a module live in the module's own scope — they don't land on `window`. The game's state stays out of the page's global namespace, where a future library or a second script could clobber it.

One more modern idiom in that snippet, since you asked for the basics: `const`. Use `const` by default; use `let` when you must reassign. `var` is the older function-scoped form you'll meet in older tutorials; you won't see it again in this series.

> [!HEADS-UP]
> **This file works by double-click because the module is inline and imports nothing** — the browser has nothing to fetch. The moment you add `import … from "./ship.js"`, the browser must *fetch* that file, and module fetches from `file://` URLs fail the browser's cross-origin (CORS) check: modern browsers treat every local file as a [separate origin](https://barker.codes/blog/javascript-modules-and-the-file-uri-scheme/), and [MDN's own guidance](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) is to test module code through a server, not from disk. That's why this series keeps everything in one file — and it's why, if you ever split the game across files, you run it with `python3 -m http.server` (then open `http://localhost:8000/game.html`) instead of double-clicking.

## Getting the context: a pen over a grid

The canvas is a bitmap; something has to paint it. That something is the **rendering context** — you get one by calling `getContext("2d")` on the element ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Basic_usage)):

Inside the module script, after the `console.log` line:

```js
const ctx = canvas.getContext("2d");
```

`ctx` is a `CanvasRenderingContext2D`: a stateful drawing machine. It holds the current fill color, stroke color, line width, and the path you're building, and its methods paint the bitmap. Two defaults to know before you draw anything, per [MDN's drawing tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes): the default **fill and stroke colors are black** — MDN's first example draws "a large black square" without ever setting a color, and that's the default doing the work — and the default line width is **1** pixel.

The canvas has only two primitive shapes: rectangles and paths. Everything else in Asteroids — the ship, the asteroids, the saucer — is built from those. Start with a rectangle, because the playfield is one. After the `ctx` line:

```js
// The playfield: paint the whole 800×600 field black.
ctx.fillStyle = "#000000";
ctx.fillRect(0, 0, canvas.width, canvas.height);
```

`fillRect(x, y, w, h)` paints immediately — no path involved. `fillStyle` sets the fill color. Reload: the box is now solid black, and the gray page around it is all that tells you where the field ends.

## The grid: (0,0) top-left, y points down

Every coordinate you pass to a drawing call is a position in the bitmap's grid. Per [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes): "The origin of this grid is positioned in the top left corner at coordinate (0,0)" — x increases to the right, and y increases *down*, the part that trips everyone up. The spec backs the scale: one pixel of image data per coordinate space unit.

Here is the field for the rest of the series — 800×600, origin top-left, and the spot the ship will occupy. Watch the y-axis: "up" on screen is *negative* y, so the ship's nose, which points up, has a smaller y than its back.

```svg
<svg viewBox="0 0 340 250" role="img" aria-label="The 800 by 600 canvas grid: origin at top-left, x right, y down, ship triangle at center (400, 300)">
  <rect x="10" y="10" width="320" height="240" fill="none" stroke="currentColor" stroke-width="1" opacity="0.4"/>
  <line x1="10" y1="10" x2="324" y2="10" stroke="currentColor" stroke-width="1"/>
  <line x1="10" y1="10" x2="10" y2="244" stroke="currentColor" stroke-width="1"/>
  <text x="16" y="26" font-size="11" font-family="sans-serif" fill="currentColor">(0,0)</text>
  <text x="168" y="26" font-size="11" font-family="sans-serif" fill="currentColor" text-anchor="middle">x: right, to 800</text>
  <text x="16" y="134" font-size="11" font-family="sans-serif" fill="currentColor">y: down, to 600</text>
  <polyline points="170,124 165.2,134.8 174.8,134.8" fill="none" stroke="#e07850" stroke-width="2"/>
  <text x="170" y="158" font-size="11" font-family="sans-serif" fill="currentColor" text-anchor="middle">ship at (400, 300)</text>
</svg>
```

Two consequences, both of which pay off later. The ship's position (400, 300) is the exact center of the field — `canvas.width / 2`, `canvas.height / 2`. And anything drawn outside the field does not error; it simply doesn't paint. The canvas is a fixed-size bitmap, so a point at (850, 300) has no pixel to land on. The canvas will not wrap objects around the edges for you — in Part 3, wrap-around is *our* job.

## A path: lift the pen, set it down, drag

A **path** is a list of points connected by line segments: you build it up, and only then do you paint it. The dance is always the same three steps, per [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes):

1. `ctx.beginPath()` — throw away whatever path exists and start fresh.
2. `ctx.moveTo(x, y)` and `ctx.lineTo(x, y)` — lift the pen, set it down, then drag it. MDN's own phrasing for `moveTo` is lifting a pen from one spot and placing it at the next.
3. `ctx.closePath()` — drag a line back to the start of the current sub-path.

Then you commit with `ctx.stroke()` (paint the outline) or `ctx.fill()` (paint the interior). There is an asymmetry you will feel within a minute: `fill()` closes open paths automatically. `stroke()` does not.

The ship is three points — a nose and two back corners — built in exactly that dance. After the `fillRect` block:

```js
// The ship's state: its position in the field.
const ship = { x: 400, y: 300 };

// The ship: a triangle, nose up.
ctx.beginPath();
ctx.moveTo(ship.x, ship.y - 15);      // nose
ctx.lineTo(ship.x - 12, ship.y + 12); // back left
ctx.lineTo(ship.x + 12, ship.y + 12); // back right
ctx.closePath();
```

Notice the coordinates are not literals. The ship's position lives in a plain object, and the drawing reads from it. That split — state as data, drawing as a function of state — is the load-bearing decision of this whole series. Part 2's loop just re-runs this drawing with a different `ship.x` and `ship.y` every frame.

Before you paint, two ways to get this wrong.

**Wrong #1 — no `closePath()`.** Delete the `closePath()` line, stroke, and you get an open "V": two sides of the triangle and no bottom, because `stroke()` traces exactly the path you gave it.

**Wrong #2 — no `beginPath()`.** Suppose you now draw a second triangle for practice, without starting a new path:

```js
// No beginPath() here — the ship's path is still current.
ctx.lineTo(620, 150);
ctx.lineTo(580, 190);
ctx.closePath();
ctx.stroke();
```

Your second shape is fine, but a connecting line runs from wherever the pen was last — the ship's back-right corner, (412, 312) — to the first new point. The pen never lifted. `beginPath()` is how you lift it. (One mercy: on an empty path — right after `beginPath()` — the first command is treated as a `moveTo` anyway, so you can skip the explicit `moveTo` in that one case.)

## The ship: white, two pixels wide

Time to paint. Watch the trap first: if you call `ctx.stroke()` right now and reload, you'll see a black field and think nothing happened. The code did run — your console log still prints — but you never set `strokeStyle`, so the stroke used the default color: black. Black ship on a black field. The most common first-canvas failure is not an error; it's an invisibility.

Set the style, then stroke. After the `closePath()` line:

```js
ctx.strokeStyle = "#ffffff";
ctx.lineWidth = 2;
ctx.stroke();

console.log("ship at", ship.x, ship.y);
```

Reload: a white triangle, nose up, in the middle of a black field. The entire 1979 universe, first frame.

> [!DESIGN-NOTE]
> **Why `lineWidth = 2` instead of 1?**
>
> A 1-pixel line is centered on the path, so a path sitting on an integer coordinate straddles two pixels, and the browser splits the ink between them — the line comes out soft and gray instead of solid. [MDN's pixel-grid section](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes) works through exactly this; the fix is to offset the path by half a pixel (draw at 400.5 instead of 400). Even-width lines don't have the problem: each half of a 2-pixel line covers whole pixels, so a path on integer coordinates comes out crisp. This game lives or dies on crisp white lines — that's the whole aesthetic — so we take the free win and use width 2 everywhere. If you ever draw a width-1 line and want it crisp, offset its coordinates by 0.5.

The pen metaphor has done its job, so it retires here. From this point on we stop talking about paper and start talking about frames — because a static drawing is about to become an animation.

## Checkpoint

> [!PREDICT]
> Before you open the file: if you delete the `ctx.strokeStyle = "#ffffff";` line and reload, what do you see — and which console lines still print?

**Run this to verify your work so far:**
Your assembled file should match the complete file below. Save it as `game.html` and open it in a browser (double-click is fine — the module imports nothing), or serve the directory and open `http://localhost:8000/game.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Asteroids</title>
  <style>
    body {
      margin: 0;
      min-height: 100vh;
      display: grid;
      place-items: center;
      background: #222222;
    }
    canvas {
      border: 1px solid #555555;
    }
  </style>
</head>
<body>
  <canvas id="game" width="800" height="600">
    Your browser does not support the canvas element — the game needs it.
  </canvas>

  <script type="module">
    const canvas = document.getElementById("game");
    console.log("module ran, canvas is", canvas.width, "x", canvas.height);

    const ctx = canvas.getContext("2d");

    // The playfield: paint the whole 800×600 field black.
    ctx.fillStyle = "#000000";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // The ship's state: its position in the field.
    const ship = { x: 400, y: 300 };

    // The ship: a triangle, nose up.
    ctx.beginPath();
    ctx.moveTo(ship.x, ship.y - 15);      // nose
    ctx.lineTo(ship.x - 12, ship.y + 12); // back left
    ctx.lineTo(ship.x + 12, ship.y + 12); // back right
    ctx.closePath();
    ctx.strokeStyle = "#ffffff";
    ctx.lineWidth = 2;
    ctx.stroke();

    console.log("ship at", ship.x, ship.y);
  </script>
</body>
</html>
```

```bash
# optional, only if you prefer serving over double-clicking:
python3 -m http.server 8000
# then open http://localhost:8000/game.html
```

Expected output:
- A solid black 800×600 field centered on the page, with the gray page around it and a 1px gray border around the field.
- A white triangle, nose up, centered — nose at (400, 285), back corners at (388, 312) and (412, 312).
- In the DevTools console (F12 → Console), two lines:

```
module ran, canvas is 800 x 600
ship at 400 300
```

**Likely errors:**
- If you see `Uncaught TypeError: Cannot read properties of null (reading 'getContext')`, the id in `getElementById("game")` doesn't match the canvas's `id` attribute — one of them has a typo.
- If the ship is an open "V" with no bottom edge, you dropped the `closePath()` line.
- If a diagonal line crosses the field from the ship toward the upper right, you left in the practice triangle from Wrong #2 — delete it, or add the missing `beginPath()`.
- If the console shows `Uncaught SyntaxError` and nothing draws, a typo (missing comma or parenthesis) in one of the drawing calls killed the whole module — modules abort entirely on a syntax error, so nothing after it runs, including the logs. Fix the line the error points at.

## What's next

The ship is parked, and nothing changes until you reload. A triangle on a bitmap is a drawing, not a ship — it has no heading, no velocity, and no way to hear a key press. Part 2's question is how a static drawing becomes a moving object, and the answer is the browser's own animation clock: [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) calls your code before the next repaint, at the display's refresh rate — usually 60 times per second, 120 or 144 on fast monitors — and the ship will learn to rotate, thrust, and drift.

Before you move on: in two sentences, why did we store the ship's position in a `ship` object that the drawing code reads, instead of hard-coding 400 and 300 into `moveTo` and `lineTo`? Write the answer that would satisfy a skeptical colleague who says "nobody is ever going to move that triangle."

## Exercises

- [ ] **Solid ship.** Replace `ctx.stroke()` with `ctx.fill()`, set `ctx.fillStyle = "#ffffff"` just above it, and reload. The triangle looks identical — `fill()` closes open paths automatically, so delete `closePath()` and confirm it still does.
- [ ] **An asteroid.** Draw a closed path of six `lineTo` points around a center at (620, 150), each at a slightly different radius (between 25 and 32) so it looks like a rock rather than a hexagon. White stroke, width 2. Remember `beginPath()` first.
- [ ] **A parameterized ship.** Wrap the ship's path code in `function drawShip(x, y) { … }` and call it twice: `drawShip(400, 300)` and `drawShip(150, 450)`. Two ships, one piece of code. In Part 2, this exact function will redraw the ship at a new position 60 times per second.
- [ ] **Crispness, made visible.** Set `ctx.lineWidth = 1` and zoom the browser to 100%: is the ship's outline softer than at width 2? Then change the ship's position to `{ x: 400.5, y: 300.5 }` and compare. You've just reproduced the half-pixel effect from the design note.
- [ ] **Prove the scope.** In the console, type `ship` and press Enter, then `ctx`. Both should throw a `ReferenceError` — the module's top-level names are not globals. Type `window` to inspect the actual global object and confirm the game's state isn't on it.

## Sources

**Primary docs**

1. [Basic usage of canvas — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Basic_usage) — the `<canvas>` element, its 300×150 defaults, and how `getContext("2d")` hands you a rendering context.
2. [Drawing shapes with canvas — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes) — the coordinate grid, the path dance, the `fill()`/`stroke()` asymmetry, and the pixel-alignment rules.
3. [JavaScript modules — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) — module semantics: automatic deferral, strict mode, script-local scope, and the `file://` local-testing caveat.
4. [Window: requestAnimationFrame() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) — the frame clock Part 2 builds on: the callback's timestamp and refresh-rate behavior.
5. [The canvas element — WHATWG HTML spec](https://html.spec.whatwg.org/multipage/canvas.html) — the canvas as a bitmap, the 300/150 defaults, and one pixel per coordinate unit, at the spec level.

**Deep dives**

6. [ECMAScript modules in browsers — Jake Archibald](https://jakearchibald.com/2017/es-modules-in-browsers/) — the practical detail that inline module scripts are deferred whether or not they import anything.
7. [JavaScript modules and the file URI scheme — Kieran Barker](https://barker.codes/blog/javascript-modules-and-the-file-uri-scheme/) — why browsers treat local files as separate origins, and what that does to `import` from `file://`.

**Background on the game**

8. [Asteroids (video game) — Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) — the 1979 Atari release and the core loop: triangular ship, asteroids that break into smaller, faster, more valuable pieces.
9. [Asteroids — The Strong National Museum of Play](https://www.museumofplay.org/games/asteroids/) — the original five controls: rotate left, rotate right, thrust, fire, hyperspace.
