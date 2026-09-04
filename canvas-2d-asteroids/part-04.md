# Asteroids in One HTML File, Part 4: The Whole Game — Score, Waves, Lives, and the Saucer

> [!RECALL]
> Part 3's `wrap()` is two lines: `v %= size;` and `return v < 0 ? v + size : v;`. Why does the second line exist? What does `-805 % 800` evaluate to in JavaScript, and what should `wrap(-805, 800)` return? (If you can't reconstruct it: `%` takes the sign of its left operand, so the remainder comes out negative, and the second line adds one full size back. Part 3's "Wrap-around: the field is a torus" walks it through.)

Part 3 ended with a field full of rules — rocks that split, bullets that die, a ship that dies and comes back — but no purpose. Nothing measures how well you're doing, clearing the field changes nothing, and dying costs nothing. This part gives the field a purpose: a score that counts what rocks pay, waves that keep the field from staying empty, lives that make death cost something, and the flying saucer — the last object from the original, and the first in this game that aims at anything. When you're done, the file is a game.

## What you'll build

Same file, same ship, same rocks. What gets added:

- **Score** — top-left, counting what rocks pay (20/50/100 for large/medium/small, the original's values), plus a spare life every 10,000 points.
- **Waves** — clear the field and the next level spawns, with more large rocks each time (4 on level 1, up to 8).
- **Lives** — three ships, with the spares drawn as small triangles in the bottom-left. Lose the last one and the game freezes on a GAME OVER screen; Enter starts a new game.
- **The saucer** — appears on a 20–40 second timer, crosses the field, and fires at the ship. Shoot it for 1,000 points.

Everything else — the loop, the ship, the rocks, the bullets, the torus — is unchanged.

## Prerequisites

Part 3, and its assembled `game.html`: rocks, bullets, wrap-around, splitting, and respawn. If your ship doesn't blink after it dies, go back before starting here.

## The score: a scoreboard

The original's scoring, from the 1979 cabinet ([Arcade History](https://www.arcade-history.com/game/126/)):

| Object | Points |
|---|---|
| Large asteroid | 20 |
| Medium asteroid | 50 |
| Small asteroid | 100 |
| Large flying saucer | 200 |
| Small flying saucer | 1,000 |

Wikipedia's summary is one line: "Smaller asteroids are also worth more points." This game uses the table's values as-is, including the 1,000 the saucer pays later.

### The scoreboard's variables

Add the game's scoreboard after the field block (the `asteroids` and `bullets` arrays):

```js
    // The game's scoreboard.
    let score = 0;
    let lives = 3; // the original starts the player with 3-5; this one takes 3
    let level = 1; // which wave is on the field
    let nextLifeAt = 10000; // score at which the next extra life is earned
```

Four numbers, each of which earns its place:

- `score` is the count.
- `lives` is how many ships the player still has. The original: "The player starts with 3–5 lives upon game start and gains an extra life per 10,000 points." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) Three is a choice — the sources give a range, and 3 makes a run last about as long as an arcade visit.
- `level` is which wave; the next section uses it.
- `nextLifeAt` exists because the extra life is a rule about *crossing* a threshold, not reaching a number. If you checked `score >= 10000` and added a life without remembering that the threshold had already been crossed, the next rock you broke would add another life, and the one after that. `nextLifeAt` is that memory, and it moves up by 10,000 after each award.

### Paying out

The score goes up where rocks die — `split`. Replace it:

```js
    // Replace the rock at index with two smaller, faster ones (or nothing).
    function split(index) {
      const a = asteroids[index];
      asteroids.splice(index, 1);
      addScore(SCORES[a.size]);
      if (a.size > 0) {
        for (let i = 0; i < 2; i++) {
          asteroids.push(makeAsteroid(a.x, a.y, a.size - 1));
        }
      }
    }
```

and add what it uses. First, the table, next to the other size tables (`RADII`, `SIDES`, `SPEEDS`):

```js
    // What each size pays, index 0 = small.
    const SCORES = [100, 50, 20];
```

The index is the rock's `size` field — 0 is small, so small comes first, and `SCORES[a.size]` is 100 for a small, 50 for a medium, 20 for a large. Then `addScore`, above `split`:

```js
    function addScore(points) {
      score += points;
      if (score >= nextLifeAt) {
        lives++;
        nextLifeAt += 10000;
      }
    }
```

`addScore` is the one place the score changes. The saucer's 1,000 goes through it too, so saucer money counts for the extra life the same way rock money does.

### Drawing it

Two things the canvas's text needs to know:

- `fillText(text, x, y)` renders a string "using the settings specified by font, textAlign, textBaseline, and direction" ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/fillText)) — and `y` is "the y-axis coordinate of the baseline on which to begin drawing the text." The text sits *above* its coordinate, so `fillText("SCORE 0", 10, 24)` draws just above y = 24.
- `ctx.font` is "a string parsed as CSS font value," and the default is `10px sans-serif` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/font)). Ten pixels is nearly invisible on an 800-pixel field, so set it before you draw.

At the end of `draw()`, after the ship block:

```js
      // The scoreboard: score and level, top-left.
      ctx.fillStyle = "#ffffff";
      ctx.font = "16px monospace";
      ctx.fillText(`SCORE ${score}`, 10, 24);
      ctx.fillText(`LEVEL ${level}`, 10, 44);
```

The backticks are template literals — a string in backticks where `${expression}` is replaced by the expression's value. `SCORE ${score}` becomes `SCORE 0` on load and `SCORE 2700` mid-game.

Reload. The field is unchanged; two lines of text sit in the top-left and climb every time you break a rock. Crossing 10,000 takes a while — one fully cleared wave is 2,080 points (4×20 + 8×50 + 16×100) — so trust the `nextLifeAt` line for now and verify it in the exercises.

