# Asteroids in One HTML File, Part 3: Rocks, Bullets, and Collisions

> [!RECALL]
> In Part 2 the frame callback computed `const dt = last === null ? 0 : Math.min((t - last) / 1000, 0.1);`. Without scrolling back: what does each piece protect against — the `last === null` check, the `/ 1000`, and the `Math.min(…, 0.1)`? If you can't reconstruct all three, Part 2's *The timestamp, and dt* section has them.

Part 2 ended with one object in an empty field: a ship that turns, thrusts, and drifts — and that can leave through the top edge and stay gone. Part 3 makes the field a place with rules in it, the way the 1979 game had it: a level that starts "with multiple large asteroids drifting across the screen," objects that "wrap around screen edges," and a ship that can "fire shots straight forward" (all [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))). That is four new things in one part — rocks, wrap, bullets, and the decision about when one object touches another — and they build on each other in that order, because the rocks have to exist before the bullets can hit them, and the field has to wrap before the collision math can be trusted.

## What you'll build

The end state is the same `game.html`, with a field in it. On load the field holds four large asteroids — lumpy polygons, each with its own random outline, spin, and drift. In the browser you can:

- Hold **J** and the ship fires: up to 4 bullets at once, each flying straight at 360 px/s and gone after 1.5 seconds.
- Hit a rock with a bullet and it splits: large becomes two medium, medium becomes two small, small is gone — and the pieces are faster than the parent.
- Fly into a rock: the console prints `ship hit — respawning at center`, the ship resets to (400, 300) with zero velocity, and blinks for 2 seconds while it is invincible.
- Watch any object cross an edge — ship, rock, or bullet — and see it reappear at the opposite edge, same direction, same spin, with no visible jump at the seam.

Three things are deliberately *not* here yet: no score, no waves (the four rocks are the whole field, forever), and no saucer. Those are Part 4, where the game gets a goal and an ending.

## Prerequisites

- Part 2, completed: a `game.html` with the `requestAnimationFrame` loop, the key table, and the drifting ship. The console prints `module ran, canvas is 800 x 600` and `first frame at … ms`.
- The same browser as before. No new tools, no installation.

## From one object to a field of objects

The ship is one object: a `const` whose properties change. A field is many objects, so it is an *array* of objects — the same pattern, plural. Add after the `ship` block:

```js
    // The field: two collections, because rocks and shots have different rules.
    const asteroids = [];
    const bullets = [];
```

Two notes on that. First, `const` and changing contents go together: the arrays are never reassigned — the name `asteroids` always means this same array — but their *contents* change, with rocks and shots pushed in and spliced out. `const` pins the binding, not the object. Second, two arrays instead of one: a rock and a bullet share the shape `{x, y, vx, vy}`, but everything else differs (rocks spin and split; bullets age out), so their update loops are different, and separate arrays keep each loop a plain `for…of`. One array of "things" with a `kind` field would also work; it would also make every loop a switch statement. Two arrays, two loops.

Nothing is visible yet — both arrays are empty. Reload if you like; the field is unchanged.

## Rocks: a shape, a spin, and a spawn

A rock in this game is not a circle. It is a closed, lumpy polygon, and each rock's lumpiness is random — no two rocks share an outline, which costs almost nothing and reads as far less repetitive than a few fixed shapes. Rocks come in three sizes, and the size does the work the original's rules describe. [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)): "As the player shoots asteroids, they break into smaller asteroids that move faster and are more difficult to hit." So the small row of the table below is the fastest and the large row the slowest. The exact numbers are ours; the direction of the trend is the original's.

Add after the tuning constants:

```js
    // Asteroid sizes, index 0 = small, 2 = large.
    const RADII = [14, 24, 40];
    const SIDES = [6, 8, 10]; // polygon vertices per size
    const SPEEDS = [
      [80, 130], // small: fast
      [40, 80], // medium
      [20, 45], // large: slow
    ];
```

The outline. A rock's shape is a list of points around the origin — the same local-coordinates trick Part 2 used for the ship. Take `sides` points at evenly spaced angles around a circle and jitter each one's radius between 75% and 125% of the nominal radius:

```js
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
```

`0.75 + Math.random() * 0.5` is a value uniformly between `0.75` and `1.25`, so each vertex sits somewhere between `0.75·radius` and `1.25·radius` from the center — uniform jitter, which is what makes the polygons look lumpy rather than merely small or large circles. The points are in *local* coordinates around `(0, 0)`, exactly like the ship's `(0, -15)` and `(±12, 12)`: the rock's position and rotation get applied later with the same `translate` + `rotate` transform.

The rock itself is a position, a velocity, a size, a spin, and its shape:

```js
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
```

Walk the choices. `SPEEDS[size]` destructures into a `[lo, hi]` pair, and `lo + Math.random() * (hi - lo)` is a uniform speed between them — 20–45 px/s for a large rock, 80–130 for a small one. The direction `dir` is a uniform angle in `[0, 2π)`, and `(cos dir, sin dir)` is the unit vector along it — no conversion, because `dir` lives in the same y-down canvas world as the ship's `angle`. `spin` is 0.3–1.3 rad/s with a random sign, so rocks turn both ways at slightly different rates. And `reach`: the furthest any vertex of *this* rock actually sits from its center, plus two pixels for the stroke. You will not see what it is for until the next section — the edges.

Spawning. The original opens each level with large rocks away from the ship — [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)): "Each level starts with multiple large asteroids drifting across the screen." Ours starts with four, all large, none of them close:

```js
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
```

The `do…while` is rejection sampling: draw a random point; if it lands within 150 px of the ship, draw again. That circle is about 15% of the field (π·150² ÷ (800·600) ≈ 0.15), so the loop rarely spins more than a couple of times. And 150 px is well clear of a large rock: its furthest ink reaches about 50 px, the ship's collision circle is 15 px, and 50 + 15 = 65 < 150 — no rock is born in a position that touches the ship.

Then the drawing. Add a function next to `draw()`, using the same transform discipline as the ship:

```js
    function drawAsteroid(a) {
      ctx.save();
      ctx.translate(a.x, a.y);
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
```

Move the origin to the rock's center, rotate by `a.angle`, stroke the lumpy polygon in local coordinates, restore. `closePath()` is what makes it a closed loop: it draws the final edge, from the last vertex back to the first, when you stroke.

And wire it into `draw()`. The two style lines currently live inside the ship block, but they apply to everything that is stroked — move them to the top of the function and add the asteroid loop. After the edit, `draw()` looks like this:

```js
    // One full repaint of the field: background, rocks, bullets, and the ship.
    function draw() {
      ctx.fillStyle = "#000000";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;

      for (const a of asteroids) drawAsteroid(a);

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
      ctx.stroke();
      ctx.restore();
    }
```

(You'll see the comment names the bullets before the bullet section — the function's comment now lists everything it will paint by the end of this part.)

Reload: the field is no longer empty. Four lumpy polygons sit in it — drawn, but not yet moving. Their state (`x`, `y`, `angle`) changes nothing yet, because `update` doesn't touch the rocks. That is the next section, and it comes with a complication.

## Wrap-around: the field is a torus

Part 2's exercise patched the ship's edges with four `if` statements. This section does the real thing for every object, and it starts from a fact about the field's shape: **the field is a torus**. Glue the left and right edges of a sheet together, then glue the top and bottom, and you get a donut-shaped surface — and that is what the field is. An object that crosses an edge is not gone; it is at the same point on the other side. [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)): objects "wrap around screen edges" — "an asteroid that drifts off the top edge of the screen reappears at the bottom and continues moving in the same direction."

The position part is one function, and it starts from a trap in JavaScript's arithmetic. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder) on the remainder operator: it "returns the remainder left over when one operand is divided by a second operand" — and "It always takes the sign of the dividend." That second sentence is the trap: `-5 % 800` is `-5`, not `795` (MDN's own example: `-13 % 5` is `-3`). So `%` alone cannot wrap a negative number, and the fix is one line:

```js
    // Bring any value back into [0, size). % keeps the sign of its left
    // operand, so a negative needs one size added back.
    function wrap(v, size) {
      v %= size;
      return v < 0 ? v + size : v;
    }
```

Check the four cases: `810 → 10`, `400 → 400`, `-5 → 795`, and `-805 → 795` (because `-805 % 800` is `-5`, and then `+ 800`). One function, and every object in the field uses it. Add it after the size tables, before `makeShape`.

Then move the objects. In `update`, the ship's two movement lines become:

```js
      ship.x = wrap(ship.x + ship.vx * dt, canvas.width);
      ship.y = wrap(ship.y + ship.vy * dt, canvas.height);
```

and the function gains an asteroid loop at the end:

```js
      // Asteroids: move, spin, wrap.
      for (const a of asteroids) {
        a.x = wrap(a.x + a.vx * dt, canvas.width);
        a.y = wrap(a.y + a.vy * dt, canvas.height);
        a.angle += a.spin * dt;
      }
```

The same three lines per object as the ship, plus the spin. Reload and fly off the top: the ship comes back at the bottom, and the rocks drift, spin, and wrap around the field on their own.

Except — there is a crack in the illusion, and you can see it at the edges. A rock crossing the left edge does not reappear on the right *whole*. Its position wraps — the center jumps from x = 2 to x = 799 — but the rock is up to 100 px wide, so for a while its drawn body is split across the seam: the part past the right edge of the canvas is clipped and invisible, and the part that should be showing on the left is simply not drawn. The position is right; the drawing is not.

> [!DESIGN-NOTE]
> **Why the drawing lags the position.** Wrapping moves the rock's *center* by exactly one field width. The drawn body, though, extends `reach` pixels on either side of the center, and the canvas only shows `[0, 800] × [0, 600]` — anything outside is clipped, not wrapped. The canvas has no idea the field is a torus; wrap-around is entirely our code's business, drawing included.

The fix is to draw the missing half *on the other side of the seam*. When a rock's center is within `reach` of an edge, draw the rock again, shifted by one full field dimension — the visible part of the copy is exactly the part of the body the base copy clipped off. The offsets an object needs — add this after the `spawnAsteroids(4, 2);` line:

```js
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
```

Read it as: the base copy is always at offset `(0, 0)`. Near the left edge (`x < r`), also draw a copy shifted right by one field width — that copy's visible sliver is the body part that went off the left. Near the right edge, the mirror image: a copy shifted left. The y-axis does the same with the field height. A rock in a corner sits near two seams at once and gets four copies — base, right, below, and the far corner — and the two nested loops generate all the combinations without a special case. (The extra copies are mostly off-canvas and get clipped for free.)

Now `drawAsteroid` draws once per offset — replace it with:

```js
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
```

The body is unchanged — it is just drawn once per offset, with the offset added to the `translate`. And `a.reach` (the furthest ink, from the last section) is exactly the right `r` for `drawOffsets`: a copy is needed precisely when some ink can be across the seam.

The ship gets the same treatment. Its ink reaches the back corners at (±12, 12) — 17 px from the center — plus the stroke, about 18 px; pass 20. In `draw()`, replace the ship block (from the `// Move and rotate the coordinate system` comment through the `ctx.restore()`) with:

```js
      // The ship.
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
```

Reload: fly off the top and the ship comes back at the bottom whole; a rock crossing the left edge completes itself on the right; a rock in a corner shows all four of its pieces at once. The seam is no longer visible anywhere in the field. That is the torus, drawn.

One deliberate simplification: the bullets in the next section skip the edge copies. A bullet is a 4-px dot, so a dot straddling a seam is missing at most 4 px of a 4-px dot for about a frame — invisible. The copies earn their cost for the rocks and the ship, not for the dot.

## Bullets: straight, fast, and short-lived

The original's ship "can rotate left and right, fire shots straight forward, and thrust forward" ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))) — and the shooting is the half we have not built. A bullet is the simplest moving object in the game: a position, a velocity, and a lifetime. It does not rotate, does not spin, does not split.

