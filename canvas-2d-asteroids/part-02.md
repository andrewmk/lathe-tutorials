# Asteroids in One HTML File, Part 2: The Loop, the Keys, and the Drift

Part 1 ended with a triangle that cannot change. Reload the page ten times and the ship stays at (400, 300), nose up, because the script runs once and stops. The fix is not a faster script. It is a different way of running the script: the browser already has a clock that ticks once per screen refresh, and it will call your code before every repaint. Part 2 builds the loop on that clock, teaches the ship to hear keyboard input, and gives it the two numbers a moving object needs — a heading and a velocity.

## What you'll build

The end state is the same `game.html` from Part 1, with its one-shot drawing replaced by a loop. In the browser you can:

- Hold **ArrowLeft** / **ArrowRight** and the ship rotates in place at 3 radians per second.
- Hold **Space** and the ship accelerates in the direction its nose points, at 120 pixels per second per second.
- Let go and the ship keeps coasting at whatever velocity it has. There is no friction and no brake in this part — the drift is the point.

Three things are deliberately *not* here yet. The ship does not wrap around the screen edges (Part 3's job, and an exercise if you can't wait). It cannot fire, and there are no asteroids, no bullets, and no collisions (Parts 3 and 4). And the loop runs even when nothing moves, because a game loop is a clock, not an event.

## Prerequisites

- Part 1, completed: a `game.html` whose module script paints the black field and the static ship, with the console logs `module ran, canvas is 800 x 600` and `ship at 400 300`.
- The same browser as before. No new tools, no installation.

## The frame clock