## Waves: the field refills itself

Last part, clearing the field "changes nothing". Change that with three lines, at the very end of `update`, after the collision passes. (If you did Part 3's "next wave" exercise, this is what that one line grows into.)

```js
      // Wave: the field is cleared, so the next level starts.
      if (asteroids.length === 0) {
        level++;
        spawnAsteroids(Math.min(3 + level, 8), 2);
      }
```

The original: "Once the screen has been cleared of all asteroids and flying saucers, a new set of large asteroids appears, thus starting the next level." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) One deliberate difference: the original's trigger includes the saucers, because its saucers *leave* — they fly in from one edge and out the other. Ours wraps, so it never leaves, and a trigger that waited for it could stall a wave for the saucer's whole 20–40 second patrol. In this game the rocks are the trigger.

The count is a design choice, pointed the way the original points it: "The game gets harder as the number of asteroids increases until after the score reaches a range between 40,000 and 60,000." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) `3 + level` gives 4 on level 1 — exactly Part 3's field — then 5, 6, 7, 8, and `Math.min` holds it there. (An 8-large wave can grow to 56 rocks at full split — 8 × 7 — so Part 3's 26-rock cap exercise is more worth doing than ever.)

## Lives: what a ship costs

Right now death is free: `respawn()` runs unconditionally, and the console line is the entire price. Change that.

First, one more line for the scoreboard block:

```js
    let over = false; // game over: the field is frozen until Enter
```

Then make death cost a life. The ship-vs-asteroid pass becomes:

```js
      // Collisions: ship vs. asteroid, only when not invincible.
      if (ship.invincible <= 0) {
        for (const a of asteroids) {
          if (hit(ship.x, ship.y, SHIP_RADIUS, a.x, a.y, a.radius)) {
            killShip();
            break;
          }
        }
      }
```

The death itself is a function, because two things will use it — rocks now, the saucer's bullets in the next section. Add it after `respawn`:

```js
    function killShip() {
      lives--;
      if (lives <= 0) {
        over = true;
        console.log("game over — final score", score);
      } else {
        respawn();
      }
    }
```

`killShip` spends a life, then either respawns or ends the run. `respawn` itself is unchanged — it still resets the ship and blinks it through the traffic.

### The spare ships

Spare ships need a display. The original draws its lives as little ship icons — Wikipedia's quirk list proves it, complaining that "The game's code continues trying to draw them even if they fall outside the boundaries of the screen." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) Ours go in the bottom-left, as small triangles — the ship's own shape at half size. In `draw()`, after the score text:

```js
      // Spare ships, bottom-left: the ship's shape at half scale.
      for (let i = 0; i < Math.min(lives - 1, 9); i++) {
        ctx.save();
        ctx.translate(24 + i * 24, canvas.height - 18);
        ctx.scale(0.5, 0.5);
        ctx.beginPath();
        ctx.moveTo(0, -15);
        ctx.lineTo(-12, 12);
        ctx.lineTo(12, 12);
        ctx.closePath();
        ctx.stroke();
        ctx.restore();
      }
```

`scale(x, y)` "scales the canvas units by x horizontally and by y vertically" ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations)) — so the ship's own three points draw at half size, with no new coordinates to invent. `save`/`restore` around it, as always: the next frame's real ship has to draw at full size.

Two choices in that loop. `lives - 1`: the ship on the field is one of the lives, and the row is the spares — 3 lives means 2 triangles. And the cap at 9: a Wikipedia quirk — "Asteroids slows down as the player gains 50–100 lives, because there is no limit to the number of lives displayed." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) The original literally drew them all and crawled. Ours stops drawing at 9 and keeps counting.

### The game-over screen

`over` is a frozen state, and two things honor it.

First, `update` stops simulating — but only after checking for a restart. Add at the top of `update`, above the `time += dt` line:

```js
      if (over) {
        if (keys.Enter) resetGame();
        return;
      }
```

The early return is the freeze: while `over` is true, no rock moves, no bullet flies, and the saucer (next section) can't fire. The restart check sits inside the block, *before* the return — the only order that works, since nothing after a return ever runs.

Second, `draw` shows the state. At the very end of `draw()`, after the spare ships:

```js
      // Game over: the frozen field behind a restart prompt.
      if (over) {
        ctx.font = "32px monospace";
        ctx.fillText("GAME OVER", canvas.width / 2 - 100, canvas.height / 2 - 10);
        ctx.font = "16px monospace";
        ctx.fillText(`FINAL SCORE ${score}`, canvas.width / 2 - 70, canvas.height / 2 + 20);
        ctx.fillText("PRESS ENTER TO RESTART", canvas.width / 2 - 110, canvas.height / 2 + 50);
      }
```

The x positions are hand-centered: no center alignment is set on `fillText` here, so each line starts a fixed offset left of middle — 100 px for the 32-px line, 110 px for the longest 16-px line. Good enough for a monospace scoreboard; the canvas can center for real (`ctx.textAlign = "center"`), but that's a third text setting this part doesn't need.

The restart itself, added after `killShip`:

```js
    function resetGame() {
      score = 0;
      lives = 3;
      level = 1;
      nextLifeAt = 10000;
      over = false;
      bullets.length = 0;
      hostileBullets.length = 0;
      asteroids.length = 0;
      saucer = null;
      saucerTimer = 20;
      ship.x = 400;
      ship.y = 300;
      ship.vx = 0;
      ship.vy = 0;
      ship.invincible = 2;
      spawnAsteroids(4, 2);
    }
```

Every variable a run owns goes back to its starting value, and the ship comes back at center with a blink — the same protection as a mid-game respawn. The ship's position is set *before* `spawnAsteroids`, so the new rocks' 150-px rule keeps clear of where the ship actually is, not where it died. And it doesn't call `respawn()`: that function's console line says "ship hit," and a fresh game is not a ship hit.

