# Simple Physics in the Browser, Part 3: A Jelly Cube You Can Grab and Throw

> [!RECALL]
> Before reading on: in Part 2's `step()`, every spring force is applied to exactly two masses, equal and opposite. What can change the total momentum of the 192-mass sheet as a whole?
>
> If you can't reconstruct it: the spring forces cancel in pairs, so only the external ones are left in the balance — gravity, and the pins along the top edge.

Part 1 promised "a stretchy bouncy sheet, a wobbling jelly cube, a square of cloth that drapes over a round table." The sheet is done. This part is the jelly cube: a soft solid, the same pile of springs pretending to be matter — except that nothing is pinned.

Two things change in this part. The first is the drawing: no more 2D canvas. The scene renders into a WebGL viewport with [Three.js](https://threejs.org/), whose API surface is new but small — a renderer, a scene, a camera, lines, points — and each piece gets introduced when it first appears. The second change is quieter, and it's the one the physics cares about: Part 2's sheet hung from a ceiling, and the ceiling was the external force that let the model get away with everything. Part 3's cube is free. Nothing holds it up, nothing absorbs its momentum, and nothing slows it down except the springs themselves.

## What you'll build

A single file, `jelly.html`:

- A 5 × 5 × 5 lattice of 125 masses, 0.1 kg each, 2.4 m on a side.
- 780 springs: 300 structural (axis-aligned) and 480 shear (face diagonals).
- Rendered with Three.js: the springs as lines in two colors, the masses as points, a grid standing in for the floor.
- A free body. The cube starts 2 m in the air, falls, splats, bounces, rings, and settles. There are no pins anywhere in the file.
- The mouse: press near a mass to grab it, drag it, and on release it keeps the velocity the drag was giving it (clamped to a 40 m/s flick). Dragging empty space orbits the camera. The camera follows the cube's center of mass, so a hard throw can't send it out of view.
- The readout: mass count, spring count, cube height, maximum spring strain, simulation time.

## Prerequisites

- All of Part 1: Hooke's law, the semi-implicit Euler update (force, then velocity, then position with the new velocity), and the `requestAnimationFrame` clock with its clamped `dt`.
- All of Part 2: the two-list architecture (masses, springs), the three-pass step, shear springs and why they exist, and the mouse grab-and-flick.
- No Three.js experience. This part introduces the minimum API as it appears. A little familiarity with ES module `import` helps; the import-map section below covers what the browser needs.

## Nothing is pinned

Part 2's sheet was pinned along its top edge. The pins are external forces, and they did two jobs: holding the sheet up, and absorbing whatever momentum its motion built up. The readout never noticed, because a pinned system has somewhere to put it all.

Take the pins off and both jobs become visible.

The good news first. Spring forces cancel in pairs — that's the RECALL answer, now in force — so the total force on the 125-mass cube is just the external stuff: gravity, and the floor once the cube lands. Wikipedia defines the [center of mass](https://en.wikipedia.org/wiki/Center_of_mass) as "the unique point at any given time where the weighted relative position of the distributed mass sums to zero." For equal masses it's just the average of the positions, and Newton's second law applies there: the center of mass of the falling cube falls exactly like a single dropped point, at 9.81 m/s², no matter how the cube jiggles on the way down.

The bad news changes one line of Part 2. Part 2's damping was per-mass drag: `(fx[i] - C * vx[i])`, a force against each mass's velocity, as if each mass moved through air. In a pinned sheet that was invisible — the pins absorbed the momentum anyway, and the drag only decided how fast the jiggle died. In a free cube it's a visible lie: a thrown cube would slow down in empty space, and its momentum would vanish into a floor the model doesn't have. So the damping moves into the springs. Each spring damps only the relative motion of its two ends, along its own axis, equal and opposite — the `C * rel` term in the step below. Total momentum now changes only under external forces, and a thrown cube in zero gravity drifts forever at exactly constant velocity. The zero-gravity exercise at the end makes that visible.

The floor is the other new external force, and the series' first contact. It's deliberately simple: a flat plane at y = 0, and a mass that penetrates it by `pen` meters gets a spring force back up, `K_FLOOR · pen`, plus a dashpot that bleeds off downward speed, `C_FLOOR`. Part 4's round table will force this trick to grow — contact with a curved surface is not a one-line `if (pen > 0)` — but a flat plane is enough for the cube.

## The stage: renderer, scene, camera

The file keeps the single-file pattern, with one new thing in the `<head>`: an import map. Three.js ships only as an ES module — `import` from a bare package name — and a browser doesn't know what a package name means without a map. The [installation guide](https://threejs.org/manual/en/installation.html) says it directly: "We imported code from 'three' (an npm package) in main.js, and web browsers don't know what that means. In index.html we'll need to add an import map defining where to get the package."

Here's the file so far — the complete HTML, the import map, and the stage. Save it as `jelly.html`.

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Jelly cube, Part 3</title>
<style>
  body { margin: 0; background: #101418; overflow: hidden; }
  #readout { position: absolute; top: 12px; left: 12px; color: #9aa4b2; font: 14px monospace; }
</style>
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.185.1/build/three.module.js"
  }
}
</script>
</head>
<body>
<div id="readout"></div>
<script type="module">
import * as THREE from "three";