Part 1 retired the pen and promised frames. Here is the frame, in the form the browser sells it: a method called `requestAnimationFrame`, in [MDN's words](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame), it "requests the browser to call a user-supplied callback function before the next repaint." You hand the browser a function; before the browser paints the next screen image, it calls that function. That is the entire contract.

Three properties of the contract matter for everything that follows:

1. **It is one-shot.** MDN: "Your callback function must call `requestAnimationFrame()` again if you want to animate another frame. `requestAnimationFrame()` is one-shot." One call buys one callback. The loop is *you*, calling it again from inside.
2. **It runs at the display's refresh rate.** MDN: "The frequency of calls to the callback function will generally match the display refresh rate. The most common refresh rate is 60hz, (60 cycles/frames per second), though 75hz, 120hz, and 144hz are also widely used." On a 120 Hz monitor your code runs twice as often as on a 60 Hz one. Remember that; it is the whole reason for the next section.
3. **It pauses when the page can't be seen.** MDN: "`requestAnimationFrame()` calls are paused in most browsers when running in background tabs or hidden `<iframe>`s, in order to improve performance and battery life." A tab you switch away from stops receiving frames, and a large chunk of game bugs comes from not believing that.

The [spec](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html) states the callback type in one line: `FrameRequestCallback = undefined(DOMHighResTimeStamp time)` — your function takes exactly one argument, a timestamp. And the callback runs once per request: the spec's algorithm removes the callback from the browser's map and then invokes it — "Remove callbacks[handle]. Invoke callback with « now »."

Now restructure the script. Part 1's file drew the field and the ship once, top to bottom. Replace everything from the `// The playfield:` comment to the end of the script with this:

```js
    // The ship's state: its position in the field.
    const ship = { x: 400, y: 300 };

    // One full repaint of the field: black background, then the ship.
    function draw() {
      ctx.fillStyle = "#000000";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.beginPath();
      ctx.moveTo(ship.x, ship.y - 15);      // nose
      ctx.lineTo(ship.x - 12, ship.y + 12); // back left
      ctx.lineTo(ship.x + 12, ship.y + 12); // back right
      ctx.closePath();
      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    // The frame callback: repaint, then ask for the next frame.
    function frame(t) {
      draw();
      requestAnimationFrame(frame);
    }

    requestAnimationFrame(frame);
```

Reload. The page looks exactly like the end of Part 1 — same black field, same ship — but the script is now running continuously: `frame` repaints the whole field, asks for the next frame, and the browser calls it again before the next repaint. That self-scheduling tail is the loop.

> [!DESIGN-NOTE]
> **Why not `setInterval`?**
>
> A 16-millisecond timer would fire roughly 60 times per second, which is close — but it has no relationship to when the browser actually paints. You can paint, then have the browser paint again before your next tick, and the two images race. `requestAnimationFrame` schedules your callback *before the next repaint*, so your drawing is guaranteed to be part of the frame the browser shows. For anything that draws, the rAF clock is the one that is synchronized with the screen; the timer is not.

One more fact about the loop: the method returns a request id you can pass to `cancelAnimationFrame()` to cancel a frame you asked for. The [spec's algorithm](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html) for `cancelAnimationFrame(handle)` is a single step — "Remove callbacks[handle]" — and the callback never runs. The game will not need it (the loop runs until the page closes), but you will meet it in the wild.

## The timestamp, and dt

Look at the callback's signature again: `function frame(t)`. The `t` is not optional decoration. It is a [DOMHighResTimeStamp](https://developer.mozilla.org/en-US/docs/Web/API/DOMHighResTimeStamp) — "a double and is used to store a time value in milliseconds," usable for "a discrete point in time or a time interval (the difference in time between two discrete points in time)." In practice: milliseconds since the page's time origin, so `t` on your first frame is a small number like 12.5, and each later frame's `t` is larger than the last.

The difference between consecutive timestamps is the elapsed time since the previous frame — the game's clock. Call it `dt` ("delta t"), in seconds:

```js
    let last = null;

    function frame(t) {
      if (last === null) {
        console.log("first frame at", t.toFixed(1), "ms");
      }
      const dt = last === null ? 0 : Math.min((t - last) / 1000, 0.1);
      last = t;

      draw();
      requestAnimationFrame(frame);
    }
```

Three decisions are packed into that `dt` line, and all three will matter before the end of this part:

- **`t - last`, not `t`.** You want the time *between frames*, not the time since the page loaded. On a 60 Hz display that is about 16.7 milliseconds, or 0.0167 seconds.
- **Divide by 1000.** The timestamp is in milliseconds; the physics below is written in per-second units.
- **Clamp with `Math.min(…, 0.1)`.** Recall that rAF pauses in background tabs. Switch to another tab for five seconds and come back: `t - last` is now 5000 milliseconds, and without the clamp one frame would try to simulate five seconds of motion — the ship would teleport across the field. The clamp says: no single frame may account for more than 0.1 seconds of game time. After a pause the game is slightly behind real time; that is a fine price for not teleporting.

And the first frame gets `dt = 0`: there is no previous frame, so no time has elapsed. `update(0)` below does nothing, which is exactly right.

Reload: the console now shows a third line,

```
first frame at 12.5 ms
```

(the number varies from load to load — it is the gap between the page's time origin and the first frame). The game still looks static, because `dt` is not being spent on anything yet. It is the difference between the next two sections: with `dt`, the game's speed is set in *real* units — pixels per second, radians per second — and the ship moves the same distance per real second whether the display is 60 Hz or 144 Hz.

> [!HEADS-UP]
> **This is where games that aren't frame-rate independent go wrong.** MDN's warning for `requestAnimationFrame` is blunt: "Be sure always to use the first argument (or some other method for getting the current time) to calculate how much the animation will progress in a frame — **otherwise, the animation will run faster on high refresh-rate screens**." A loop that moves the ship 2 pixels *per frame* moves 120 pixels per second at 60 Hz but 288 pixels per second at 144 Hz — the game runs 2.4 times faster on the nicer monitor. Multiplying by `dt` is what converts "per frame" into "per second."

## The keys: a table, not a list of events

The browser reports keyboard input as events: in [MDN's table](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent), `keydown` means "A key has been pressed" and `keyup` means "A key has been released." A naive game reacts to each `keydown`: press ArrowRight, rotate a little. That feels wrong the moment you hold the key, because holding a key is not one event. Per MDN: "When a key is pressed and held down, it begins to auto-repeat," dispatching `keydown` over and over, "repeating until the user releases the key."

So a game does not want a *log* of key presses. It wants the answer to one question, asked 60 times a second: **which keys are held down right now?** That question has a table-shaped answer:

```js
    // Keyboard state: which keys are held down right now.
    const keys = {};
    const GAME_KEYS = ["ArrowLeft", "ArrowRight", "Space"];

    addEventListener("keydown", (e) => {
      keys[e.code] = true;
      if (GAME_KEYS.includes(e.code)) e.preventDefault();
    });
    addEventListener("keyup", (e) => {
      keys[e.code] = false;
    });
```

Add this after the `ship` line. The table starts empty. A `keydown` marks the key as held; a `keyup` unmarks it. Auto-repeat is now harmless: the repeated `keydown`s just set `true` on a slot that is already `true`. And the game loop never listens for events — it reads the table.

Three details in that block earn their keep:

**`e.code`, not `e.key`.** A keyboard event carries both the *key* (the character the press produces) and the *code* (the physical key pressed). [MDN](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/code): "`KeyboardEvent.code` … represents a physical key on the keyboard (as opposed to the character generated by pressing the key)" and "isn't altered by keyboard layout or the state of the modifier keys." MDN singles out games by name: handling keys by physical position is "especially common when writing code to handle input for games that simulate a gamepad-like environment using keys on the keyboard." A gamepad-like environment is exactly what arrow keys plus Space are. Use `e.code` and the controls work on a French AZERTY keyboard, on a German QWERTZ, and with Caps Lock on.

**The code values are strings you can read.** For the three keys we use, [MDN's code table](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_code_values) lists `"ArrowLeft"`, `"ArrowRight"`, and `"Space"` — the same on Windows, Linux, and macOS. That is why the table is keyed `keys[e.code]` and the game later reads `keys.ArrowLeft` and `keys.Space` as if they were properties.

**`preventDefault()` stops the page from stealing the keys.** Space and the arrow keys are also the browser's page-scrolling keys. If the page can scroll, holding Space would scroll it. The listener marks only the three game keys (`GAME_KEYS.includes(e.code)`) and cancels their default action — so the page scrolls normally with any other key, and the game keys are the game's.

Nothing is visible yet: the table fills up, but no code reads it. Reload and hold ArrowRight — the ship does not turn, but if you open the console and type `keys` you will get a `ReferenceError` (module scope, as Part 1's exercise showed). The table is real; the console just can't see inside the module. You'll get a peek in the exercises.

## Heading, velocity, and drift

A moving object needs two more numbers than a parked one. Expand the ship's state — replace the one-line `ship` declaration with:

```js
    const ship = {
      x: 400,
      y: 300,
      vx: 0,
      vy: 0,
      angle: 0 // 0 points up; positive rotates clockwise
    };
```

`vx` and `vy` are velocity in pixels per second — `vy` positive means *down*, because the canvas's y-axis points down. `angle` is the heading, in radians, with a convention worth fixing now: **`angle = 0` means the nose points up, and a positive angle turns the ship clockwise.** That convention is not ours to invent — the canvas's own rotation is defined clockwise. [MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/rotate): the angle is "clockwise in radians. You can use `degree * Math.PI / 180` to calculate a radian from a degree." Canvas coordinates have y pointing down, so "clockwise" on screen is "positive angle" in the math, and the ship's heading can just be the angle we'll pass to `ctx.rotate()` later. No conversion, ever.

Then the simulation step. Add after the `keys` block:

```js
    // Tuning constants, in per-second units.
    const ROT_SPEED = 3; // radians of turn per second
    const THRUST = 120;  // pixels per second, gained per second

    // One step of the simulation, dt seconds of game time.
    function update(dt) {
      if (keys.ArrowLeft)  ship.angle -= ROT_SPEED * dt;
      if (keys.ArrowRight) ship.angle += ROT_SPEED * dt;

      if (keys.Space) {
        ship.vx += Math.sin(ship.angle) * THRUST * dt;
        ship.vy += -Math.cos(ship.angle) * THRUST * dt;
      }

      ship.x += ship.vx * dt;
      ship.y += ship.vy * dt;
    }
```

and call it from the frame callback, before the draw:

```js
    function frame(t) {
      if (last === null) {
        console.log("first frame at", t.toFixed(1), "ms");
      }
      const dt = last === null ? 0 : Math.min((t - last) / 1000, 0.1);
      last = t;

      update(dt);
      draw();
      requestAnimationFrame(frame);
    }
```

Walk the three stanzas.

**Rotation.** Holding ArrowRight adds `ROT_SPEED * dt` to the angle every frame. At 60 Hz that is `3 × 0.0167 ≈ 0.05` radians per frame, which sums to 3 radians per second regardless of the frame count — a full turn (2π ≈ 6.28 radians) in about 2.1 seconds. Left subtracts.

**Thrust.** The nose's direction is a unit vector, and it comes from the angle. At `angle = 0` the nose points up, which in canvas coordinates is `(0, -1)`. Rotate that vector clockwise by `angle` and it becomes:

```
nose direction = ( sin(angle), -cos(angle) )
```

Check the four compass points: at 0 it is `(0, -1)` — up; at π/2 it is `(1, 0)` — right; at π it is `(0, 1)` — down; at −π/2 it is `(−1, 0)` — left. So thrust adds `THRUST * dt` of velocity *in the nose direction*, per frame. Hold Space for one second and the ship gains 120 pixels per second of velocity, aimed exactly where the nose pointed while you held it.

**Movement.** `x += vx * dt` is the rest of Newton's first law: position changes by velocity × time, and nothing in the function touches `vx` and `vy` except thrust. There is no friction term, no brake, no maximum speed. Whatever velocity the ship has, it keeps.

Reload and hold Space. The ship flies upward — and here is the first visible crack in the illusion: the triangle still points up no matter what, because `draw()` does not know about `angle` yet. Rotate the ship with ArrowRight while holding Space and the ship flies *straight up* while pointing right. The state is right and the drawing is wrong. That is what the next section fixes.

One deliberate difference from the arcade original, so nobody goes looking for a bug: in the 1979 game, [per Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)), "The ship eventually loses momentum and comes to a stop when not thrusting." Ours never does. That is a design choice, not an oversight — frictionless drift is the simplest physics that still plays like Asteroids, it is what makes the drift in this part's title real, and the original's drag is a one-line change you can add as an exercise below. The point to hold onto: the ship in your game moves because *you* keep its velocity, frame after frame. Nothing in the browser slows it down.

## Draw the ship where it is

The drawing problem is that `draw()` computes rotated coordinates by hand: the nose at `ship.y - 15` assumes the ship is unrotated. You *could* compute each corner's rotated position with sin and cos in JavaScript — it is three points, after all — but the canvas has a tool for exactly this, and using it means the ship's path stays written in its own local coordinates, nose at `(0, -15)`, for the rest of the series.

The tool is the **transform**: instead of moving the ship, you move the coordinate system. Three methods, per [MDN's transforms tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations):

- `ctx.translate(x, y)` — "Moves the canvas and its origin on the grid." The origin — the point every subsequent drawing call measures from — moves to `(x, y)`.
- `ctx.rotate(angle)` — "Rotates the canvas clockwise around the current origin by the angle number of radians. The rotation center point is always the canvas origin."
- `ctx.save()` / `ctx.restore()` — "Saves the entire state of the canvas" / "Restores the most recently saved canvas state. Canvas states are stored on a stack."

The second sentence of `rotate`'s description is the load-bearing one: rotation always happens around *the current origin*. So to spin the ship around its own center, you first move the origin to the ship's center with `translate`, *then* rotate. The order is not optional — `translate` after `rotate` would rotate the origin around the canvas's top-left corner first, and the ship would orbit the corner of the field like a planet around the wrong star.

Replace the ship's path in `draw()` with a save / translate / rotate / draw / restore sequence:

```js
    function draw() {
      ctx.fillStyle = "#000000";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Move and rotate the coordinate system, then draw the ship
      // around the origin in its own local coordinates.
      ctx.save();
      ctx.translate(ship.x, ship.y);
      ctx.rotate(ship.angle);
      ctx.beginPath();
      ctx.moveTo(0, -15);      // nose
      ctx.lineTo(-12, 12);     // back left
      ctx.lineTo(12, 12);      // back right
      ctx.closePath();
      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.restore();
    }
```

Read it as a change of frame of reference. `translate(ship.x, ship.y)` puts the origin at the ship's center. `rotate(ship.angle)` spins everything that follows around that point. The triangle is now drawn *around the origin* — nose at `(0, -15)`, back corners at `(±12, 12)` — the exact same three offsets Part 1 used around (400, 300), just recentered. The transform does the arithmetic Part 1 did by hand: `moveTo(ship.x, ship.y - 15)` became `translate(ship.x, ship.y)` plus `moveTo(0, -15)`.

The `save()`/`restore()` pair is the part that looks like ceremony and isn't. Transforms are *state* on the context, the same as the fill color. Without `restore()`, the next frame's `translate` and `rotate` stack on top of this frame's: frame two moves the origin to (400, 300) *as seen from a rotated, shifted frame one*, and the ship's apparent heading compounds 60 times a second into a spinning blur. `save()` pushes the current state onto a stack, `restore()` pops it, and each frame starts from the same clean, unrotated grid. The discipline for the rest of the series: every `save()` gets a matching `restore()`, and everything drawn between them may assume the transform.

Reload. Now the ship does what the state says: ArrowLeft and ArrowRight spin it in place, Space fires it off in the nose direction, and when you let go it coasts — still pointing where it was aimed, still moving, no brake in sight. Aim it, thrust, and watch it drift across the field at a steady speed. That is the Asteroids feel, and all of it is the four lines in `update` plus the transform in `draw`.

## Checkpoint

> [!PREDICT]
> Point the ship straight up. Hold Space for exactly one second, then let go. What is the ship's speed when you release? (Answer: 120 pixels per second — 1 second × 120 px/s².) And from the moment you release, how long until the ship's *center* reaches the top edge of the field? (It starts 300 pixels from the edge, at y = 300 — but during that one second of thrust it already covered ½ × 120 × 1² = 60 pixels, so when you let go it is 240 pixels from the edge, moving at 120 px/s: exactly 2.0 seconds.) What happens after that? (Nothing, in this part: the ship leaves the field and stays gone. There is no wrap-around until Part 3, and no way back but reloading.)

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

    // The ship's state: position, velocity, and heading.
    const ship = {
      x: 400,
      y: 300,
      vx: 0,
      vy: 0,
      angle: 0 // 0 points up; positive rotates clockwise
    };

    // Keyboard state: which keys are held down right now.
    const keys = {};
    const GAME_KEYS = ["ArrowLeft", "ArrowRight", "Space"];

    addEventListener("keydown", (e) => {
      keys[e.code] = true;
      if (GAME_KEYS.includes(e.code)) e.preventDefault();
    });
    addEventListener("keyup", (e) => {
      keys[e.code] = false;
    });

    // Tuning constants, in per-second units.
    const ROT_SPEED = 3; // radians of turn per second
    const THRUST = 120;  // pixels per second, gained per second

    // One step of the simulation, dt seconds of game time.
    function update(dt) {
      if (keys.ArrowLeft)  ship.angle -= ROT_SPEED * dt;
      if (keys.ArrowRight) ship.angle += ROT_SPEED * dt;

      if (keys.Space) {
        ship.vx += Math.sin(ship.angle) * THRUST * dt;
        ship.vy += -Math.cos(ship.angle) * THRUST * dt;
      }

      ship.x += ship.vx * dt;
      ship.y += ship.vy * dt;
    }

    // One full repaint of the field: black background, then the ship.
    function draw() {
      ctx.fillStyle = "#000000";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Move and rotate the coordinate system, then draw the ship
      // around the origin in its own local coordinates.
      ctx.save();
      ctx.translate(ship.x, ship.y);
      ctx.rotate(ship.angle);
      ctx.beginPath();
      ctx.moveTo(0, -15);      // nose
      ctx.lineTo(-12, 12);     // back left
      ctx.lineTo(12, 12);      // back right
      ctx.closePath();
      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.restore();
    }

    let last = null;

    // The frame callback: step the simulation, repaint, ask for the next frame.
    function frame(t) {
      if (last === null) {
        console.log("first frame at", t.toFixed(1), "ms");
      }
      const dt = last === null ? 0 : Math.min((t - last) / 1000, 0.1);
      last = t;

      update(dt);
      draw();
      requestAnimationFrame(frame);
    }

    requestAnimationFrame(frame);
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
- The same page as Part 1 — black 800×600 field, gray border, gray page — but the ship is now an object.
- In the DevTools console (F12 → Console), two lines on load:

```
module ran, canvas is 800 x 600
first frame at 12.5 ms
```

(the number on the second line varies from load to load).
- Hold **ArrowRight**: the ship spins clockwise, one full turn in about 2.1 seconds.
- Point it up, hold **Space** for one second, release: the ship moves upward at a steady 120 px/s, and its center crosses the top edge about 2 seconds after you let go (3 seconds from the moment you pressed it).
- Let the ship leave the field: it does not come back. That is expected.

**Likely errors:**
- **The ship orbits the top-left corner of the field.** In `draw()`, the `rotate` call comes before the `translate` call. Rotation happens around the *current* origin; move the origin to the ship first, then rotate.
- **The ship blurs or spirals into a spinning smear.** The `restore()` is missing, so each frame's transform stacks on the last frame's. Every `save()` needs a matching `restore()`.
- **The ship teleports across the field when you switch back to the tab.** You deleted the `Math.min(…, 0.1)` clamp. After a background pause, the first frame's `dt` is seconds long; the clamp caps it at 0.1.
- **Arrows rotate but Space does nothing** (or the page scrolls while thrusting). You are probably checking `keys[" "]` or `e.key` somewhere — the Space bar's *key* value is a single space character, but its *code* is `"Space"`. Everything in this part goes through `e.code`.
- **Nothing happens when you press keys, and the page scrolls.** The listeners are on `window` (`addEventListener` with no element), so the page doesn't need focus on the canvas itself — but if you rewrote them as `canvas.addEventListener(…)`, the canvas has to be focused first; click it, and the keys work. The simplest fix is to keep the listeners on `window`, as written.
- **The console shows `Uncaught SyntaxError` and nothing draws.** Same as Part 1: a module aborts on a syntax error, so the loop never starts and the field is a blank black box. Fix the line the error points at.

## What's next

The ship now has the 1979 feel — rotate, thrust, drift — but it is the only object in the field, and it can leave and never return. Part 3 makes the field a place with rules in it: asteroids spawn and drift across the screen (with wrap-around, which the original has: an asteroid "drifts off the top edge of the screen" and "reappears at the bottom," per [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))), bullets fly straight, and something has to decide when a bullet meets a rock. That decision is collision detection — the first time this game needs real geometry, not just drawing.

Before you move on: the ship's velocity is stored as two numbers, `vx` and `vy`. In one or two sentences, say what each of them means, and why the code stores them separately instead of one speed with one direction.

## Exercises

- [ ] **Feel the constants.** Set `THRUST = 240`. After one second of thrust the ship moves at 240 px/s, and it has already covered ½ × 240 × 1² = 120 pixels — so the top edge takes about 0.75 seconds after you release, not the 2.0 seconds of the default. Then set `ROT_SPEED = 1.5`: a full turn should now take about 4.2 seconds (2π ÷ 1.5). Restore the original values when you're done.
- [ ] **The original's drag.** Wikipedia: in the 1979 game, "The ship eventually loses momentum and comes to a stop when not thrusting." Add a constant `const DRAG = 0.3;` and, at the end of `update`, bleed off velocity proportionally to what is there: `ship.vx -= ship.vx * DRAG * dt; ship.vy -= ship.vy * DRAG * dt;` With DRAG = 0.3 the ship keeps about three quarters of its speed each second and settles down to a stop. That is the original's feel — compare it to the frictionless default, which this series keeps on purpose.
- [ ] **Wrap-around, early.** The ship leaving and staying gone is Part 3's problem, but you can fix it now with four lines at the end of `update`: if `ship.x < 0`, add `canvas.width`; if `ship.x > canvas.width`, subtract it; same for `ship.y` with `canvas.height`. The ship reappears on the far edge — it "continues moving in the same direction," as the original's asteroids do. Part 3 will do this for every object, with the drawing complications that brings.
- [ ] **Read the speed.** You can't type `ship` in the console — module scope. Instead, instrument it: in the `keyup` listener, after clearing the key, add `if (e.code === "Space") console.log("speed", Math.hypot(ship.vx, ship.vy).toFixed(0));` (`Math.hypot` is the built-in for √(x² + y²).) Hold Space for one second, release: the console should print a speed of about 120. Hold it two seconds: about 240.
- [ ] **Prove the dt.** In Chrome's DevTools, open More tools → Rendering, enable the "Frame Rate" option, and set it to 30 fps. Thrust for one second and release: the ship should still reach the top edge about 2 seconds after you let go — same as at 60 fps, because `dt` is 0.033 now instead of 0.0167 and the product `speed × dt` per frame is the same. Then, just to see the failure mode, temporarily move the ship a fixed amount per frame — replace `ship.x += ship.vx * dt;` with `ship.x += ship.vx / 60;` — and watch the game run at half speed on the 30 fps throttle. That is MDN's warning, demonstrated.

## Sources

**The frame clock**

1. [Window: requestAnimationFrame() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) — the callback runs "before the next repaint"; one-shot semantics; refresh-rate frequency (60 Hz most common, up to 144 Hz); background-tab pausing; the timestamp argument; and the warning that without the timestamp, "the animation will run faster on high refresh-rate screens."
2. [DOMHighResTimeStamp — MDN](https://developer.mozilla.org/en-US/docs/Web/API/DOMHighResTimeStamp) — the timestamp type: a double storing "a time value in milliseconds," for a point in time or an interval.
3. [Image bitmaps and animations — WHATWG HTML spec](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html) — the `FrameRequestCallback = undefined(DOMHighResTimeStamp time)` signature, the one-invocation-per-request algorithm, and `cancelAnimationFrame`'s single "Remove callbacks[handle]" step.

**Drawing in motion**

4. [Transformations — MDN Canvas API tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations) — `translate` moves "the canvas and its origin on the grid"; `rotate` turns "clockwise around the current origin"; `save()`/`restore()` and the state stack.
5. [CanvasRenderingContext2D.rotate() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/rotate) — the angle parameter: "clockwise in radians," with the `degree * Math.PI / 180` conversion.

**Keyboard input**

6. [KeyboardEvent — MDN](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent) — `keydown` ("A key has been pressed."), `keyup` ("A key has been released."), and the auto-repeat sequence that repeats `keydown` until the key is released.
7. [KeyboardEvent.code — MDN](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/code) — the physical-key identifier, "not altered by keyboard layout or the state of the modifier keys," and why games prefer it.
8. [KeyboardEvent: code values — MDN](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_code_values) — the code strings used in this part: `"ArrowLeft"`, `"ArrowRight"`, `"Space"` (0x0039).

**The original game**

9. [Asteroids (video game) — Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) — the ship's controls (rotate, fire, thrust), the original's momentum decay ("eventually loses momentum and comes to a stop"), and screen-edge wrap-around.