Three names in that block don't exist yet — `hostileBullets`, `saucer`, `saucerTimer`: the saucer's state. Declare them now, between the field block and the scoreboard, so `resetGame` has something to clear:

```js
    // The saucer's state: null until one enters the field.
    let saucer = null;
    let saucerTimer = 20; // seconds until the next saucer
    const hostileBullets = []; // the saucer's shots: they hit only the ship
```

The next section fills in what they do. For now they sit idle: `saucer` stays `null`, the timer never matters, and the hostile array never gets a bullet.

## The saucer

The original has two saucers ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))): "Two flying saucers appear periodically on the screen; the 'big saucer' shoots randomly and poorly, while the 'small saucer' fires frequently at the ship." This game builds one: the small one, because *aiming* is the new skill here — and it pays the table's 1,000.

### Aiming: which way is the ship

Every collision so far has asked "do these two circles touch." A saucer's shot asks something different: "which way is the ship, from where I am?" The answer is an angle, and `Math.atan2` is the tool. The `Math.atan2()` method "measures the counterclockwise angle θ, in radians, between the positive x-axis and the point (x, y). Note that the arguments to this function pass the y-coordinate first and the x-coordinate second." ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/atan2))

Two things to notice. The argument order — y first, x second — is the opposite of nearly every coordinate pair you've written; `Math.atan2(dx, dy)` is the bug this part's likely-errors list is waiting for. And the returned angle is in canvas coordinates, where 0 points right (+x) and `Math.PI / 2` points down (+y). That's a different convention from the ship's angle (0 points up) — and the two coexist fine as long as each is used with its own cos/sin. The saucer's shot doesn't need the ship's convention at all: a velocity of `(Math.cos(angle) * speed, Math.sin(angle) * speed)` flies in the direction `atan2` measured.

And the aiming has to respect the torus. If the saucer is at x = 50 and the ship at x = 760, the plain delta says the ship is 710 px to the right: `atan2` aims right, the shot crosses the whole field, wraps, and arrives — if it survives — from behind, a second and a half late. `wrappedDelta` says the ship is 90 px to the *left*, and the shot takes the short way. Part 3's `hit()` already uses `wrappedDelta` inside, so a bullet can hit the saucer across a seam without any new code; the aim is the only place that needs it, and it needs it in both axes.

### The saucer's object

Constants first, added to the tuning block:

```js
    const SAUCER_SPEED = 60; // pixels per second, horizontal
    const SAUCER_RADIUS = 16; // the saucer's collision circle
```

60 px/s crosses the 800-px field in about 13 seconds — a slow, patient threat, not a race. The spawn, added after `resetGame`:

```js
    function spawnSaucer() {
      const dir = Math.random() < 0.5 ? 1 : -1;
      saucer = {
        x: dir === 1 ? -20 : canvas.width + 20,
        y: 60 + Math.random() * (canvas.height - 120),
        vx: dir * SAUCER_SPEED,
        fire: 1.5, // seconds until its first shot
      };
    }
```

It enters 20 px off one of the horizontal edges (so it doesn't pop in on screen), at a random height that keeps it clear of the top and bottom borders, and drifts across. `fire` is its own timer, counting down to the first shot.

### Moving and firing

In `update`, after the asteroid-movement block:

```js
      // The saucer: spawn on a timer, then move and fire at the ship.
      if (saucer === null) {
        saucerTimer -= dt;
        if (saucerTimer <= 0) {
          spawnSaucer();
          saucerTimer = 20 + Math.random() * 20;
        }
      } else {
        saucer.x = wrap(saucer.x + saucer.vx * dt, canvas.width);
        saucer.fire -= dt;
        if (saucer.fire <= 0) {
          const dx = wrappedDelta(saucer.x, ship.x, canvas.width);
          const dy = wrappedDelta(saucer.y, ship.y, canvas.height);
          const angle = Math.atan2(dy, dx);
          hostileBullets.push({
            x: saucer.x,
            y: saucer.y,
            vx: Math.cos(angle) * BULLET_SPEED,
            vy: Math.sin(angle) * BULLET_SPEED,
            life: BULLET_LIFE,
          });
          saucer.fire = 1.5;
        }
      }
```

The structure mirrors the player's bullets, inverted: there's no cap on `hostileBullets`, because the timer is the cap — a shot lives 1.5 seconds and one leaves every 1.5 seconds, so at most one is alive at a time. And the shots reuse `BULLET_SPEED` and `BULLET_LIFE`, so your gun and its gun trade at equal range.

> [!DESIGN-NOTE]
> **The seam, and the strategy it closed.** The original's saucers "can only aim at the player's ship on-screen; they are not capable of aiming across a screen boundary." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) That limitation is what made the game's famous exploit possible: "These behaviors allow a 'lurking' strategy, in which the player stays near the edge of the screen opposite the saucer." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) Hold the far edge, let the saucer fire across the seam past you, and shoot it for 1,000 points at almost no risk. Ours aims the short way across the seam, so the strategy dies. The trade is deliberate: a saucer that can't see half the field feels broken in a wrapping game, and 13 seconds of patrol is long enough that you'll usually have it in your sights.

The original's other quirk goes with it: "As the player's score increases, the angle range of the shots from the small saucer diminishes until the saucer fires extremely accurately." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) Ours is always accurate; accuracy as a difficulty dial is an exercise below.

### Its bullets, like yours

They move, wrap, and age out exactly the way yours do. After the player's bullet block in `update`:

```js
      // Hostile bullets: move, wrap, age out.
      for (const h of hostileBullets) {
        h.x = wrap(h.x + h.vx * dt, canvas.width);
        h.y = wrap(h.y + h.vy * dt, canvas.height);
        h.life -= dt;
      }
      for (let i = hostileBullets.length - 1; i >= 0; i--) {
        if (hostileBullets[i].life <= 0) hostileBullets.splice(i, 1);
      }
```

One difference from the player's bullets: they hit only the ship. There's no rule that a saucer's shot hits a rock, so its shots pass through the field's rocks — and one block, after the ship-vs-asteroid pass:

```js
      // Collisions: hostile bullet vs. ship, only when not invincible.
      if (ship.invincible <= 0) {
        for (let i = hostileBullets.length - 1; i >= 0; i--) {
          const h = hostileBullets[i];
          if (hit(h.x, h.y, 2, ship.x, ship.y, SHIP_RADIUS)) {
            hostileBullets.splice(i, 1);
            killShip();
            break;
          }
        }
      }
```

`killShip()` again — the saucer's bullets and the rocks buy the same thing: a life. The invincibility guard means a freshly respawned ship is safe while it blinks, against rocks and bullets alike.

### Killing it

The player's bullet pass gets a second target. Replace Part 3's bullet-vs-asteroid pass with this:

```js
      // Collisions: bullet vs. asteroid and vs. saucer.
      for (let i = bullets.length - 1; i >= 0; i--) {
        const b = bullets[i];
        let spent = false;
        for (let j = asteroids.length - 1; j >= 0; j--) {
          const a = asteroids[j];
          if (hit(b.x, b.y, 2, a.x, a.y, a.radius)) {
            split(j);
            spent = true;
            break; // this bullet is spent; on to the next
          }
        }
        if (!spent && saucer !== null &&
            hit(b.x, b.y, 2, saucer.x, saucer.y, SAUCER_RADIUS)) {
          addScore(1000);
          saucer = null;
          spent = true;
        }
        if (spent) bullets.splice(i, 1);
      }
```

The `spent` flag is the only structural change: each bullet checks the rocks, then the saucer, and the single `splice` at the end removes it once if it hit anything. The 1,000 goes through `addScore`, so a saucer kill can award the extra life.

### Drawing it

A saucer is a disk with a dome — two ellipses. The `ellipse()` method "creates an elliptical arc centered at (x, y) with the radii radiusX and radiusY" ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/ellipse)). After `drawAsteroid`:

```js
    function drawSaucer() {
      for (const [dx, dy] of drawOffsets(saucer.x, saucer.y, 20)) {
        ctx.save();
        ctx.translate(saucer.x + dx, saucer.y + dy);
        ctx.beginPath();
        ctx.ellipse(0, 0, 18, 8, 0, 0, Math.PI * 2); // the disk
        ctx.stroke();
        ctx.beginPath();
        ctx.ellipse(0, -3, 8, 5, 0, Math.PI, Math.PI * 2); // the dome
        ctx.stroke();
        ctx.restore();
      }
    }
```

The disk is a full ellipse (0 to `2π`), 36 wide by 16 tall. The dome runs from `π` to `2π` — the top half of a smaller ellipse, sitting on the disk's top edge. `drawOffsets` with r = 20 does the seam copies the way it does for the ship: the saucer is 36 px wide, and 20 is just past half that.

In `draw()`, after the player's bullet loop — the hostile dots first, then the saucer:

```js
      // Hostile bullets: the same dots, so its fire looks like yours.
      for (const h of hostileBullets) {
        ctx.beginPath();
        ctx.arc(h.x, h.y, 2, 0, Math.PI * 2);
        ctx.fill();
      }

      if (saucer !== null) drawSaucer();
```

Reload and wait. Somewhere between 20 and 40 seconds a saucer glides in from an edge, and a dot leaves it every 1.5 seconds — always aimed at the ship, always taking the short way across the seam. One of your dots ends it for 1,000 points, and the timer starts its next patrol.

## Checkpoint