No new input code. Part 2's key table records *every* key you press — `keys[e.code] = true` runs for all of them — and the physical J key has the code `"KeyJ"` ([MDN's code table](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_code_values) lists the letter keys as `"KeyA"` through `"KeyZ"`). So firing needs no new listener, and J does not join `GAME_KEYS` either: `preventDefault` exists to stop the page from scrolling, and J scrolls nothing. The only firing code is the reading of the table, in `update`.

Constants first — add to the tuning block, after `THRUST`:

```js
    const BULLET_SPEED = 360; // pixels per second
    const BULLET_LIFE = 1.5; // seconds a bullet stays alive
    const MAX_BULLETS = 4; // shots on screen at once
```

360 px/s is three times the fastest rock (130): a bullet always outruns its target, covering 540 px in its 1.5-second life — most of the 800-px field. The lifetime answers "how far should a shot travel" with: far enough to reach most of the field, short enough that old shots never accumulate. Four on screen at once keeps the field readable. Holding **J** fires one bullet per frame up to the cap, and the cap frees itself as old bullets expire.

The firing itself, in `update`, between the ship's movement lines and the asteroid loop:

```js
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
```

The nose direction `(sin angle, −cos angle)` is Part 2's thrust vector, reused. The bullet is born 18 px along it — three px past the nose, which is 15 px from the center — so a shot starts outside the ship's own outline. The cap does its job quietly: with J held down, the first four frames fire four bullets, and after that one slot opens per expiring bullet.

Then the bullets' own update — move, wrap, and age out:

```js
      // Bullets: move, wrap, age out.
      for (const b of bullets) {
        b.x = wrap(b.x + b.vx * dt, canvas.width);
        b.y = wrap(b.y + b.vy * dt, canvas.height);
        b.life -= dt;
      }
      for (let i = bullets.length - 1; i >= 0; i--) {
        if (bullets[i].life <= 0) bullets.splice(i, 1);
      }
```

Two loops, in that order: everyone moves and ages first, then the dead are removed. The second loop walks *backward*, and that is not style. `splice` removes an element and shifts everything after it one index down. Walk the array forward and the element that slides into the slot you just emptied sits at the very index you are about to visit — you skip it. Walk backward and every element still to be visited sits at a *lower* index than the one you just removed; nothing shifts under your feet. You will meet this pattern again, inside the collision loop.

Drawing: a bullet is a 4-px dot — a circle of radius 2. The method is `arc`, which per [MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/arc) "adds a circular arc to the current sub-path," starting at `startAngle` and ending at `endAngle`. From `0` to `Math.PI * 2` that is the whole circle. Add to `draw()`, after the asteroid loop:

```js
      // Bullets: 4-pixel dots, small enough to skip the edge copies.
      ctx.fillStyle = "#ffffff";
      for (const b of bullets) {
        ctx.beginPath();
        ctx.arc(b.x, b.y, 2, 0, Math.PI * 2);
        ctx.fill();
      }
```

The `beginPath` before every dot matters: `arc` adds to the *current* sub-path, so without a fresh path each dot would be joined to the previous one by a straight line. And `fill`, not `stroke` — a stroked radius-2 circle is a ring, and a filled one is a dot.

Reload: hold **J** and the field gets four dots that shoot out of the nose, fly straight, wrap at the edges, and vanish one by one 1.5 seconds after they were born. They hit nothing yet. That is the last section.

## Collisions: when a circle meets a circle

