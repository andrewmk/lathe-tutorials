# Boids that dodge obstacles

Drop a pillar into the flock's airspace and nothing happens. The cones pass through the stone as if it were smoke — and the model is behaving correctly: nothing in the three Reynolds rules knows the pillar exists. Separation, alignment, and cohesion each look only at flockmates, and a pillar is not a flockmate.

Reynolds knew this. The model documented on his [boids page](http://www.red3d.com/cwr/boids/) was more elaborate than the three-rule version we built in part 01: "A slightly more elaborate behavioral model was used in the early experiments. It included predictive obstacle avoidance and goal seeking. Obstacle avoidance allowed the boids to fly through simulated environments while dodging static objects." The page carries a frame from 1986 of a "simulated boid flock avoiding cylindrical obstacles" — the scene this part puts back into `boids.html`.

The catch: the pipeline you already have cannot carry the fourth behavior. Part 01's flock turns in smooth arcs because every force is capped at `MAX_FORCE = 0.04`. That cap is a feature — for preferences. Separation, alignment, and cohesion can afford to yield to each other; the cap is what makes their interference produce arcs instead of fights. A pillar does not yield. This part adds obstacle avoidance, and it does so by deliberately breaking the one invariant part 01 set up.

> [!RECALL]
> Before continuing: every rule in part 01 ends in `steerToward`. What does `steerToward` do to a rule's desired velocity, and which of its steps is responsible for the flock's smooth arcs?
>
> If you cannot reconstruct both the correction (`desired` at `MAX_SPEED`, minus the current velocity) and the cap (`MAX_FORCE`), stop and re-read part 01's steering-formula section first. This part's central decision is what to do with that cap.

By the end of this part, `boids.html` will have three full-height pillars in the flock's airspace and a fourth force that dodges them — five small edits, one of them a single line — and you will know exactly why that force is the only one in the file that skips the `MAX_FORCE` cap, and what it cannot do.

## What you'll build

The same single file, now with three cylindrical pillars standing in the box — radii 3, 4, and 5, the full height of the box, no way over or under — and the flock streams around them: it splits at each pillar's warning band, flows around both sides, and rejoins behind it. The avoidance is a force with a linear distance ramp, added straight into `boid.acceleration` — the only term in the frame loop that is not capped at `MAX_FORCE`. You will also build the wrong version first, avoidance run through the same capped pipeline as the three rules, because the way it fails is the point of this part.

## Prerequisites

- Part 01 complete: a working `boids.html` with a flock that forms.
- No new math: square roots and the `Vector3` methods part 01 already uses.
- No new three.js beyond `CylinderGeometry`, which gets its own paragraph.

## Three pillars in the flock's airspace

The obstacles are data first. Three plain objects in an array, right after the boids-creation loop (the `for (let i = 0; i < BOID_COUNT; i++) { … }` block) and before `steerToward`:

```js
const obstacles = [
  { position: new THREE.Vector3(-12, 0, 6),  radius: 4 },
  { position: new THREE.Vector3( 10, 0, -8), radius: 5 },
  { position: new THREE.Vector3(  6, 0, 11), radius: 3 },
];
```

Three pillars, three radii, spread across the box. The surfaces do not overlap; the 8-unit warning bands do — a boid caught between two pillars gets pushed by both, and the vector sum does the rest. `position` is the pillar's axis, at `y = 0` in the x/z plane; why the y coordinate is a constant is a simplification, and it gets named in the limitations section below. Plain objects, no class — the same packaging choice as the boids.

Then make them visible. `CylinderGeometry` builds the pillar: `new CylinderGeometry(radiusTop, radiusBottom, height, radialSegments, heightSegments, openEnded, thetaStart, thetaLength)`, per the [reference](https://threejs.org/docs/pages/CylinderGeometry.html), with defaults `1, 1, 1, 32, 1, false, 0, 2π`. We build one unit cylinder and scale it per pillar — one geometry and one material for all three, the shared-resources pattern from part 01:

```js
const pillarGeometry = new THREE.CylinderGeometry(1, 1, 2 * WORLD.y, 24);
const pillarMaterial = new THREE.MeshBasicMaterial({ color: 0x3a4a63 });
for (const ob of obstacles) {
  const pillar = new THREE.Mesh(pillarGeometry, pillarMaterial);
  pillar.scale.set(ob.radius, 1, ob.radius);
  pillar.position.copy(ob.position);
  scene.add(pillar);
}
```

In the complete file this goes after `scene.add(boidMesh);`, at the top of the rendering section, above the `UP`, `dummy`, and `dir` scratch objects. Four details earn their keep:

- The height is `2 * WORLD.y` = 36: the pillar spans y = −18 to +18, the full height of the box. A boid cannot go over or under it — which is why the avoidance math below uses only x and z.
- Scaling a right circular cylinder in x and z by the same factor keeps its cross-section circular, so the 24 radial segments stay fair at any radius.
- `0x3a4a63` is a desaturated blue-gray: darker than the boids' `0x9fd3ff`, so the cones read against the stone.
- Three pillars do not need an `InstancedMesh`. Instancing pays at 150 cones; at 3, three meshes are three draw calls, which is nothing.

Reload the page. The flock forms the way it always does — and then a cone starts passing through stone. Watch for ten seconds: every boid that enters a pillar goes straight through, undisturbed, because nothing in the model says the pillar is there. That is the problem. The first attempt to fix it is tempting, and it is wrong — deliberately, because the way it fails is the point.

## The tempting fix: cap it like everything else

Every force in this file obeys one invariant: it is capped at `MAX_FORCE`. The tempting move is to write avoidance the same way. Two new knobs first, right after the `WORLD` line:

```js
const AVOID_CUSHION = 8;  // warning band beyond each pillar's surface
const AVOID_FORCE = 1.0;  // push at the surface, acceleration units per frame
```

Then the rule, with the other three, after `cohesion`:

```js
function avoidObstacles(boid) {
  const sum = new THREE.Vector3();
  for (const ob of obstacles) {
    const dx = boid.position.x - ob.position.x;
    const dz = boid.position.z - ob.position.z;
    const d2 = dx * dx + dz * dz;
    const trigger = ob.radius + AVOID_CUSHION;
    if (d2 < trigger * trigger && d2 > 0) {
      const d = Math.sqrt(d2);
      const push = AVOID_FORCE * (1 - (d - ob.radius) / AVOID_CUSHION);
      sum.addScaledVector(new THREE.Vector3(dx / d, 0, dz / d), push);
    }
  }
  return sum.clampLength(0, MAX_FORCE);
}
```

Read the ramp, because it is the part we keep:

- It is zero at the edge of the trigger ring (`d = radius + AVOID_CUSHION`), full `AVOID_FORCE` at the surface (`d = radius`), and stronger still inside the surface. Linear in distance — not inverse-distance. Reynolds's force-field model used a force "whose strength is inversely related to distance," which grows without bound as the surface approaches; the linear ramp is bounded by construction, and that boundedness is what keeps a boid spawned inside a pillar from exploding.
- The ramp is also Reynolds's own advice, applied. In the same 1988 notes: "Small steering accelerations applied earlier make for much more robust obstacle avoidance behavior." Eight units of warning band, a small push across most of it, ramping up only as the surface gets close.
- `d2 < trigger * trigger` is part 01's squared-distance trick: no square root until a pillar is actually inside the warning band. The `d2 > 0` guard handles the degenerate case of a boid exactly on the pillar's axis — "away" is undefined at zero distance, the same logic as separation's `d2 > 0`.
- The distance is x/z only, and the push's y-component is `0`: the pillars are full height, so dodging in y is not an option. (Named again in the limitations section.)
- The last line is the only line this part will change. Every force in this file is capped at `MAX_FORCE` — part 01 called the cap the thing that "produces the smooth, sweeping curves." This function obeys the house rule.

One wiring line, in pass 1 of `updateBoids`, after the cohesion line:

```js
    boid.acceleration.add(avoidObstacles(boid));
```

No weight. The ramp already scales the force by urgency; a weight would be a second dial on the same knob.

Reload. The flock still forms. Now look at the pillars: cones slow at the warning band, then push through the stone. In headless runs of this exact code — 150 boids, five random seeds, 1200 frames, measured after a 60-frame spawn warmup — four of the five seeds send boid centers through a pillar surface, up to 4.7 units deep, with up to 235 boid-frames inside a pillar; the fifth seed barely survives, its closest approach 0.5 units of clearance. The failure is probabilistic — some reloads look fine — which makes it worse, not better. The flock treats the pillars like thick fog: it notices them, it tries, and it goes through.

Why the pipeline cannot do the job is a geometry question with a number in it.

## Why the cap cannot save you

Start from the model itself. The frame loop you wrote in part 01 is Reynolds's own simple vehicle model, transcribed. From his GDC '99 paper:

```
steering_force = truncate (steering_direction, max_force)
acceleration = steering_force / mass
velocity = truncate (velocity + acceleration, max_speed)
position = position + velocity
```

"The physics of the simple vehicle model is based on forward Euler integration" — with `mass = 1`, and our `MIN_SPEED` floor as the one local addition. The cap is not an accident of our implementation; the first line of his model is a `max_force` truncation. The question this part answers is what budget each behavior gets.

At the surface, the capped push is `MAX_FORCE = 0.04` acceleration per frame. A boid at `MAX_SPEED = 1.0`. Two numbers follow from those.

The tightest arc a boid can fly at full speed has radius `v²/a = 1.0² / 0.04 = 25` units. A 90° turn is a quarter of that circle — about 39 units of path. The warning band is 8 units wide. A boid that starts avoiding at the edge of the band cannot bend its path anywhere near a 90° arc. Head-on, the same 0.04 as deceleration: the stopping distance from full speed is `v²/(2a) = 1.0 / 0.08 = 12.5` units. The boid gets 8 units of warning. It cannot stop before the surface, and it cannot turn around the pillar. It arrives either way.

This is the regime Reynolds names in that same paper: the steering behaviors "relate to 'fast' motion: running versus crawling. This is an informal notion, but is meant to suggest that the typical velocity of a character is large relative to its maximum turning acceleration. As a result, the steering behaviors must anticipate the future, and take into account eventual consequences of current actions." Our boids are fast by that measure — 1.0 unit per frame against a 0.04 turn budget — and the 8-unit warning band is the anticipation. The cap makes the anticipation useless.

His 1988 SIGGRAPH notes carry the other half of the argument: "The 'steering around obstacles' technique seeks to prevent collisions, and if steering accelerations are not bounded it will usually succeed. But in a more realistic, physically based model, the steering accelerations would be strictly limited, and so collisions become a possibility." Bounded steering is the realistic choice — it is what gives the flock its arcs — and bounded steering makes collision avoidance a separate design problem, not a fourth rule.

The cap cannot be raised. It is a global knob: part 01's third exercise already showed `MAX_FORCE` is what makes the arcs lazy or hooky, and raising it to 1.0 gives every rule — alignment included — a 25× steering budget, and the flock churns into a knot of hooks. (Exercise 4 below does exactly this.) The fix is to give the one behavior that cannot yield a budget of its own.

## The fix: a force, not a rule

The change is one line. Inside `avoidObstacles`, the last line:

```js
  return sum.clampLength(0, MAX_FORCE);   // the tempting version
```

becomes

```js
  return sum;
```

The wiring in pass 1 was already the right line — `boid.acceleration.add(avoidObstacles(boid));` does not change. The whole fix is the cap, removed from one function.

What the uncapped force does:

- At the surface the push is `AVOID_FORCE = 1.0` — 25× the budget of the flocking rules. A boid that cohesion is dragging toward a pillar (at most 0.04) gets a net push of 0.96 away from it. The pillar wins every argument.
- The ramp extends inside the surface, up to `AVOID_FORCE × (1 + radius / AVOID_CUSHION)` on the center line — 1.625 at most, on the radius-5 pillar. The whole force, in its largest case, is one straight line:

```svg
<svg viewBox="0 0 500 240" role="img" aria-label="Linear ramp: push force rises from zero at the 13-unit trigger ring to 1.0 at the pillar surface, then to 1.625 at the axis">
  <line x1="80" y1="20" x2="80" y2="200" stroke="currentColor" stroke-width="1" opacity="0.6"/>
  <line x1="80" y1="200" x2="460" y2="200" stroke="currentColor" stroke-width="1" opacity="0.6"/>
  <line x1="80" y1="150" x2="340" y2="150" stroke="currentColor" stroke-width="1" stroke-dasharray="4 4" opacity="0.4"/>
  <line x1="80" y1="100" x2="180" y2="100" stroke="currentColor" stroke-width="1" stroke-dasharray="4 4" opacity="0.4"/>
  <line x1="180" y1="100" x2="180" y2="200" stroke="currentColor" stroke-width="1" stroke-dasharray="4 4" opacity="0.4"/>
  <line x1="340" y1="200" x2="340" y2="50" stroke="currentColor" stroke-width="1" stroke-dasharray="4 4" opacity="0.4"/>
  <polyline points="80,50 180,100 340,200" fill="none" stroke="#7a66ff" stroke-width="2.5"/>
  <circle cx="80" cy="50" r="3" fill="#7a66ff"/>
  <circle cx="180" cy="100" r="3" fill="#7a66ff"/>
  <circle cx="340" cy="200" r="3" fill="#7a66ff"/>
  <text x="74" y="54" text-anchor="end" font-size="11" font-family="sans-serif" fill="currentColor">1.625</text>
  <text x="74" y="104" text-anchor="end" font-size="11" font-family="sans-serif" fill="currentColor">1.0</text>
  <text x="74" y="154" text-anchor="end" font-size="11" font-family="sans-serif" fill="currentColor">0.5</text>
  <text x="74" y="204" text-anchor="end" font-size="11" font-family="sans-serif" fill="currentColor">0</text>
  <text x="80" y="216" text-anchor="middle" font-size="11" font-family="sans-serif" fill="currentColor">0</text>
  <text x="180" y="216" text-anchor="middle" font-size="11" font-family="sans-serif" fill="currentColor">5</text>
  <text x="340" y="216" text-anchor="middle" font-size="11" font-family="sans-serif" fill="currentColor">13</text>
  <text x="46" y="120" font-size="12" font-family="sans-serif" fill="currentColor" transform="rotate(-90 46 120)">push (acceleration per frame)</text>
  <text x="270" y="232" text-anchor="middle" font-size="12" font-family="sans-serif" fill="currentColor">d, units from the pillar's axis</text>
  <text x="230" y="92" text-anchor="middle" font-size="11" font-family="sans-serif" fill="currentColor" opacity="0.8">surface: full AVOID_FORCE</text>
  <text x="354" y="214" font-size="11" font-family="sans-serif" fill="currentColor" opacity="0.8">trigger ring</text>
</svg>
```

- The velocity clamp in pass 2 is unchanged: `clampLength(MIN_SPEED, MAX_SPEED)`. The bigger budget buys turn rate, not speed — a boid never moves faster than 1.0 unit per frame, and it turns faster than it ever has only near a pillar.
- A boid that spawns inside a pillar gets the inside-surface push — 1.375 to 1.625, which is more than `MAX_SPEED`. It is at full outward speed after one frame, and clears the surface in 3–6 frames, depending on where it spawned. If a cone pops out of a pillar in the first moment after a reload, that is a spawn, not a bug.

> [!DESIGN-NOTE]
> **A reflex, not a preference.**
>
> The three rules are preferences. Each answers "which way would I like to go?" by computing a desired velocity, and `steerToward` converts that desire into a correction — `desired − current` — that goes to zero when the boid is already moving the desired way. The cap then lets the competing corrections blend into arcs instead of fights, and the weights say which preference counts for more when they disagree.
>
> Avoidance answers a different question: "how hard do I have to push right now?" Its magnitude is urgency — distance to the surface — not a correction toward a desired velocity, and there is nothing to weigh it against: a pillar does not negotiate. So it bypasses the whole preference pipeline — no desired, no correction, no cap, no weight — and talks to `boid.acceleration` directly. The pipeline is the right machinery for things a boid would prefer. It is the wrong machinery for things a boid cannot negotiate.

What this buys is real, and Reynolds wrote down exactly where models like this one break.

## What this model is, and what it is not

The model is a specific entry in Reynolds's taxonomy of avoidance techniques, and the same 1988 notes document its failure modes. Four honest limits:

**The center line.** A radial push has no lateral component on the pillar's axis — by symmetry, no radial model can steer a boid aimed exactly at the axis. Reynolds's version: "If we are moving directly toward a wall we need to decide which way to turn to avoid a collision. But the force field from the wall will be directed exactly opposite to our velocity. As a result we will not be turned to either side but rather we will just decelerate to a stop." Our model has the same property on the center line: a lone boid aimed dead-on at a pillar slows, stops about 2–3 units short of the surface (2.6 in the probe, for pillar C), and flies back the way it came. Part 01's `MIN_SPEED = 0.4` floor changes "decelerate to a stop" into "decelerate to the floor, hover, and reverse." In a flock of 150, an exact center-line approach is a measure-zero event, and the neighbors' forces — which do have lateral components — break the symmetry before it matters. Exercise 2 shows the dead spot with one boid.

The taxonomy placement is worth naming. Our push is Reynolds's "steer away from center" — "considers the obstacle to be a point and says to steer in the direction opposite to that center point" — with the two improvements he lists: a distance limit (the trigger ring) and "the steering behavior's strength can be made a function of distance" (the ramp). He grades that model better than the surface force field: "There is no 'dead spot' near the center line of the obstacle." The dead spot shrinks to the center line itself — where the symmetry still wins.

**No tunnel vision.** "The moving object needs to have a sort of 'tunnel vision', generally ignoring obstacles that are off to the side or behind it. This is presumably why horses that work in crowded environments are fitted with blinders." Ours has none: a pillar 5 units behind a boid pushes as hard as one 5 units ahead — the same distance-only perception as part 01's neighborhood. The consequence is visible: the flock gets shoved by pillars it is flying away from, a slight reluctance to hug a pillar's wake. Exercise 5 adds the direction test — one dot product.

**A force, not a constraint.** The push is strong, but bounded, and still a force: a boid arriving at a bad angle can penetrate for a frame or two before it turns around. In the probe runs of this part's exact code (150 boids, five seeds, 1200 frames, after the spawn warmup), the uncapped version never penetrated a surface: closest approach was 4.7–5.8 units of clearance. That is an observation from a test, not a guarantee — faster boids, denser pillars, or a bad spawn can still clip. If clipping is unacceptable for your application, the upgrade is a hard constraint after the move: project the position outside the nearest surface.

**Two dimensions by choice.** The pillars span the full height of the box, so "distance to the pillar" is a distance in the x/z plane, and the push has its y-component set to 0. A pillar you could fly over would need the full 3D distance to a finite cylinder — a different function, and a different behavior: boids that can go over a pillar will.

Reload, and see what all of this actually produces.

## Checkpoint

> [!PREDICT]
> Before you reload: three pillars in the path, one force at 25× the others. In one sentence, what should the flock do when it meets the big one — pillar B, radius 5?

**Run this to verify your work:**

```bash
npx serve .
# then open http://localhost:3000/boids.html
```

Expected output: within a few seconds the stream bends around the pillars. The flock splits at a pillar's warning band, the two halves flow around either side, and they rejoin behind it, leaving a visible hollow around the stone. Near the pillars the arcs are sharper than the flock's usual lazy curves, and the motion stays smooth. After the first moment — a cone or two popping out of a pillar at spawn is normal — no cone is inside a pillar. The console is empty.

**Likely errors:**

- If cones pass straight through the pillars, the last line of `avoidObstacles` is still `return sum.clampLength(0, MAX_FORCE);`. The fix is `return sum;` — one line, and only that one.
- If the flock shreds near the pillars — boids vibrate and ricochet — `AVOID_FORCE` is far above 1.0 (10 or 100, from an earlier experiment). The push at the surface is `AVOID_FORCE` per frame against a velocity clamp of 1.0, so at 10 the force is 10× the velocity every frame and the motion is a strobe. Restore 1.0.
- If you see flat gray pancakes on the floor instead of pillars, the `CylinderGeometry` arguments are in the wrong order: `radiusTop, radiusBottom, height, radialSegments`. `CylinderGeometry(36, 36, 1, 24)` is a 72-unit-wide, 1-unit-tall pancake; the call is `CylinderGeometry(1, 1, 2 * WORLD.y, 24)`, scaled per pillar.
- If a cone is inside a pillar in the first second after a reload, that is not an error: boids spawn uniformly in a box that includes the pillars' interiors, and the ramp ejects them in 3–6 frames.

Before you move on, answer this in two sentences: why does avoidance talk to `boid.acceleration` directly instead of going through `steerToward` like the other three rules, and what specifically would break if you ran it through the same pipeline with a weight of 1.5?

## What's next

The obstacles cost `N × 3` distance checks per frame — 450 at 150 boids — against 67,500 for the boid-boid scans, and the ratio only gets worse as the flock grows: at 10,000 boids the obstacle checks are 30,000 against 300,000,000. The wall part 01 left standing is still the wall, and the pillars did not move it. How do you check neighbors fast when every boid has to see its neighborhood — the spatial binning Reynolds proposed in 2000 — is the next part's subject.

## Exercises

- [ ] Set `AVOID_FORCE` to `0.04` — the same value as `MAX_FORCE` — and reload. The push at the surface is now the same size as any flocking force, and the pillars go back to being fog. Watch what the flock does at each pillar, then restore 1.0. This is the capped failure reached without the clamp line.
- [ ] See the dead spot. Set `BOID_COUNT` to 1 and make the single boid spawn at `(0, 0, 11)` with velocity `(1, 0, 0)` at `MAX_SPEED` — aimed dead-on at pillar C's axis at `(6, 0, 11)`. It slows, stops about 2–3 units short of the surface, and reverses: Reynolds's "decelerate to a stop," with our `MIN_SPEED` floor. On the center line there is no away that has a sideways component. Restore the count and the random spawn when you are done.
- [ ] Build a gate. Move pillar A to `(-12, 0, 0)` and pillar C to `(-12, 0, 14)`: two pillars on a line with a 7-unit gap between their surfaces (A's at z = 4, C's at z = 11). The flock splits, threads the gap, and some of it goes around the outside. Then tighten the gap to 3 — A to `(-12, 0, 2)`, C to `(-12, 0, 12)`, surfaces at z = 6 and z = 9, equal to `SEPARATION_DISTANCE` — and describe what a gap boids cannot fit through side by side does to the flow.
- [ ] Raise the whole flock's steering budget. Restore the tempting version — `return sum.clampLength(0, MAX_FORCE);` — and raise `MAX_FORCE` from 0.04 to 0.5. Now the capped avoidance has a 1-unit stopping distance and a 2-unit turn radius, and the pillars get dodged. But the flocking rules got a 12.5× budget too: the arcs turn into sharp hooks and the flock churns. Restore `MAX_FORCE` to 0.04 and `return sum;`. The cap is a global knob; raising it fixes avoidance and wrecks the flock, while the direct force fixes only avoidance.
- [ ] Give the flock blinders. Inside `avoidObstacles`, compute the vector from the boid to the pillar — `new THREE.Vector3(ob.position.x - boid.position.x, 0, ob.position.z - boid.position.z)` — and skip the pillar when the boid is moving away from it: `if (boid.velocity.dot(toOb) < 0) continue;` (`Vector3.dot` is in part 01's reference list). A pillar behind the boid stops pushing; a pillar beside it (dot near 0) still does. Watch what changes when the flock streams away from a pillar, and say what the test still does not filter.

## Sources

**Algorithms**

1. [Craig Reynolds — Boids](http://www.red3d.com/cwr/boids/) — the 1986 model already included "predictive obstacle avoidance": "allowed the boids to fly through simulated environments while dodging static objects," with a frame of a flock avoiding cylindrical obstacles.
2. [Craig Reynolds — Not Bumping Into Things](http://www.red3d.com/cwr/nobump/nobump.html) — the SIGGRAPH '88 notes on obstacle avoidance: the force field and its "dead spot," "tunnel vision" and directional discrimination, steer-away-from-center with a distance limit and distance-weighted strength, and why bounded steering makes collisions possible.
3. [Craig Reynolds — Steering Behaviors For Autonomous Characters](http://www.red3d.com/cwr/steer/gdc99/) — the GDC '99 paper: the simple vehicle model (max force, max speed, forward Euler integration) that this file implements, and the "fast motion" argument that steering must anticipate the future.

**three.js (r185 documentation)**

4. [CylinderGeometry](https://threejs.org/docs/pages/CylinderGeometry.html) — the pillar's shape: `radiusTop`, `radiusBottom`, `height`, `radialSegments`, and their defaults.
5. [Vector3](https://threejs.org/docs/pages/Vector3.html) — the force arithmetic: `addScaledVector`, `clampLength`, and `dot` for the tunnel-vision exercise.