> [!PREDICT]
> Clear the whole field. As the last small rock breaks, what appears — and what does the scoreboard say? (Level 2, and 5 large rocks spawn — `3 + 2`. The score is 2,080: 4 large × 20 + 8 medium × 50 + 16 small × 100. The spare ships are still two: 2,080 is nowhere near 10,000.) Now fly into rocks until the run ends. (Deaths one and two print `ship hit — respawning at center` and each cost a triangle. The third prints `game over — final score N`, and the field *freezes* — the rocks hold mid-drift behind the GAME OVER text, because `update` no longer runs. Press Enter: fresh field, `SCORE 0`, `LEVEL 1`, two triangles, the ship blinking at center.)

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
      angle: 0, // 0 points up; positive rotates clockwise
      invincible: 0 // seconds of invincibility left after a respawn
    };

    // The field: two collections, because rocks and shots have different rules.
    const asteroids = [];
    const bullets = [];

    // The saucer's state: null until one enters the field.
    let saucer = null;
    let saucerTimer = 20; // seconds until the next saucer
    const hostileBullets = []; // the saucer's shots: they hit only the ship

    // The game's scoreboard.
    let score = 0;
    let lives = 3; // the original starts the player with 3-5; this one takes 3
    let level = 1; // which wave is on the field
    let nextLifeAt = 10000; // score at which the next extra life is earned
    let over = false; // game over: the field is frozen until Enter

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
    const BULLET_SPEED = 360; // pixels per second
    const BULLET_LIFE = 1.5; // seconds a bullet stays alive
    const MAX_BULLETS = 4; // shots on screen at once
    const SHIP_RADIUS = 15; // the ship's collision circle
    const SAUCER_SPEED = 60; // pixels per second, horizontal
    const SAUCER_RADIUS = 16; // the saucer's collision circle

    // Asteroid sizes, index 0 = small, 2 = large.
    const RADII = [14, 24, 40];
    const SIDES = [6, 8, 10]; // polygon vertices per size
    const SPEEDS = [
      [80, 130], // small: fast
      [40, 80], // medium
      [20, 45], // large: slow
    ];

    // What each size pays, index 0 = small.
    const SCORES = [100, 50, 20];

    // Bring any value back into [0, size). % keeps the sign of its left
    // operand, so a negative needs one size added back.
    function wrap(v, size) {
      v %= size;
      return v < 0 ? v + size : v;
    }

    // One rock's outline: a closed polygon of uneven radii around the origin.
    function makeShape(radius, sides) {
      const pts = [];
      for (let i = 0; i < sides; i++) {
        const a = (i / sides) * Math.PI * 2;
        const r = radius * (0.75 + Math.random() * 0.5);
        pts.push([Math.cos(a) * r, Math.sin(a) * r]);
      }
      return pts;
    }

    // size: 2 large, 1 medium, 0 small.
    function makeAsteroid(x, y, size) {
      const radius = RADII[size];
      const [lo, hi] = SPEEDS[size];
      const speed = lo + Math.random() * (hi - lo);
      const dir = Math.random() * Math.PI * 2;
      const shape = makeShape(radius, SIDES[size]);
      let reach = 0;
      for (const [px, py] of shape) {
        reach = Math.max(reach, Math.hypot(px, py));
      }
      return {
        x,
        y,
        size,
        radius,
        vx: Math.cos(dir) * speed,
        vy: Math.sin(dir) * speed,
        angle: Math.random() * Math.PI * 2,
        spin: (0.3 + Math.random()) * (Math.random() < 0.5 ? -1 : 1),
        shape,
        reach: reach + 2, // furthest ink, plus the stroke — used at the edges
      };
    }

    // Spawn n rocks of a size, none within 150 px of the ship.
    function spawnAsteroids(n, size) {
      for (let i = 0; i < n; i++) {
        let x, y;
        do {
          x = Math.random() * canvas.width;
          y = Math.random() * canvas.height;
        } while (Math.hypot(x - ship.x, y - ship.y) < 150);
        asteroids.push(makeAsteroid(x, y, size));
      }
    }

    spawnAsteroids(4, 2);

    // Draw offsets for an object straddling a seam: the base position,
    // plus copies on the opposite edge(s). Handles corners (two seams).
    function drawOffsets(x, y, r) {
      const xs = [0];
      if (x < r) xs.push(canvas.width);
      if (x > canvas.width - r) xs.push(-canvas.width);
      const ys = [0];
      if (y < r) ys.push(canvas.height);
      if (y > canvas.height - r) ys.push(-canvas.height);
      const out = [];
      for (const dx of xs) for (const dy of ys) out.push([dx, dy]);
      return out;
    }

    // The shortest signed distance from a to b, across the seam:
    // a bullet at x = 5 and a rock at x = 795 are 10 apart, not 790.
    function wrappedDelta(a, b, size) {
      let d = a - b;
      if (d > size / 2) d -= size;
      if (d < -size / 2) d += size;
      return d;
    }

    // Circle vs. circle, on the torus. Squared, so no sqrt.
    function hit(ax, ay, ar, bx, by, br) {
      const dx = wrappedDelta(ax, bx, canvas.width);
      const dy = wrappedDelta(ay, by, canvas.height);
      const r = ar + br;
      return dx * dx + dy * dy < r * r;
    }

    function addScore(points) {
      score += points;
      if (score >= nextLifeAt) {
        lives++;
        nextLifeAt += 10000;
      }
    }

    // Replace the rock at index with two smaller, faster ones (or nothing).
    function split(index) {
      const a = asteroids[index];
      asteroids.splice(index, 1);
      addScore(SCORES[a.size]);
      if (a.size > 0) {
        for (let i = 0; i < 2; i++) {
          asteroids.push(makeAsteroid(a.x, a.y, a.size - 1));
        }
      }
    }

    function respawn() {
      console.log("ship hit — respawning at center");
      ship.x = 400;
      ship.y = 300;
      ship.vx = 0;
      ship.vy = 0;
      ship.invincible = 2;
    }

    function killShip() {
      lives--;
      if (lives <= 0) {
        over = true;
        console.log("game over — final score", score);
      } else {
        respawn();
      }
    }

    function resetGame() {
      score = 0;
      lives = 3;
      level = 1;
      nextLifeAt = 10000;
      over = false;
      bullets.length = 0;
      hostileBullets.length = 0;
      asteroids.length = 0;
      saucer = null;
      saucerTimer = 20;
      ship.x = 400;
      ship.y = 300;
      ship.vx = 0;
      ship.vy = 0;
      ship.invincible = 2;
      spawnAsteroids(4, 2);
    }

    function spawnSaucer() {
      const dir = Math.random() < 0.5 ? 1 : -1;
      saucer = {
        x: dir === 1 ? -20 : canvas.width + 20,
        y: 60 + Math.random() * (canvas.height - 120),
        vx: dir * SAUCER_SPEED,
        fire: 1.5, // seconds until its first shot
      };
    }

    let time = 0; // game time in seconds, for the respawn blink

    function update(dt) {
      if (over) {
        if (keys.Enter) resetGame();
        return;
      }
      time += dt;

      if (keys.ArrowLeft)  ship.angle -= ROT_SPEED * dt;
      if (keys.ArrowRight) ship.angle += ROT_SPEED * dt;

      if (keys.Space) {
        ship.vx += Math.sin(ship.angle) * THRUST * dt;
        ship.vy += -Math.cos(ship.angle) * THRUST * dt;
      }

      ship.x = wrap(ship.x + ship.vx * dt, canvas.width);
      ship.y = wrap(ship.y + ship.vy * dt, canvas.height);
      if (ship.invincible > 0) ship.invincible -= dt;

      // Fire: hold J, capped at MAX_BULLETS, at most one shot per frame.
      if (keys.KeyJ && bullets.length < MAX_BULLETS) {
        const nx = Math.sin(ship.angle);
        const ny = -Math.cos(ship.angle);
        bullets.push({
          x: ship.x + nx * 18,
          y: ship.y + ny * 18,
          vx: nx * BULLET_SPEED,
          vy: ny * BULLET_SPEED,
          life: BULLET_LIFE,
        });
      }

      // Bullets: move, wrap, age out.
      for (const b of bullets) {
        b.x = wrap(b.x + b.vx * dt, canvas.width);
        b.y = wrap(b.y + b.vy * dt, canvas.height);
        b.life -= dt;
      }
      for (let i = bullets.length - 1; i >= 0; i--) {
        if (bullets[i].life <= 0) bullets.splice(i, 1);
      }

      // Hostile bullets: move, wrap, age out.
      for (const h of hostileBullets) {
        h.x = wrap(h.x + h.vx * dt, canvas.width);
        h.y = wrap(h.y + h.vy * dt, canvas.height);
        h.life -= dt;
      }
      for (let i = hostileBullets.length - 1; i >= 0; i--) {
        if (hostileBullets[i].life <= 0) hostileBullets.splice(i, 1);
      }

      // Asteroids: move, spin, wrap.
      for (const a of asteroids) {
        a.x = wrap(a.x + a.vx * dt, canvas.width);
        a.y = wrap(a.y + a.vy * dt, canvas.height);
        a.angle += a.spin * dt;
      }

      // The saucer: spawn on a timer, then move and fire at the ship.
      if (saucer === null) {
        saucerTimer -= dt;
        if (saucerTimer <= 0) {
          spawnSaucer();
          saucerTimer = 20 + Math.random() * 20;
        }
      } else {
        saucer.x = wrap(saucer.x + saucer.vx * dt, canvas.width);
        saucer.fire -= dt;
        if (saucer.fire <= 0) {
          const dx = wrappedDelta(saucer.x, ship.x, canvas.width);
          const dy = wrappedDelta(saucer.y, ship.y, canvas.height);
          const angle = Math.atan2(dy, dx);
          hostileBullets.push({
            x: saucer.x,
            y: saucer.y,
            vx: Math.cos(angle) * BULLET_SPEED,
            vy: Math.sin(angle) * BULLET_SPEED,
            life: BULLET_LIFE,
          });
          saucer.fire = 1.5;
        }
      }

      // Collisions: bullet vs. asteroid and vs. saucer.
      for (let i = bullets.length - 1; i >= 0; i--) {
        const b = bullets[i];
        let spent = false;
        for (let j = asteroids.length - 1; j >= 0; j--) {
          const a = asteroids[j];
          if (hit(b.x, b.y, 2, a.x, a.y, a.radius)) {
            split(j);
            spent = true;
            break; // this bullet is spent; on to the next
          }
        }
        if (!spent && saucer !== null &&
            hit(b.x, b.y, 2, saucer.x, saucer.y, SAUCER_RADIUS)) {
          addScore(1000);
          saucer = null;
          spent = true;
        }
        if (spent) bullets.splice(i, 1);
      }

      // Collisions: ship vs. asteroid, only when not invincible.
      if (ship.invincible <= 0) {
        for (const a of asteroids) {
          if (hit(ship.x, ship.y, SHIP_RADIUS, a.x, a.y, a.radius)) {
            killShip();
            break;
          }
        }
      }

      // Collisions: hostile bullet vs. ship, only when not invincible.
      if (ship.invincible <= 0) {
        for (let i = hostileBullets.length - 1; i >= 0; i--) {
          const h = hostileBullets[i];
          if (hit(h.x, h.y, 2, ship.x, ship.y, SHIP_RADIUS)) {
            hostileBullets.splice(i, 1);
            killShip();
            break;
          }
        }
      }

      // Wave: the field is cleared, so the next level starts.
      if (asteroids.length === 0) {
        level++;
        spawnAsteroids(Math.min(3 + level, 8), 2);
      }
    }

    function drawAsteroid(a) {
      for (const [dx, dy] of drawOffsets(a.x, a.y, a.reach)) {
        ctx.save();
        ctx.translate(a.x + dx, a.y + dy);
        ctx.rotate(a.angle);
        ctx.beginPath();
        ctx.moveTo(a.shape[0][0], a.shape[0][1]);
        for (const [px, py] of a.shape.slice(1)) {
          ctx.lineTo(px, py);
        }
        ctx.closePath();
        ctx.stroke();
        ctx.restore();
      }
    }

    function drawSaucer() {
      for (const [dx, dy] of drawOffsets(saucer.x, saucer.y, 20)) {
        ctx.save();
        ctx.translate(saucer.x + dx, saucer.y + dy);
        ctx.beginPath();
        ctx.ellipse(0, 0, 18, 8, 0, 0, Math.PI * 2); // the disk
        ctx.stroke();
        ctx.beginPath();
        ctx.ellipse(0, -3, 8, 5, 0, Math.PI, Math.PI * 2); // the dome
        ctx.stroke();
        ctx.restore();
      }
    }

    // One full repaint of the field: background, rocks, bullets, the saucer, and the ship.
    function draw() {
      ctx.fillStyle = "#000000";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;

      for (const a of asteroids) drawAsteroid(a);

      // Bullets: 4-pixel dots, small enough to skip the edge copies.
      ctx.fillStyle = "#ffffff";
      for (const b of bullets) {
        ctx.beginPath();
        ctx.arc(b.x, b.y, 2, 0, Math.PI * 2);
        ctx.fill();
      }

      // Hostile bullets: the same dots, so its fire looks like yours.
      for (const h of hostileBullets) {
        ctx.beginPath();
        ctx.arc(h.x, h.y, 2, 0, Math.PI * 2);
        ctx.fill();
      }

      if (saucer !== null) drawSaucer();

      // The ship, blinking while invincible.
      if (ship.invincible <= 0 || Math.floor(time / 0.1) % 2 === 0) {
        for (const [dx, dy] of drawOffsets(ship.x, ship.y, 20)) {
          ctx.save();
          ctx.translate(ship.x + dx, ship.y + dy);
          ctx.rotate(ship.angle);
          ctx.beginPath();
          ctx.moveTo(0, -15); // nose
          ctx.lineTo(-12, 12); // back left
          ctx.lineTo(12, 12); // back right
          ctx.closePath();
          ctx.stroke();
          ctx.restore();
        }
      }

      // The scoreboard: score and level, top-left.
      ctx.fillStyle = "#ffffff";
      ctx.font = "16px monospace";
      ctx.fillText(`SCORE ${score}`, 10, 24);
      ctx.fillText(`LEVEL ${level}`, 10, 44);

      // Spare ships, bottom-left: the ship's shape at half scale.
      for (let i = 0; i < Math.min(lives - 1, 9); i++) {
        ctx.save();
        ctx.translate(24 + i * 24, canvas.height - 18);
        ctx.scale(0.5, 0.5);
        ctx.beginPath();
        ctx.moveTo(0, -15);
        ctx.lineTo(-12, 12);
        ctx.lineTo(12, 12);
        ctx.closePath();
        ctx.stroke();
        ctx.restore();
      }

      // Game over: the frozen field behind a restart prompt.
      if (over) {
        ctx.font = "32px monospace";
        ctx.fillText("GAME OVER", canvas.width / 2 - 100, canvas.height / 2 - 10);
        ctx.font = "16px monospace";
        ctx.fillText(`FINAL SCORE ${score}`, canvas.width / 2 - 70, canvas.height / 2 + 20);
        ctx.fillText("PRESS ENTER TO RESTART", canvas.width / 2 - 110, canvas.height / 2 + 50);
      }
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
- The same page as before — black 800×600 field, gray border, gray page — now holding four lumpy, slowly spinning and drifting asteroids, none of them within 150 px of the ship at the center.
- In the top-left: `SCORE 0` and `LEVEL 1` in white monospace. In the bottom-left: two small triangles — the spare ships.
- In the DevTools console (F12 → Console), two lines on load:

```
module ran, canvas is 800 x 600
first frame at 12.5 ms
```

(the number on the second line varies from load to load).
- Break a large rock: the score jumps by 20. Clear the whole field (28 rocks — 4 large, 8 medium, 16 small): the score reads 2,080, `LEVEL 2`, and five large rocks spawn.
- Wait 20–40 seconds: a saucer glides in from a horizontal edge, and a dot leaves it every 1.5 seconds, aimed at the ship — across the seam if that's the short way. A player dot that hits it adds 1,000 to the score.
- Fly into a rock: the console prints `ship hit — respawning at center`, a triangle leaves the bottom-left, and the ship blinks at center. Die with the last triangle: the console prints `game over — final score N`, the field freezes, and GAME OVER sits over it. Press Enter: fresh field, `SCORE 0`, `LEVEL 1`, two triangles.

**Likely errors:**
- **The game-over screen shows, but the rocks keep drifting behind it.** `update` is missing the `if (over) { …; return; }` gate — the draw shows the state, but the simulation never stops.
- **Enter does nothing on the game-over screen.** The restart check is *after* the early return, so it never runs. The `if (keys.Enter) resetGame();` line belongs inside the `over` block, before the `return`.
- **The spare-ship row never shrinks, and the game never ends.** The ship-vs-asteroid pass still calls `respawn()` directly instead of `killShip()` — death is still free, `lives` never decrements, and `over` is never set.
- **The saucer fires straight up, straight down, or mirrored at the ship.** The `atan2` arguments are swapped: it is `Math.atan2(dy, dx)`, y first. Swapped, every shot aims 90° off, in a direction that looks plausible while it's wrong.
- **The wave never comes, even after the field is clear.** The `asteroids.length === 0` check is missing — or it waits for the saucer too (`asteroids.length === 0 && saucer === null`), which stalls the wave for the saucer's whole patrol, since this saucer wraps instead of leaving.
- **The extra life never comes, or it comes on every rock past 10,000.** The second problem is the subtle one: `nextLifeAt` is never incremented, so every rock after the threshold awards another life and the spare-ship row fills up in seconds. (The first — it never comes — is usually the check missing from `addScore`, or a `>` where a `>=` is needed.)

## What's next

This is the last part of the series, so "next" means two things: what this game deliberately left out, and what you can build next with the same spine.

Left out, by choice:

- **Sound.** Every effect in the original — the rock's crack, the saucer's wail, the ship's explosion — is a short oscillator burst, and the Web Audio API makes one in a few lines.
- **Hyperspace.** The original's fifth control: "The player can also send the ship into hyperspace, causing it to disappear and reappear in a random location on the screen, at the risk of self-destructing or appearing on top of an asteroid." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) One key, two random coordinates, and a coin flip on the outcome.
- **The large saucer, and the 40,000-point switch.** "After reaching a score of 40,000, only the small saucer appears." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) A second saucer type is a state machine with one extra variable.
- **A high score that survives the reload.** A `high` variable that `resetGame` doesn't clear gets you a session high score; `localStorage` gets you a permanent one.
- **Physics polish.** The original's ship has drag (ours drifts forever); its small saucer gets more accurate as the score rises (ours is always accurate); and its machine "turns over" at 99,990 points (ours has no ceiling).