[The object of the game](https://en.wikipedia.org/wiki/Asteroids_(video_game)) is "to shoot and destroy the asteroids and saucers while avoiding colliding with either or being hit by the saucers' counterfire." This section builds both halves that involve rocks: a bullet meeting one, and the ship meeting one.

The decision is one question, asked 60 times a second for every pair that could touch: *do these two objects overlap?* The answer uses circles. Every object in the field has a collision circle — the ship 15 px (its nose, roughly its silhouette), a bullet 2 px (the dot), a rock its nominal `radius` — and two circles overlap exactly when the distance between their centers is less than the sum of the radii. By Pythagoras the squared distance is `dx·dx + dy·dy`, so the test never needs a square root. But the distance has to be the distance *around the torus*, and that needs one more function first:

```js
    // The shortest signed distance from a to b, across the seam:
    // a bullet at x = 5 and a rock at x = 795 are 10 apart, not 790.
    function wrappedDelta(a, b, size) {
      let d = a - b;
      if (d > size / 2) d -= size;
      if (d < -size / 2) d += size;
      return d;
    }
```

The field is a torus, so the straight-line difference between two positions can be the *long* way around. `wrappedDelta` takes the raw difference and, if it is more than half a field in either direction, subtracts (or adds) one full field — the short way around. Check: bullet at x = 5, rock at x = 795: `d = −790`, which is less than `−400`, so `d += 800` → `10`. Ten pixels apart, across the seam, and the test sees it.

Add `wrappedDelta` after `drawOffsets`, and then the test itself:

```js
    // Circle vs. circle, on the torus. Squared, so no sqrt.
    function hit(ax, ay, ar, bx, by, br) {
      const dx = wrappedDelta(ax, bx, canvas.width);
      const dy = wrappedDelta(ay, by, canvas.height);
      const r = ar + br;
      return dx * dx + dy * dy < r * r;
    }
```

> [!DESIGN-NOTE]
> **Circles, not the real shapes.** The ship is a triangle and the rocks are lumpy polygons, but every collision here is circle-versus-circle. That is deliberate: the circle test is a few subtractions, multiplications, and one comparison — and a round hitbox plays better than an exact one, because you die when a rock is clearly on top of you, not when a corner grazes you at a shallow angle. The drawn shapes are a drawing problem; the collision circles are the physics. The last exercise replaces the ship's circle with its exact triangle.

Now the two collision passes, at the end of `update`. First, bullets against rocks:

```js
      // Collisions: bullet vs. asteroid.
      for (let i = bullets.length - 1; i >= 0; i--) {
        const b = bullets[i];
        for (let j = asteroids.length - 1; j >= 0; j--) {
          const a = asteroids[j];
          if (hit(b.x, b.y, 2, a.x, a.y, a.radius)) {
            bullets.splice(i, 1);
            split(j);
            break; // this bullet is spent; on to the next
          }
        }
      }
```

Both loops backward — the outer because a hit splices the bullet (the reason from the last section), the inner because the next function splices the asteroid. And one bullet kills at most one rock per frame: the `break` ends the rock scan as soon as the bullet is spent.

The split — add it after `hit`:

```js
    // Replace the rock at index with two smaller, faster ones (or nothing).
    function split(index) {
      const a = asteroids[index];
      asteroids.splice(index, 1);
      if (a.size > 0) {
        for (let i = 0; i < 2; i++) {
          asteroids.push(makeAsteroid(a.x, a.y, a.size - 1));
        }
      }
    }
```

`size - 1` is the whole rule: a large (2) becomes two medium (1), a medium becomes two small (0), and a small — `a.size > 0` is false — simply stops existing. The pieces are born at the parent's position with fresh random velocities from `makeAsteroid`, and because the size tables make the smaller rows faster, the field speeds up as you break it — the original's rule: "they break into smaller asteroids that move faster and are more difficult to hit" ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))).

> [!DESIGN-NOTE]
> **The original's 26-rock cap.** The arcade machine had a hard memory limit, and [Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) records what it did when you hit it: "There is a limit of 26 asteroids. If there are already that many, shooting a large asteroid turns it into a single medium one, rather than two as per normal. Similarly, a medium asteroid turns into a single small one instead of splitting." `split` has no cap — there is no memory pressure to reproduce — but the rule is a fun exercise below, and it is the kind of detail that makes a clone feel like the game it is copying.

And the ship. First, a property on the ship (add a comma after `angle`, then a last property),

```js
      invincible: 0 // seconds of invincibility left after a respawn
```

and a radius in the tuning block:

```js
    const SHIP_RADIUS = 15; // the ship's collision circle
```