// ---- The stage: renderer, scene, camera ---------------------------------
// A WebGLRenderer is a WebGL 2 viewport. We ask it for a canvas the size of
// the window, hand the canvas to the page, and everything we add to the scene
// is drawn into it by renderer.render().
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// The scene is a container for everything that can be drawn.
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x101418);

// The camera is where the eye is. A PerspectiveCamera projects 3D to the 2D
// viewport with a vertical field of view in degrees (here 50) and a near/far
// clipping range. It looks at `target` from a point on a sphere of `radius`
// around it: `theta` is the angle around the y axis, `phi` is the angle down
// from straight up.
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 200);
const target = new THREE.Vector3(0, 1.0, 0);
let theta = 0.25, phi = 1.4, radius = 14;
function positionCamera() {
  camera.position.set(
    target.x + radius * Math.sin(phi) * Math.sin(theta),
    target.y + radius * Math.cos(phi),
    target.z + radius * Math.sin(phi) * Math.cos(theta)
  );
  camera.lookAt(target);
}
positionCamera();

// A GridHelper is a flat grid of lines on the x/z plane at y = 0. Twelve
// meters across, twelve cells, it stands in for the floor the cube lands on.
const grid = new THREE.GridHelper(12, 12);
scene.add(grid);
// One-shot draw so you can check the stage. The loop added at the end of
// this part draws every frame; these calls exist to check progress.
renderer.render(scene, camera);
```

The import map has one entry: the package name `three` maps to a pinned version, 0.185.1, served by jsDelivr. Pin the version, and keep every import on the same CDN and same version — the installation guide's warning: "Import all dependencies from the same version of three.js, and from the same CDN. Mixing files from different sources may cause duplicate code to be included, or even break the application in unexpected ways."

Then, in order, the four objects the [fundamentals](https://threejs.org/manual/en/fundamentals.html) page describes: "There is a Renderer. This is arguably the main object of three.js. You pass a Scene and a Camera to a Renderer and it renders (draws) the portion of the 3D scene that is inside the frustum of the camera as a 2D image to a canvas."

- **Renderer.** "This renderer uses WebGL 2 to display scenes" ([WebGLRenderer](https://threejs.org/docs/pages/WebGLRenderer.html)). It builds a canvas the size of the window, and the page keeps it.
- **Scene.** The container for everything drawn, with a background color.
- **Camera.** A [PerspectiveCamera](https://threejs.org/docs/pages/PerspectiveCamera.html) with a 50° field of view — "The vertical field of view, from bottom to top of view, in degrees." `near` and `far` "represent the space in front of the camera that will be rendered. Anything before that range or after that range will be clipped (not drawn)." By default the camera looks down its −Z axis with +Y up (fundamentals), so the scene's floor is the x/z plane and up is +y — the opposite of the canvas, where y was down.
- **Placement.** The camera sits on a sphere of radius `radius` around a point `target`; `theta` is the angle around the y axis, `phi` the angle down from straight up. `positionCamera()` computes that position and calls `camera.lookAt(target)`. The mouse section mutates `theta` and `phi`, and the clock mutates `target`, so they're plain variables now.

The [grid](https://threejs.org/docs/pages/GridHelper.html) is 12 m wide, 12 cells, in the x/z plane at y = 0: "The helper is an object to define grids. Grids are two-dimensional arrays of lines." It stands in for the floor the physics uses. There's also an empty `#readout` div up top; the clock section puts text in it.

**Run it.** Open it directly, or serve the folder (`python3 -m http.server`) and visit the page. You get a dark room and a grid. The single cross-origin import is what keeps the double-click alive; the server is the habit that survives any future addition.

## The cube, standing still

Now the physics data and the drawing objects.

The constants first. Three carry over unchanged from Part 2: `SPACING = 0.6`, `MASS = 0.1`, `G = 9.81`. Two change, and three are new:

- `K_STRUCT = 100`, not Part 2's 32. Part 2's sheet was pinned and never had to survive an impact. The cube is free: it falls 2 m and lands at 6.2 m/s. At K = 32 the impact tangles the lattice — measured: 91% peak strain, and the gaps between layers go negative, which means adjacent layers have crossed and the "cube" has folded through itself. At K = 100 the same drop peaks at 31.7% and the layers never cross; a 5 m drop stays clean. The ratio is the one Part 2's "A square with no center" established from the Eurographics tutorial's stiffness hierarchy — "very large constants" for structural springs, "small values" for shear: 50 = 100/2, the same 2:1 as 16 = 32/2.
- `C = 1.0`, now per-spring and pairwise, per "Nothing is pinned" (Part 2's was 0.5 per mass, against the world).
- The floor: `FLOOR = 0`, `RADIUS = 0.05` — each mass has a 5 cm contact "skin", so the floor acts at y = 0.05, not 0 — `K_FLOOR = 980`, `C_FLOOR = 2`.
- `BOTTOM = 2.0`: starting height of the bottom row.
- `STEP_DT = 1/240`: why that is, two sections down. Trust it for now.

The lattice. `i3(a, b, c)` packs a 3D index into a flat array index, the same trick Part 2 used for `idx`. The masses sit at `(a − 2, y, c − 2)` in x and z, so the cube is centered on the y axis, with the bottom row at y = 2.0. From each mass, three structural springs reach to its right, up, and forward: 3 × 4 × 5 × 5 = 300. Then the shear: both face diagonals, in all three face orientations — Part 2's "square with no center" in 3D, since without them any face can shear into a rhombus for free. 3 orientations × 16 faces × 2 diagonals × 5 planes = 480. Total 780.

The drawing. Springs are lines: "A series of lines drawn between pairs of vertices" ([LineSegments](https://threejs.org/docs/pages/LineSegments.html)). One `LineSegments` per family, two vertices per spring, so each family keeps its own color, the way the sheet colored its two spring families. `makeLines` wraps a `Float32Array` of positions in a BufferGeometry and a [BufferAttribute](https://threejs.org/docs/pages/BufferAttribute.html); the physics copies into that array every frame, and the attribute's `needsUpdate` is the flag the GPU needs: "Flag to indicate that this attribute has changed and should be re-sent to the GPU. Set this to `true` when you modify the value of the array." Masses are points: "A class for displaying points or point clouds" ([Points](https://threejs.org/docs/pages/Points.html)), with `sizeAttenuation: false` — "Specifies whether size of individual points is attenuated by the camera depth (perspective camera only)" — so the points stay 7 px whatever the camera distance. `updateGeometry()` is the one function that copies physics positions into the drawing buffers and sets the flags.

```js
// ---- The cube as data: 125 masses, 780 springs --------------------------
const N3 = 5;              // 5 masses per edge
const N = N3 ** 3;         // 125 masses
const SPACING = 0.6;       // m between neighbors (cube edge = 4 * 0.6 = 2.4 m)
const MASS = 0.1;          // kg per mass
const G = 9.81;            // m/s^2, down
const K_STRUCT = 100;      // N/m, axis-aligned springs
const K_SHEAR = 50;        // N/m, face-diagonal springs
const C = 1.0;             // N*s/m, pairwise damping per spring
const FLOOR = 0;           // m, height of the floor plane
const RADIUS = 0.05;       // m, contact "skin" of each mass
const K_FLOOR = 980;       // N/m, how hard the floor pushes back per meter of penetration
const C_FLOOR = 2;         // N*s/m, how the floor bleeds off downward speed
const BOTTOM = 2.0;        // m, starting height of the bottom row
const STEP_DT = 1 / 240;   // s, fixed physics step

// One flat array per coordinate and per velocity component, indexed 0..N-1.
// The force arrays are scratch space, reset every step.
const x = new Float64Array(N), y = new Float64Array(N), z = new Float64Array(N);
const vx = new Float64Array(N), vy = new Float64Array(N), vz = new Float64Array(N);
const fx = new Float64Array(N), fy = new Float64Array(N), fz = new Float64Array(N);
const springs = [];

function i3(a, b, c) { return (a * N3 + b) * N3 + c; }

// A spring remembers its two endpoints, its stiffness, and its rest length
// (measured from the current positions at build time).
function addSpring(a, b, k) {
  springs.push({ a, b, k, rest: Math.hypot(x[b] - x[a], y[b] - y[a], z[b] - z[a]) });
}

// Lay the 125 masses out in a 5x5x5 lattice, centered on the y axis, with the
// bottom row at y = BOTTOM.
for (let a = 0; a < N3; a++)
  for (let b = 0; b < N3; b++)
    for (let c = 0; c < N3; c++) {
      const i = i3(a, b, c);
      x[i] = (a - 2) * SPACING;
      y[i] = BOTTOM + b * SPACING;
      z[i] = (c - 2) * SPACING;
    }

// Structural springs: the axis-aligned edges. From each mass, reach right,
// up, and forward. 3 * 4 * 5 * 5 = 300 of them.
for (let a = 0; a < N3; a++)
  for (let b = 0; b < N3; b++)
    for (let c = 0; c < N3; c++) {
      const i = i3(a, b, c);
      if (a + 1 < N3) addSpring(i, i3(a + 1, b, c), K_STRUCT);
      if (b + 1 < N3) addSpring(i, i3(a, b + 1, c), K_STRUCT);
      if (c + 1 < N3) addSpring(i, i3(a, b, c + 1), K_STRUCT);
    }

// Shear springs: both diagonals of every face, in all three orientations.
// Each face is a square of four masses; its two diagonals are the springs that
// cost something when the face is sheared into a rhombus. 480 of them.
for (let a = 0; a < N3; a++)
  for (let b = 0; b < N3; b++)
    for (let c = 0; c < N3; c++) {
      const i = i3(a, b, c);
      if (a + 1 < N3 && b + 1 < N3) { // face in the a/b plane, fixed c
        addSpring(i, i3(a + 1, b + 1, c), K_SHEAR);
        addSpring(i3(a + 1, b, c), i3(a, b + 1, c), K_SHEAR);
      }
      if (a + 1 < N3 && c + 1 < N3) { // face in the a/c plane, fixed b
        addSpring(i, i3(a + 1, b, c + 1), K_SHEAR);
        addSpring(i3(a + 1, b, c), i3(a, b, c + 1), K_SHEAR);
      }
      if (b + 1 < N3 && c + 1 < N3) { // face in the b/c plane, fixed a
        addSpring(i, i3(a, b + 1, c + 1), K_SHEAR);
        addSpring(i3(a, b + 1, c), i3(a, b, c + 1), K_SHEAR);
      }
    }

// ---- The cube as geometry: springs are lines, masses are points ----------
// A LineSegment is "a series of lines drawn between pairs of vertices," so a
// LineSegments object with 2 vertices per spring draws every spring at once.
// We keep the two families separate so they can have different colors, the
// way the 2D sheet colored structural and shear springs differently.
function makeLines(list, color) {
  const positions = new Float32Array(list.length * 6); // 2 vertices * 3 coords per spring
  const geo = new THREE.BufferGeometry();
  // A BufferAttribute wraps the flat array; the position attribute holds the
  // vertices. We copy the physics into `positions` every frame and flag the
  // attribute with needsUpdate so it is re-sent to the GPU.
  geo.setAttribute("position", new THREE.BufferAttribute(positions, 3));
  const lines = new THREE.LineSegments(geo, new THREE.LineBasicMaterial({ color }));
  lines.userData.positions = positions;
  lines.userData.list = list;
  scene.add(lines);
  return lines;
}
const structSprings = springs.filter(s => s.k === K_STRUCT);
const shearSprings = springs.filter(s => s.k === K_SHEAR);
const structLines = makeLines(structSprings, 0x8fb3ff);
const shearLines = makeLines(shearSprings, 0x3a4552);

// A Points object displays one point per vertex. sizeAttenuation: false keeps
// the points a fixed number of pixels whatever the camera distance.
const massPositions = new Float32Array(N * 3);
const massGeo = new THREE.BufferGeometry();
massGeo.setAttribute("position", new THREE.BufferAttribute(massPositions, 3));
const massPoints = new THREE.Points(massGeo, new THREE.PointsMaterial({ color: 0xe8ecf0, size: 7, sizeAttenuation: false }));
scene.add(massPoints);

// Copy the physics positions into the drawing buffers.
function updateGeometry() {
  for (const lines of [structLines, shearLines]) {
    const p = lines.userData.positions, list = lines.userData.list;
    let o = 0;
    for (const s of list) {
      p[o++] = x[s.a]; p[o++] = y[s.a]; p[o++] = z[s.a];
      p[o++] = x[s.b]; p[o++] = y[s.b]; p[o++] = z[s.b];
    }
    lines.geometry.attributes.position.needsUpdate = true;
  }
  for (let i = 0; i < N; i++) {
    massPositions[i * 3] = x[i];
    massPositions[i * 3 + 1] = y[i];
    massPositions[i * 3 + 2] = z[i];
  }
  massGeo.attributes.position.needsUpdate = true;
}
updateGeometry();
renderer.render(scene, camera);
```

**Run it.** The cube, frozen in the air above the grid: two colors of lines, one cloud of points. Nothing moves yet — no clock — and the step is next.

## The step: the same three passes, plus a floor

The step is the one Part 1 earned, with three coordinates and a floor. Three passes, same order:

1. Seed every mass's force with gravity. Down is −y now — the canvas's y was down; the world's y is up.
2. Each spring: the Hooke force plus the pairwise damping, applied to its two ends equal and opposite.
3. The soft floor, then the semi-implicit update: velocity first, then position with the new velocity.

```js
// ---- The step: forces, then semi-implicit Euler --------------------------
function step(dt) {
  // Pass 1: seed every mass's force with gravity (down, so negative y).
  for (let i = 0; i < N; i++) { fx[i] = 0; fy[i] = -MASS * G; fz[i] = 0; }

  // Pass 2: each spring acts on exactly the two masses at its ends, equal and
  // opposite. Hooke's law plus pairwise damping: the spring also resists the
  // relative motion of its two ends along its own axis.
  for (const s of springs) {
    const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a], dz = z[s.b] - z[s.a];
    const len = Math.hypot(dx, dy, dz);
    const ux = dx / len, uy = dy / len, uz = dz / len;
    const f = s.k * (len - s.rest);
    const rel = (vx[s.b] - vx[s.a]) * ux + (vy[s.b] - vy[s.a]) * uy + (vz[s.b] - vz[s.a]) * uz;
    const F = f + C * rel;
    fx[s.a] += F * ux; fy[s.a] += F * uy; fz[s.a] += F * uz;
    fx[s.b] -= F * ux; fy[s.b] -= F * uy; fz[s.b] -= F * uz;
  }

  // Pass 3: the soft floor, then the semi-implicit Euler update Part 1 earned:
  // velocity first, then position, with the new velocity.
  for (let i = 0; i < N; i++) {
    // A mass that has penetrated the floor by `pen` gets a spring force back
    // up proportional to pen, plus a dashpot term that bleeds off its
    // downward speed. Together they approximate a soft, slightly bouncy floor.
    const pen = FLOOR + RADIUS - y[i];
    if (pen > 0) {
      fy[i] += K_FLOOR * pen;
      if (vy[i] < 0) fy[i] -= C_FLOOR * vy[i];
    }
    vx[i] += fx[i] / MASS * dt;
    vy[i] += fy[i] / MASS * dt;
    vz[i] += fz[i] / MASS * dt;
    x[i] += vx[i] * dt;
    y[i] += vy[i] * dt;
    z[i] += vz[i] * dt;
  }
}
```

The floor's placement deserves a word. It's a force assembled before the velocity update, not a position clamp after it. A clamp (`if (y < floor) y = floor;`) looks fine, but the bottom row never rests — it hops one step up and one step down forever, because the clamp and the spring never agree on a resting place. As a force, penetration, push-back, and velocity all balance, and the bottom row has a real rest.

> [!PREDICT]
> The top row's resting height, before you run it. In each column the bottommost spring carries the four masses above it: 4 × 0.1 × 9.81 / 100 = 39.2 mm of compression. The next spring carries three (29.4 mm), then two (19.6 mm), then one (9.8 mm). The column squashes 98.1 mm in total, so the top row should rest at 0.05 + 2.4 − 0.0981 = 2.352 m — the 0.05 is the bottom mass's contact radius, which is what the floor actually supports. The measured value is in the checkpoint.

## Why the step shrinks to 1/240

Part 2's sheet ran fine at the clock's native ~1/60 s step. The cube can't. Here's the shortest stability argument in this series, and it comes from Part 1's one spring.

Semi-implicit Euler on a spring of angular frequency ω = √(k/m):

```text
v' = v − ω²·dt·x
x' = x + v'·dt
```

Substitute the first line into the second, and (x, v) maps to (x', v') by a 2×2 matrix:

```text
[1 − ω²·dt²    dt]
[−ω²·dt        1 ]
```

Wikipedia writes it out for the general iteration, "and since the determinant of the matrix is 1 the transformation is area-preserving." A unit-determinant map keeps orbits bounded exactly while its trace stays between −2 and 2, which here reduces to ω·dt < 2. Below that limit the orbits are the "stable periodic orbits (for sufficiently small step size)" Wikipedia describes; above it, one eigenvalue leaves the unit circle and the amplitude grows geometrically. The spring pumps itself, and Part 1's "the numerical solution grows very large" is the same failure.

So the step is capped by the system's fastest mode: dt < 2/ω_max.

For the cube, ω_max isn't guessable — it's a property of all 780 springs acting together. The honest way to get it: nudge each of the 125 masses one coordinate at a time, read off the resulting forces, and build the 375 × 375 stiffness matrix (3 coordinates × 125 masses). Its largest eigenvalue is 692.9 N/m, so ω_max = √(692.9 / 0.1) = 83.2 rad/s, 13.25 Hz. That's the lattice's fastest ring — adjacent layers sliding against each other as fast as the springs allow.

The undamped limit is dt < 2/83.2 = 0.0240 s — 41.6 steps per second. The clock's native 1/60 s step sits inside it with a 1.44× margin to spare; 1/40 s is outside. The measurements agree: the undamped cube is stable at 1/60 s steps and blows up at 1/40.

Damping moves the practical limit. With C = 1.0, the 1/60 s step the undamped cube survives now explodes — the dashpot term adds to the step's effective stiffness, and the margin was thin. 1/80 s survives. The file takes 1/240 s: ω·dt = 0.347, 5.8× inside the theoretical undamped limit and three times finer than the largest step measured stable. The cost of that safety is four substeps per 60 fps frame.

**The accumulator callback.** Part 1 parked the fixed-timestep accumulator with a design note: "One mass never comes close to that. Hundreds of masses will. When this series gets there, the accumulator comes back." Part 2 measured the step (24 µs), ruled the accumulator out, and said it comes back in Part 4, when contact makes steps expensive. It comes back one part early, for a reason nobody predicted: not cost, stability. The mechanism is the one Part 1 sketched: the frame clock banks its actual `dt`, and the physics spends the bank in fixed 1/240 s steps, however many that takes.

The cost story is unchanged. A 3D step takes 40 µs — more springs, three coordinates, a floor check — against Part 2's 24 µs for 2D. Four substeps per frame is 160 µs, about 1% of a 16.7 ms frame. The accumulator is a stability device here. It's also cheap enough to be a cost device later.

## The mouse: pick, drag, orbit

```js
// ---- The mouse: grab a mass, or orbit the camera -------------------------
let grabbed = -1;
let orbiting = false;
const MAX_FLICK = 40;   // m/s, a hard flick, not a rocket launch
const mouse = { x: 0, y: 0 };
const raycaster = new THREE.Raycaster();
const ndc = new THREE.Vector2();
const dragPlane = new THREE.Plane();
const dragPos = new THREE.Vector3();   // where the grabbed mass should go
const dragPrev = new THREE.Vector3();  // where it was last frame
const tmpV = new THREE.Vector3();

// Normalized device coordinates are the camera's own -1..1 screen: x runs
// left to right, y runs bottom to top, both from -1 to 1. e.clientX/Y are
// window pixels, so this converts them.
function toNDC(e) {
  ndc.x = (e.clientX / window.innerWidth) * 2 - 1;
  ndc.y = -(e.clientY / window.innerHeight) * 2 + 1;
}

window.addEventListener("mousedown", (e) => {
  mouse.x = e.clientX; mouse.y = e.clientY;
  // Picking: project every mass to the screen, keep the nearest within 20 px.
  let best = -1, bestD = 20;
  for (let i = 0; i < N; i++) {
    tmpV.set(x[i], y[i], z[i]).project(camera);
    const sx = (tmpV.x * 0.5 + 0.5) * window.innerWidth;
    const sy = (-tmpV.y * 0.5 + 0.5) * window.innerHeight;
    const d = Math.hypot(e.clientX - sx, e.clientY - sy);
    if (d < bestD) { bestD = d; best = i; }
  }
  if (best >= 0) {
    grabbed = best;
    // The drag plane faces the camera and passes through the grabbed mass.
    // Dragging moves the mass within this plane, which is what "moving the
    // cursor" means in 3D.
    tmpV.set(x[best], y[best], z[best]);
    dragPos.copy(tmpV);
    dragPrev.copy(tmpV);
    const normal = new THREE.Vector3();
    camera.getWorldDirection(normal);
    dragPlane.setFromNormalAndCoplanarPoint(normal, tmpV);
  } else {
    orbiting = true;
  }
});

window.addEventListener("mousemove", (e) => {
  const dx = e.clientX - mouse.x, dy = e.clientY - mouse.y;
  mouse.x = e.clientX; mouse.y = e.clientY;
  if (orbiting) {
    // Dragging empty space swings the camera around the target.
    theta -= dx * 0.005;
    phi = Math.max(0.15, Math.min(1.5, phi - dy * 0.005));
    positionCamera();
  } else if (grabbed >= 0) {
    // Cast a ray from the camera through the cursor; where it crosses the
    // drag plane is where the mass goes.
    toNDC(e);
    raycaster.setFromCamera(ndc, camera);
    raycaster.ray.intersectPlane(dragPlane, dragPos);
  }
});

window.addEventListener("mouseup", () => {
  grabbed = -1;    // the mass keeps the velocity the drag last gave it: the flick
  orbiting = false;
});
```

The three handlers, in order.

**Picking** is Part 2's, with a projection step added. The sheet picked by pixel distance: the cursor was a 2D point and the masses were drawn at 2D points. In 3D the masses live in 3D, so each is first projected to the screen: `vector.project(camera)` "Projects this vector from world space into the camera's normalized device coordinate (NDC) space" ([Vector3](https://threejs.org/docs/pages/Vector3.html)) — NDC is the −1..1 screen the raycaster also speaks. From NDC to pixels is two arithmetic operations, and picking is the same nearest-within-20-px test as Part 2.

**Dragging** is where 2D and 3D genuinely differ. In the sheet, "move the cursor" had a meaning: the cursor was in the sheet's plane. In 3D, the cursor is a line of sight — every point on that ray is equally "under the cursor." The file resolves the ambiguity with a plane: the one facing the camera, through the grabbed mass. `camera.getWorldDirection(normal)` gives the look direction, and `plane.setFromNormalAndCoplanarPoint(normal, massPos)` makes the plane — the [Plane](https://threejs.org/docs/pages/Plane.html) docs are literal: "Sets the plane from the given normal and coplanar point (that is a point that lies onto the plane)." On every mousemove, the [raycaster](https://threejs.org/docs/pages/Raycaster.html) recomputes "a new origin and direction for the internal ray" from the cursor's NDC (X and Y "should be between -1 and 1"), and `ray.intersectPlane(plane, out)` gives the mass its new position. The mass slides on the camera-facing plane — the motion the cursor most plausibly meant.

**Flicking** comes from the same place as in Part 2: not from the mousemove event (which carries no time), but from comparing where the drag target is this frame to where it was last frame. `dragPos` and `dragPrev` are the 3D version of Part 2's `mouse.x`/`mouse.px`. On release, the grabbed mass keeps that velocity — clamped by total speed, not per component: clamping each axis at 40 m/s would allow 40√3 ≈ 69 m/s at a diagonal, so the clamp scales the whole vector to 40.

**Orbiting** is the empty-space case: no mass within 20 px, so the press orbits the camera. Horizontal drag changes `theta`, vertical changes `phi` — the spherical coordinates the stage section set up — at 0.005 radians per pixel, with `phi` clamped to [0.15, 1.5] so you can't flip through the floor or the zenith.

One more thing the mouse section sets up, whose code lives in the clock: when you throw the cube hard, it drifts — that's the physics, and nothing in the file is going to stop it. The camera follows, and that's where the stage section's `target` earns its keep.

The handlers are in, but nothing moves yet — there's no clock. Same as Part 2: the mouse section comes before the clock, and the page stays still until the next section.

## The clock starts the machine

The readout is a DOM div, not canvas text: the canvas is a WebGL viewport now, and text belongs in the HTML. `#readout` was in the file from the start.

The clock is the accumulator, with two clamps:

- `dt = Math.min(dt, 0.1)`: after a tab switch, never try to make up more than 0.1 s in one frame — that's 24 substeps. Part 1's clamp limited a single step; this one limits a frame's worth of physics.
- `STEP_DT = 1/240`: the step itself, per the stability section.

`frame()` runs the accumulator loop, then the per-frame housekeeping, in this order:

1. **Steps.** `while (accumulator >= STEP_DT) { step(STEP_DT); ... }` — the physics.
2. **The grab.** If a mass is grabbed, its velocity is the drag's motion this frame — `dragPos` minus `dragPrev`, divided by the frame's `dt` — clamped to 40 m/s total, and its position is the drag target. The springs still pull on it during the steps; this overrides the result, which is what "held" means.
3. **The camera follow.** When nothing is grabbed, `target` moves 10% of the way toward the cube's center of mass per frame — a fraction of a second to catch up — and `positionCamera()` re-aims. While a mass is grabbed the camera holds still: the drag plane faces the camera, and a moving camera would slide the plane out from under the cursor.
4. **The draw.** `updateGeometry()` copies physics into the buffers, `updateReadout()` fills the div, `renderer.render(scene, camera)` draws, and `requestAnimationFrame(frame)` schedules the next one.

```js
// ---- The clock: fixed timestep, accumulator, readout ---------------------
const readout = document.getElementById("readout");
let accumulator = 0, last = null, t = 0;

function updateReadout() {
  let minY = Infinity, maxY = -Infinity, maxStrain = 0;
  for (let i = 0; i < N; i++) {
    if (y[i] < minY) minY = y[i];
    if (y[i] > maxY) maxY = y[i];
  }
  for (const s of springs) {
    const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a], dz = z[s.b] - z[s.a];
    const e = Math.abs(Math.hypot(dx, dy, dz) - s.rest) / s.rest;
    if (e > maxStrain) maxStrain = e;
  }
  readout.textContent =
    `${N} masses · ${springs.length} springs · height ${(maxY - minY).toFixed(3)} m · max strain ${(maxStrain * 100).toFixed(1)}%   t = ${t.toFixed(1)} s`;
}

function frame(now) {
  if (last === null) last = now;
  let dt = (now - last) / 1000;
  last = now;
  dt = Math.min(dt, 0.1);   // never try to make up more than 0.1 s in one frame
  accumulator += dt;
  while (accumulator >= STEP_DT) {
    step(STEP_DT);
    t += STEP_DT;
    accumulator -= STEP_DT;
  }
  if (grabbed >= 0) {
    // The grabbed mass is pinned to the cursor: its velocity is the drag's
    // motion this frame, and its position is the drag target. The springs
    // still pull on it during the steps; this overrides the result, which is
    // what "held" means. On release it keeps this last velocity, limited to
    // MAX_FLICK so a wild mouse move becomes a hard flick, not a rocket.
    const inv = 1 / Math.max(dt, 1e-6);
    let dvx = (dragPos.x - dragPrev.x) * inv;
    let dvy = (dragPos.y - dragPrev.y) * inv;
    let dvz = (dragPos.z - dragPrev.z) * inv;
    const speed = Math.hypot(dvx, dvy, dvz);
    if (speed > MAX_FLICK) { const sc = MAX_FLICK / speed; dvx *= sc; dvy *= sc; dvz *= sc; }
    vx[grabbed] = dvx;
    vy[grabbed] = dvy;
    vz[grabbed] = dvz;
    x[grabbed] = dragPos.x;
    y[grabbed] = dragPos.y;
    z[grabbed] = dragPos.z;
  }
  dragPrev.copy(dragPos);
  // The camera follows the cube's center of mass, so a hard throw can't send
  // the cube out of view (on a frictionless floor it would drift forever).
  // While a mass is grabbed the camera holds still, so the drag plane stays
  // steady under the cursor.
  if (grabbed < 0) {
    let cx = 0, cy = 0, cz = 0;
    for (let i = 0; i < N; i++) { cx += x[i]; cy += y[i]; cz += z[i]; }
    const inv = 1 / N;
    target.x += (cx * inv - target.x) * 0.1;
    target.y += (cy * inv - target.y) * 0.1;
    target.z += (cz * inv - target.z) * 0.1;
    positionCamera();
  }
  updateGeometry();
  updateReadout();
  renderer.render(scene, camera);
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
</script>
</body>
</html>
```

**Run it.** The whole show. The cube falls 0.63 s, splats, bounces, rings, and settles. The readout counts up and holds.

## Checkpoint

> [!PREDICT]
> With the file closed, from memory: how long is the cube in the air, what does the splat peak at, and where does the top row rest?

**Run this to verify your work so far:** build the file from the blocks in the sections above, then open `jelly.html` in a browser (double-click it, or `python3 -m http.server` from its folder and visit the page).

Expected: the cube is in the air 0.63 s — free fall from 2.0 m to the 0.05 m contact surface, t = √(2 × 1.95 / 9.81) — and lands at 6.2 m/s. The splat peaks at 31.7% strain at 0.75 s, and the peak is compression: the cube is being squashed, not stretched. At the bottom of the splat the cube is 1.85 m tall — 77% of its 2.4 m. It then rings at about 2 Hz and settles: under 1 cm/s by about 8 s and stays there, under 1 mm/s by about 10 s. At rest the readout holds at `125 masses · 780 springs · height 2.341 m · max strain 4.1%`.

The top row rests at 2.385 m, against the PREDICT's 2.352 m — 33 mm higher. The bottom layer is a plate of 25 masses, not five independent columns: the shear springs spread the floor's push sideways before it climbs, so the load is shared between columns. The measured total squash is 59 mm, 1.65× less than the column calculation's 98 mm.

The 4.1% is compression, not stretch. At rest, the most deformed spring is a bottom-layer spring squashed 4.1%, and the most stretched spring is only 0.9% stretched. The readout takes the absolute deformation either way — that's why it says "max strain" where Part 2 said "max stretch." A stretch-only readout would show the resting cube as nearly relaxed.

The PREDICT answers: 0.63 s in the air, 31.7% peak, top row 2.385 m.

Play: grab a top-layer mass and flick it sideways — the cube shears and springs back; that's the shear springs doing the job "A square with no center" described. Throw it hard and it drifts, and the camera follows.

**Likely errors:**
- If the cube tangles on landing — layers fold through each other and the readout spikes past 90% — `K_STRUCT` is still 32 (or a decimal slip to 10.0). Part 2's K was right for a pinned sheet; the free cube needs 100.
- If the readout says `300 springs`, the shear loop is missing. The cube still falls and splats — peak 38.4% instead of 31.7% — and looks fine for about 7 s after landing, and then buckles into a pancake. That's the no-shear exercise, arriving by accident.
- If a thrown cube leaves the frame and never comes back, the camera-follow block is missing. The drift itself is correct physics: a frictionless floor has nothing to stop it.
- If it blows up to NaN a few seconds in, `STEP_DT` is 1/60 (or a decimal slip put it there). The stability section is why the file takes 1/240.
- If the page freezes for seconds after a tab switch, the 0.1 s clamp is gone: the accumulator is trying to catch up on the entire hidden interval, one 1/240 s step at a time.

The two one-shot render calls are now redundant — the loop draws every frame. They're harmless; leave them or delete them.

## What's next

Part 2 listed Part 4's ingredients; here's where each one lands. Cloth resists stretching, not compression — the cube's springs resist both, and Part 4 needs tension-only springs, the end of Part 1's compression cheat. A draped cloth has to fold, not crumple — the cube's lattice is rigid where cloth must bend, and the Eurographics tutorial's third family, the interleaving (bending) springs, comes back. And the floor's flat `if (pen > 0)` meets a round table: contact with a curved surface is what makes steps expensive enough to matter for cost — the accumulator's original reason, the one Part 1's design note actually predicted. Same two lists, same three passes, same semi-implicit step.

## Exercises

- [ ] **Stiffer jelly.** Set `K_STRUCT = 200` and `K_SHEAR = 100`. The splat peak drops from 31.7% to 23.2%, the cube rests at 2.371 m (against 2.341), and the readout holds at 2.1% strain. Stiffer springs, less squash, a shorter ring — flick a corner and count.
- [ ] **Drop it from higher.** Set `BOTTOM = 5.0`. The peak is 52.5%, and the cube still recovers (3.0 m gives 39.3%). The peak grows roughly like the square root of the drop height: 31.7 × √2.5 ≈ 50, measured 52.5 — the springs are linear, so the energy goes up linearly and the amplitude with its square root.
- [ ] **Take out the shear springs.** Set `K_SHEAR = 0`. The splat peak is 38.4%, the cube looks fine for about 7 s after landing — and then it buckles: height under 2 m at about 7.5 s, a pancake by about 10 s. The vertical springs carry the weight, but nothing holds the columns straight. "A square with no center", in three dimensions.
- [ ] **Less damping.** Set `C = 0.2`. The peak is 37.3%, and the cube is still jiggling at 30 s — max speed 1.8 mm/s — under 1 mm/s only by about 40 s, against 10 s at C = 1.0.
- [ ] **A bigger step.** Set `STEP_DT = 1/60`. It blows up to NaN by about 7 s. Set it to 1/80: it survives (peak 32.4%). That's the boundary the stability section measured.
- [ ] **Zero gravity.** Set `G = 0`. The lattice keeps its shape — the shear springs hold it a cube — and floats. Throw it: it drifts at exactly constant velocity forever. The measured center-of-mass speed after 8 s is 0.400000 m/s, unchanged. The RECALL answer, made visible: nothing internal can change the cube's total momentum.

## Sources

Three.js (pinned to 0.185.1):

1. [Installation — three.js manual](https://threejs.org/manual/en/installation.html) — the import map for a no-build browser page; same CDN and same version for every import.
2. [Fundamentals — three.js manual](https://threejs.org/manual/en/fundamentals.html) — the renderer/scene/camera structure; the camera's default axes; fov in degrees; near/far clipping.
3. [WebGLRenderer — three.js docs](https://threejs.org/docs/pages/WebGLRenderer.html) — the WebGL 2 viewport.
4. [PerspectiveCamera — three.js docs](https://threejs.org/docs/pages/PerspectiveCamera.html) — the vertical field of view, in degrees.
5. [Vector3 — three.js docs](https://threejs.org/docs/pages/Vector3.html) — `project()`, world space to NDC.
6. [LineSegments — three.js docs](https://threejs.org/docs/pages/LineSegments.html) — lines drawn between pairs of vertices.
7. [Points — three.js docs](https://threejs.org/docs/pages/Points.html) — displaying points or point clouds.
8. [PointsMaterial — three.js docs](https://threejs.org/docs/pages/PointsMaterial.html) — `sizeAttenuation`.
9. [BufferAttribute — three.js docs](https://threejs.org/docs/pages/BufferAttribute.html) — `needsUpdate`.
10. [Raycaster — three.js docs](https://threejs.org/docs/pages/Raycaster.html) — `setFromCamera`; NDC input, −1 to 1.
11. [Plane — three.js docs](https://threejs.org/docs/pages/Plane.html) — `setFromNormalAndCoplanarPoint`.
12. [GridHelper — three.js docs](https://threejs.org/docs/pages/GridHelper.html) — a grid of lines.

Physics:

13. [Semi-implicit Euler method — Wikipedia](https://en.wikipedia.org/wiki/Semi-implicit_Euler_method) — the iteration's matrix form; unit determinant, area-preserving; stable periodic orbits for small step size.
14. [Center of mass — Wikipedia](https://en.wikipedia.org/wiki/Center_of_mass) — the weighted-mean point; Newton's laws apply there.
15. [Hauth, Etzmuss, Eberhardt, Klein, Sarlette, Sattler, Daubert, Kautz, "Cloth Animation and Rendering," Eurographics 2002 Tutorial T3](http://www.mirkosattler.de/publications/2002_tutorial_eg.pdf) — the structural/shear spring families and the stiffness hierarchy (cited in Part 2; the 3D lattice keeps the same two families).