The spine that carried all four parts, if you're building something else: state is data (`ship`, `asteroids`, `bullets`, the scoreboard), drawing is a function of that state, the clock is `requestAnimationFrame`'s timestamp, and `dt` is what makes speed mean something independent of the monitor. Any 2D game — breakout, a space shooter, a top-down arena — is more of the same: more objects in the field, more rules in `update`, more shapes in `draw`.

Before you go: in one sentence each — what does the canvas do for you that you had to write yourself, and what do you do for the canvas? (One honest pair: it rasterizes vector shapes to pixels and hands you a frame clock; you decide what the shapes mean, when they move, and what touches what. If your answers name `update` and `draw`, you've got the split right.)

## Exercises

- [ ] **The 10,000-point life.** Verify the rule you trusted earlier: play until the score crosses 10,000 (about five cleared waves) and watch a triangle appear in the bottom-left. Then push past 20,000 and confirm `nextLifeAt` moved — a second life, not a life per rock.
- [ ] **The large saucer.** The original's other saucer — the 'big saucer' — "shoots randomly and poorly" and is worth 200 points ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) plus the scoring table). Give `saucer` a `type`: "large" while `score < 40000`, "small" after — "After reaching a score of 40,000, only the small saucer appears." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) The large one fires every 2 seconds in a *random* direction (`Math.random() * Math.PI * 2` is a full circle of angles) and pays 200; draw it bigger — the same two ellipses with radii × 1.4.
- [ ] **Lurking.** Make the saucer aim the way the original does: replace the `wrappedDelta` pair with plain deltas (`const dx = ship.x - saucer.x;` and the y equivalent), and skip firing entirely when the ship is across a seam from the saucer (a hint: that's when `Math.abs(dx) > canvas.width / 2` or the y equivalent). Then play the strategy the sources describe — hold position on the edge opposite the saucer, keep one or two rocks in play, and let its shots cross the field past you while you shoot it for 1,000 points. See why it worked, and why this part closed the door on it.
- [ ] **Hyperspace.** The original's fifth control. On **H**, edge-triggered (a `hyperspace` flag cleared in `keyup`, or a 1-second cooldown), teleport the ship to a random position at least 60 px from every rock's edge (loop random picks until one clears, giving up after 20 tries), keep its velocity, and grant 0.5 s of invincibility so the blink covers the jump. The risk the original keeps — appearing on a rock — you've just removed; restoring it (death on a bad roll) is the honest version.
- [ ] **Accuracy as difficulty.** "As the player's score increases, the angle range of the shots from the small saucer diminishes until the saucer fires extremely accurately." ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) Add jitter to the aimed angle: `angle + (Math.random() - 0.5) * spread`, with `spread` starting around 0.6 radians and shrinking toward 0 as the score passes 40,000 — `Math.max(0, 0.6 * (1 - score / 40000))`. Low scores feel the mercy; high scores don't.

## Sources

**The original game**

1. [Asteroids (video game) — Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) — the wave start ("Once the screen has been cleared of all asteroids and flying saucers, a new set of large asteroids appears, thus starting the next level"), the difficulty ramp ("The game gets harder as the number of asteroids increases until after the score reaches a range between 40,000 and 60,000"), the lives ("The player starts with 3–5 lives upon game start and gains an extra life per 10,000 points" and "Play continues to the last ship lost, which ends the game"), the saucers ("Two flying saucers appear periodically on the screen; the 'big saucer' shoots randomly and poorly, while the 'small saucer' fires frequently at the ship"), the 40,000-point switch ("After reaching a score of 40,000, only the small saucer appears"), the aiming limits ("saucers can only aim at the player's ship on-screen; they are not capable of aiming across a screen boundary" and "These behaviors allow a 'lurking' strategy, in which the player stays near the edge of the screen opposite the saucer"), the accuracy ramp ("As the player's score increases, the angle range of the shots from the small saucer diminishes until the saucer fires extremely accurately"), hyperspace ("The player can also send the ship into hyperspace … at the risk of self-destructing or appearing on top of an asteroid"), the uncapped-lives quirk ("Asteroids slows down as the player gains 50–100 lives, because there is no limit to the number of lives displayed" and "The game's code continues trying to draw them even if they fall outside the boundaries of the screen"), and the 99,990 turnover ("The machine 'turns over' at 99,990 points, which is the maximum high score that can be achieved").

**The original's scoring**

2. [Asteroids (1979) — Arcade History](https://www.arcade-history.com/game/126/) — the cabinet's scoring table: large asteroid 20 points, medium 50, small 100, large flying saucer 200, small flying saucer 1,000.

**Drawing text**

3. [CanvasRenderingContext2D.fillText() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/fillText) — renders the string "using the settings specified by font, textAlign, textBaseline, and direction"; `y` is "the y-axis coordinate of the baseline on which to begin drawing the text" — why the scoreboard text sits above its coordinate.
4. [CanvasRenderingContext2D.font — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/font) — "a string parsed as CSS font value"; the default is `10px sans-serif`, the reason the score needs its own font line.

**Aiming and the saucer's shape**

5. [Math.atan2() — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/atan2) — "measures the counterclockwise angle θ, in radians, between the positive x-axis and the point (x, y)," with "the y-coordinate first and the x-coordinate second" — the argument order that trips up the first saucer.
6. [CanvasRenderingContext2D.ellipse() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/ellipse) — "creates an elliptical arc centered at (x, y) with the radii radiusX and radiusY"; 0 to 2π is the full disk, π to 2π the top half.
7. [Transformations — MDN Canvas API tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations) — `scale(x, y)` "scales the canvas units by x horizontally and by y vertically," the spare ships' half-size triangles.

**Keys**

8. [KeyboardEvent.code — MDN](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/code) and [Keyboard event code values — MDN](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_code_values) — `Enter` is the key's code as-is, the restart key behind `keys.Enter`.