Then a game-time accumulator above `update` (the respawn blink will use it), incremented as the first line of `update`, with a blank line after it:

```js
    let time = 0; // game time in seconds, for the respawn blink
```

```js
      time += dt;
```

And the invincibility countdown, right after the ship's two movement lines:

```js
      if (ship.invincible > 0) ship.invincible -= dt;
```

Finally, add `respawn` after `split`, and then the second collision pass, after the bullet pass:

```js
    function respawn() {
      console.log("ship hit — respawning at center");
      ship.x = 400;
      ship.y = 300;
      ship.vx = 0;
      ship.vy = 0;
      ship.invincible = 2;
    }
```

```js
      // Collisions: ship vs. asteroid, only when not invincible.
      if (ship.invincible <= 0) {
        for (const a of asteroids) {
          if (hit(ship.x, ship.y, SHIP_RADIUS, a.x, a.y, a.radius)) {
            respawn();
            break;
          }
        }
      }
```

No backward loop here: nothing in this pass splices anything — the ship object is permanent (there is exactly one, and it is a `const`), so "death" is a *reset*, not a removal. And the `invincible` guard matters for two reasons. Gameplay: without it, a respawn *into* a rock (rocks can be anywhere by now) would re-kill the ship on the very next frame. And the countdown: `respawn` sets `invincible` to 2, and each frame the line you just added spends it down over two seconds, after which the guard opens again and the ship is solid.

The blink is the last piece. In `draw()`, replace the `// The ship.` line and the loop under it with:

```js
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
```

`Math.floor(time / 0.1)` changes value every 0.1 seconds of game time; taking it modulo 2 gives alternating on/off windows — the ship is drawn on the even ones and skipped on the odd ones, ten full blink cycles in the 2 seconds of invincibility. The `||` does the rest: once `invincible` reaches 0, the first half is always true and the ship is drawn every frame, whatever the clock says. The blink is drawn, not simulated — the ship is still fully solid for collision purposes during the 2 seconds, but the ship-vs-rock pass simply skips it, which is the invincibility.

Reload, and the field is a place with rules in it. Rocks drift and wrap. J fires dots that fly, wrap, and expire. A dot meeting a rock splits it into two faster pieces. The ship meeting a rock dies, prints a line, comes back at the center, and blinks through anything that happens to be there for two seconds.

> [!HEADS-UP]
> **Tunneling.** At normal frame rates a bullet moves `360 × 0.0167 ≈ 6` px per frame — far less than the smallest rock's 28-px diameter, so the circle test at each frame's endpoint never misses. But the dt clamp allows a single 0.1-second frame after a background-tab pause, and in one such frame a bullet moves 36 px — more than a small rock's diameter — and can pass straight through it unregistered. Games call this *tunneling*, and the standard fix is to test the whole segment a bullet swept during the frame, not just its endpoint. We live with the one-frame edge case: it costs one rock, once, after a tab switch.

## Checkpoint

> [!PREDICT]
> Count the rocks on load. Fly to the nearest one and shoot it. Then shoot both of the medium pieces it spawned. What is on the field? (Four large rocks on load. After the first shot: 3 large + 2 medium = 5 — the large one is gone and two faster mediums are flying apart from where it was. After both mediums: 3 large + 4 small = 7. Breaking a rock *adds* to the count until you kill the smalls.) Now fly straight into any rock. (The console prints `ship hit — respawning at center`; the ship is back at (400, 300) with zero velocity and blinks — rocks pass through it for 2 seconds — then it is solid again.)

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

    // Asteroid sizes, index 0 = small, 2 = large.
    const RADII = [14, 24, 40];
    const SIDES = [6, 8, 10]; // polygon vertices per size
    const SPEEDS = [
      [80, 130], // small: fast
      [40, 80], // medium
      [20, 45], // large: slow
    ];

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

    // Replace the rock at index with two smaller, faster ones (or nothing).
    function split(index) {
      const a = asteroids[index];
      asteroids.splice(index, 1);
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

    let time = 0; // game time in seconds, for the respawn blink

    function update(dt) {
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

      // Asteroids: move, spin, wrap.
      for (const a of asteroids) {
        a.x = wrap(a.x + a.vx * dt, canvas.width);
        a.y = wrap(a.y + a.vy * dt, canvas.height);
        a.angle += a.spin * dt;
      }

      // Collisions: bullet vs. asteroid.
      for (let i = bullets.length - 1; i >= 0; i--) {
        const b = bullets[i];
        for (let j = asteroids.length - 1; j >= 0; j--) {
          const a = asteroids[j];
          if (hit(b.x, b.y, 2, a.x, a.y, a.radius)) {
            bullets.splice(i, 1);
            split(j);
            break; // this bullet is spent; on to the next
          }
        }
      }

      // Collisions: ship vs. asteroid, only when not invincible.
      if (ship.invincible <= 0) {
        for (const a of asteroids) {
          if (hit(ship.x, ship.y, SHIP_RADIUS, a.x, a.y, a.radius)) {
            respawn();
            break;
          }
        }
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

    // One full repaint of the field: background, rocks, bullets, and the ship.
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
- In the DevTools console (F12 → Console), two lines on load:

```
module ran, canvas is 800 x 600
first frame at 12.5 ms
```

(the number on the second line varies from load to load).
- Hold **J**: up to 4 white dots fly out of the nose in a straight line, wrap at the edges, and each one vanishes 1.5 seconds after it was fired.
- Hit a large rock: it disappears and two faster medium rocks burst out of the same spot, spinning at random. Hit a medium: two smalls. Hit a small: it is gone.
- Let a rock cross an edge: its continuation appears at the opposite edge in the same direction and spin, and the seam is not visible. A rock sitting in a corner shows all four of its pieces.
- Fly into any rock: the console prints `ship hit — respawning at center`, the ship resets to (400, 300) with zero velocity, and blinks (drawn for 0.1 s, hidden for 0.1 s) for 2 seconds while rocks pass through it.

**Likely errors:**
- **Rocks pop in and out at the edges instead of crossing smoothly.** `drawAsteroid` is missing the `drawOffsets` loop — the position wraps, but the drawing doesn't know about the seam. Or `drawOffsets` is missing a corner case: a rock in a corner needs four copies, and the two nested loops are what generate them.
- **A bullet passes through a rock sitting across the seam.** The collision test was written with a plain `b.x - a.x` instead of `wrappedDelta` — two objects 10 px apart across the seam measure 790 px apart, and `hit` never fires.
- **The gun jams after a few shots and never fires again.** The bullets never age out: `b.life -= dt` is missing (or landed in the wrong loop), so the 4-bullet cap is permanently full and the `bullets.length < MAX_BULLETS` check is permanently false.
- **The ship dies again immediately after respawning, and the console spams `ship hit`.** The ship-vs-asteroid pass is missing the `ship.invincible <= 0` guard, or `respawn()` doesn't set `invincible` — if the ship comes back next to a rock, it re-dies every frame.
- **The ship doesn't wrap, or it wraps the wrong way.** The ship's movement lines were never changed to use `wrap`, or `canvas.width` and `canvas.height` are swapped in one of them — the ship reappears at the wrong edge (a ship leaving the top should come back at the bottom, not the right).
- **The console shows `Uncaught SyntaxError` and the field is a blank black box.** Same as Parts 1 and 2: a module aborts on a syntax error, so the loop never starts. Fix the line the error points at.

## What's next

The field now has rules, but no goal: the four rocks are the whole field, forever, and clearing them changes nothing. Part 4 turns the game into a game — waves, because the original does exactly this: "Once the screen has been cleared of all asteroids and flying saucers, a new set of large asteroids appears, thus starting the next level" ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))). A score, because the original's rocks pay: "Smaller asteroids are also worth more points." Lives and a game-over state — the original gives you "3–5 lives" and plays "to the last ship lost." And the flying saucer, the first object in the field that *aims at something*, which is new geometry: not "does this circle touch that circle" but "which way is the ship, from where I am."

Before you move on: the bullet-vs-asteroid pass walks *both* arrays backward, but the ship-vs-asteroid pass walks the asteroids forward with a plain `for…of`. Why is forward safe there? (And `split` splices the asteroids from inside the inner bullet loop — say why *that* splice is safe when `j` is walking backward.)

## Exercises

- [ ] **The next wave.** The original starts a new level "Once the screen has been cleared of all asteroids and flying saucers" ([Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game))). Add one line at the end of `update`: `if (asteroids.length === 0) spawnAsteroids(4, 2);` The field now regenerates, and `spawnAsteroids`' 150-px rule keeps the new rocks away from wherever the ship is, not just from the center. (A full wave is 28 rocks — 4 large, 8 medium, 16 small — so a cleared field is 28 successful shots away.)
- [ ] **The 26-rock cap.** Implement the quirk from the design note: in `split`, when the field is already full, produce one piece instead of two. The check belongs after the `splice` (the parent is already gone): if `asteroids.length >= 25`, push a single `makeAsteroid(a.x, a.y, a.size - 1)` instead of the loop. Watch the field stop growing once it hits the cap — the arcade's memory limit, reproduced in two lines.
- [ ] **Debug circles.** In `draw()`, after the asteroid loop, stroke every collision circle: a `ctx.arc(a.x, a.y, a.radius, 0, Math.PI * 2)` for each rock, plus the ship's circle at `SHIP_RADIUS`, and leave the bullets' 2-px circles out (they are the dots). Fly slowly at a rock and watch *where* the hit actually happens — well before the drawn polygons visually touch, because the circles are the physics. Remove the overlay when you're done.
- [ ] **Rocks that bump.** Your field has no rule for two rocks meeting, and the sources I read for this series don't say whether the 1979 game let them collide — so this is your design choice: pass through, or bounce. For a bounce: in `update`, after the movement, compare every pair of asteroids with `hit` (using each one's `radius`); when a pair touches, swap the two velocity vectors and push the rocks apart along the line between their centers so they don't re-collide on the next frame. (Pairs, not indices: `i < j`, no duplicates.)
- [ ] **The real ship hitbox.** Replace the ship's circle with its exact triangle. A point is inside a triangle if it is on the same side of all three edges — for each edge from `p1` to `p2`, the sign of the cross product `(p2.x - p1.x)·(p.y - p1.y) - (p2.y - p1.y)·(p.x - p1.x)` must be the same for all three edges. Do it in the ship's local coordinates (rotate the rock's center by `-ship.angle` around the ship's position first) so the three edge points are the fixed `(0, -15)`, `(-12, 12)`, `(12, 12)`. The difference is subtle but real: a rock grazing the ship's back corner no longer counts.

## Sources

**The original game**

1. [Asteroids (video game) — Wikipedia](https://en.wikipedia.org/wiki/Asteroids_(video_game)) — the level start ("Each level starts with multiple large asteroids drifting across the screen"), the wrap-around rule ("Objects wrap around screen edges … reappears at the bottom and continues moving in the same direction"), the splitting rule ("they break into smaller asteroids that move faster and are more difficult to hit"), the controls ("rotate left and right, fire shots straight forward, and thrust forward"), the objective ("while avoiding colliding with either"), the 26-asteroid cap, and Part 4's setup: the next level ("Once the screen has been cleared … a new set of large asteroids appears"), the scoring ("Smaller asteroids are also worth more points"), and the lives ("The player starts with 3–5 lives upon game start … Play continues to the last ship lost").

**Drawing bullets**

2. [CanvasRenderingContext2D.arc() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/arc) — the method "adds a circular arc to the current sub-path," from `startAngle` to `endAngle` around a center and radius; `0` to `Math.PI * 2` is the full circle (the `counterclockwise` parameter defaults to clockwise, which a full circle never notices).

**JavaScript**

3. [Remainder (%) — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder) — the remainder "returns the remainder left over when one operand is divided by a second operand" and "always takes the sign of the dividend" (`-13 % 5` is `-3`) — the reason `wrap()` needs its `v < 0 ? v + size : v` line.
