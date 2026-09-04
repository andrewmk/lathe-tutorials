# Simple Physics in the Browser, Part 4: A Square Cloth over a Round Table

> [!RECALL]
> Before reading on: Part 3's stability limit came down to one number, the largest ω in the system, ω² = λ_max / m, and the largest stable step was dt < 2/ω_max. What happens to that limit if you make the springs ten times stiffer?
>
> If you can't reconstruct it: ω scales as √K, so a ten-times stiffer system cuts the largest stable step by about √10. Stiffness buys rigidity at the cost of time. Part 3's cube ran at 1/240 with a 5.8× margin to its limit, and this part spends that margin — and then runs out of it.

Part 1 promised "a stretchy bouncy sheet, a wobbling jelly cube, a square of cloth that drapes over a round table." The sheet is done, the cube is done. This is the last one: a square of cloth — 225 masses, 1202 springs — dropped half a meter above a round table, and it drapes. And this part ends with a confession: the step you've used for three parts can't make a good cloth, and the step that can is a different way of computing.

## What you'll build

Two files, same scene. First, a force-based cloth: the same semi-implicit step as Part 3, with three upgrades — the in-plane springs go tension-only (the end of Part 1's compression cheat), the third spring family comes back (the bending springs Part 2's exercise promised), and the flat floor becomes a round table. It works, it settles, and the readout tells you the truth: at rest, the cloth is 12.7% stretched. A real tablecloth isn't.

Then a rewrite. The step stops computing forces and starts moving positions: Position Based Dynamics (PBD). Same two lists, same spring families, same table, same drop. The cloth comes to rest at 1.3% stretch and stays calm. And the stability limit you've been managing since Part 3 — the reason the file runs four substeps per frame — is gone, for a reason that falls out of the math in one paragraph.

## Prerequisites

Parts 1–3 of this series, and Part 3's file in particular: the two-list mass/spring architecture, the three-pass step, pairwise damping, the accumulator, and the ω·dt < 2 stability story. The Three.js setup — renderer, scene, camera, `LineSegments`, `Points`, picking, drag, orbit — is Part 3's, unchanged, and this part says so when it says so. One new idea is introduced properly: PBD, from the 2007 paper that named it.

## The stage: the same stage, plus a table

Part 3's stage is still the stage: a renderer, a scene, a camera on a homemade orbit, a grid on the floor. It comes with one new object, the table, which is where this part's physics starts. Save the file as `cloth.html` and paste this first block:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Cloth over a round table, Part 4</title>
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

// ---- The stage: renderer, scene, camera, table ---------------------------
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x101418);

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 200);
const target = new THREE.Vector3(0, 1.5, 0);
let theta = 0.25, phi = 1.25, radius = 17;
function positionCamera() {
  camera.position.set(
    target.x + radius * Math.sin(phi) * Math.sin(theta),
    target.y + radius * Math.cos(phi),
    target.z + radius * Math.sin(phi) * Math.cos(theta)
  );
  camera.lookAt(target);
}
positionCamera();

const grid = new THREE.GridHelper(16, 16);
scene.add(grid);

// The table is a solid cylinder in the physics: a disk top at height TABLE_H
// with radius TABLE_R, and a round side wall down to the floor. We draw it as
// lines: two rims, twelve struts, and spokes across the top.
const TABLE_R = 2.4;
const TABLE_H = 2.4;
{
  const seg = 48, pos = [];
  for (let s = 0; s < seg; s++) {
    const a0 = (s / seg) * 2 * Math.PI, a1 = ((s + 1) / seg) * 2 * Math.PI;
    pos.push(Math.cos(a0) * TABLE_R, TABLE_H, Math.sin(a0) * TABLE_R,
             Math.cos(a1) * TABLE_R, TABLE_H, Math.sin(a1) * TABLE_R);
    pos.push(Math.cos(a0) * TABLE_R, 0, Math.sin(a0) * TABLE_R,
             Math.cos(a1) * TABLE_R, 0, Math.sin(a1) * TABLE_R);
  }
  for (let s = 0; s < 12; s++) {
    const a = (s / 12) * 2 * Math.PI;
    pos.push(Math.cos(a) * TABLE_R, TABLE_H, Math.sin(a) * TABLE_R,
             Math.cos(a) * TABLE_R, 0, Math.sin(a) * TABLE_R);
  }
  const geo = new THREE.BufferGeometry();
  geo.setAttribute("position", new THREE.BufferAttribute(new Float32Array(pos), 3));
  scene.add(new THREE.LineSegments(geo, new THREE.LineBasicMaterial({ color: 0x5a6675 })));
  const sp = [];
  for (let s = 0; s < 8; s++) {
    const a = (s / 8) * 2 * Math.PI;
    sp.push(0, TABLE_H, 0, Math.cos(a) * TABLE_R, TABLE_H, Math.sin(a) * TABLE_R);
  }
  const sg = new THREE.BufferGeometry();
  sg.setAttribute("position", new THREE.BufferAttribute(new Float32Array(sp), 3));
  scene.add(new THREE.LineSegments(sg, new THREE.LineBasicMaterial({ color: 0x39434f })));
}
renderer.render(scene, camera);
```

Run it (double-click the file, or `python3 -m http.server` from its folder and visit the page). The grid and a wireframe table: two rims, twelve struts, eight spokes across the top. The table is drawn as lines, but in the physics it's a solid — the comment in the code says so, and the next section leans on it. The solid is a cylinder: a disk top at height `TABLE_H = 2.4` m with radius `TABLE_R = 2.4` m, and a round side wall down to the floor. Drawing it as lines is the cheap part; pushing masses out of it is the part with cases.

## The cloth as data: 225 masses, 1202 springs

The cloth is a 15×15 lattice, `SPACING = 0.6` m between neighbors, so it's 8.4 m on a side — big enough to overhang the table badly, which is the point. `MASS = 0.1` kg and `G = 9.81` m/s² carry over unchanged from Parts 2 and 3. It starts flat, centered on the table, `DROP = 0.5` m above the tabletop: the center of the cloth is 0.45 m above the contact skin, which is the number the first PREDICT below uses.

```js
// ---- The cloth as data: 225 masses, 1202 springs -------------------------
const SIDE = 15;           // 15 masses per side
const N = SIDE * SIDE;     // 225 masses
const SPACING = 0.6;       // m between neighbors (cloth side = 14 * 0.6 = 8.4 m)
const MASS = 0.1;          // kg per mass
const G = 9.81;            // m/s^2, down
const DROP = 0.5;          // m, the cloth starts this far above the tabletop
const RADIUS = 0.05;       // m, contact "skin", carried over from Part 3
const K_STRUCT = 100;      // N/m, the in-plane springs (tension-only)
const K_SHEAR = 50;        // N/m, the diagonals (tension-only)
const K_BEND = 20;         // N/m, the bending family (two-sided)
const C = 1.0;             // N*s/m, pairwise damping per spring, from Part 3
const K_FLOOR = 980;       // N/m, contact stiffness, now against three surfaces
const C_FLOOR = 2;         // N*s/m, contact dashpot
const C_SLIDE = 0.8;       // N*s/m, tangential friction per surface in contact
const STEP_DT = 1 / 240;   // s, the fixed physics step

const x = new Float64Array(N), y = new Float64Array(N), z = new Float64Array(N);
const vx = new Float64Array(N), vy = new Float64Array(N), vz = new Float64Array(N);
const fx = new Float64Array(N), fy = new Float64Array(N), fz = new Float64Array(N);
const springs = [];

function ij(a, b) { return a * SIDE + b; }

function addSpring(a, b, k, shear, bend) {
  springs.push({ a, b, k, shear, bend,
    rest: Math.hypot(x[b] - x[a], y[b] - y[a], z[b] - z[a]) });
}

// The cloth is a flat square lattice in the x/z plane, centered on the y
// axis, DROP meters above the tabletop.
for (let a = 0; a < SIDE; a++)
  for (let b = 0; b < SIDE; b++) {
    const i = ij(a, b);
    x[i] = (a - (SIDE - 1) / 2) * SPACING;
    y[i] = TABLE_H + DROP;
    z[i] = (b - (SIDE - 1) / 2) * SPACING;
  }

// Structural springs: the lattice edges. 2 * 14 * 15 = 420 of them.
for (let a = 0; a < SIDE; a++)
  for (let b = 0; b < SIDE; b++) {
    const i = ij(a, b);
    if (a + 1 < SIDE) addSpring(i, ij(a + 1, b), K_STRUCT, false, false);
    if (b + 1 < SIDE) addSpring(i, ij(a, b + 1), K_STRUCT, false, false);
  }

// Shear springs: both diagonals of every cell. 2 * 14 * 14 = 392 of them.
for (let a = 0; a < SIDE; a++)
  for (let b = 0; b < SIDE; b++) {
    if (a + 1 < SIDE && b + 1 < SIDE) {
      addSpring(ij(a, b), ij(a + 1, b + 1), K_SHEAR, true, false);
      addSpring(ij(a + 1, b), ij(a, b + 1), K_SHEAR, true, false);
    }
  }

// Bending springs: every second neighbor, (a+2, b) and (a, b+2). These are
// the springs Part 2's exercise promised: they resist the cloth folding,
// which edge and diagonal springs do not. 2 * 13 * 15 = 390 of them.
for (let a = 0; a < SIDE; a++)
  for (let b = 0; b < SIDE; b++) {
    const i = ij(a, b);
    if (a + 2 < SIDE) addSpring(i, ij(a + 2, b), K_BEND, false, true);
    if (b + 2 < SIDE) addSpring(i, ij(a, b + 2), K_BEND, false, true);
  }

// ---- The cloth as geometry: three line families and a point cloud --------
function makeLines(list, color) {
  const positions = new Float32Array(list.length * 6);
  const geo = new THREE.BufferGeometry();
  geo.setAttribute("position", new THREE.BufferAttribute(positions, 3));
  const lines = new THREE.LineSegments(geo, new THREE.LineBasicMaterial({ color }));
  lines.userData.positions = positions;
  lines.userData.list = list;
  scene.add(lines);
  return lines;
}
const structSprings = springs.filter(s => !s.shear && !s.bend);
const shearSprings = springs.filter(s => s.shear);
const bendSprings = springs.filter(s => s.bend);
const structLines = makeLines(structSprings, 0x8fb3ff);
const shearLines = makeLines(shearSprings, 0x3a4552);
const bendLines = makeLines(bendSprings, 0x2a333d);

const massPositions = new Float32Array(N * 3);
const massGeo = new THREE.BufferGeometry();
massGeo.setAttribute("position", new THREE.BufferAttribute(massPositions, 3));
const massPoints = new THREE.Points(massGeo, new THREE.PointsMaterial({ color: 0xe8ecf0, size: 7, sizeAttenuation: false }));
scene.add(massPoints);

function updateGeometry() {
  for (const lines of [structLines, shearLines, bendLines]) {
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

Three families, all from the Eurographics tutorial Part 2 cited, all three of them for the first time:

- **Structural springs** — the lattice edges, 420 of them. These are the fibers.
- **Shear springs** — both diagonals of every cell, 392. Same job as Part 2's: stop the lattice shearing into rhombi.
- **Bending springs** — every second neighbor, 390. Part 2's sheet skipped these on purpose; a draped cloth needs them, because edge and diagonal springs do nothing to stop the cloth folding along a line.

The numbers on the K constants are still N/m — this is the force-based file — and the hierarchy follows the survey's one line on cloth: it "resists bending weakly, but has a relatively strong resistance to stretching," so `K_STRUCT = 100` > `K_SHEAR = 50` > `K_BEND = 20`. The damping is Part 3's pairwise `C = 1.0`, and the contact constants (`K_FLOOR`, `C_FLOOR`, `C_SLIDE`) are Part 3's floor constants reused, because the table is three surfaces: a disk top, a round wall, and the actual floor.

Run it. The one-shot render at the end of the block draws the cloth: a flat blue grid floating 0.5 m above the tabletop, dark shear lines inside it, and the faintest bending lines underneath. Nothing moves yet — there's no loop.

## The step: the same three passes, minus the compression

The step is Part 3's step with three changes: the in-plane springs are tension-only, the bending springs are two-sided, and the floor's one plane becomes the table's three surfaces.

> [!PREDICT]
> The cloth's center of mass starts 0.45 m above the contact skin. When does the first mass touch the table, and at what speed?

The free-fall answer is t = √(2 × 0.45 / 9.81) = 0.30 s at v = g·t = 2.97 m/s. The measured numbers (substep resolution, the first frame in which any mass is inside the skin) are 0.304 s and 2.98 m/s. Check them when the clock runs, below.

```js
// ---- The step: the same three passes, minus the compression --------------
function step(dt) {
  // Pass 1: seed every mass's force with gravity, as always.
  for (let i = 0; i < N; i++) { fx[i] = 0; fy[i] = -MASS * G; fz[i] = 0; }

  // Pass 2: the springs. Same Hooke plus pairwise damping as Part 3, with
  // one change: the in-plane springs are tension-only. Cloth is a lattice of
  // fibers, and a fiber can pull but not push, so a spring shorter than its
  // rest length exerts no spring force. The dashpot still works both ways:
  // it bleeds off the relative motion of the two ends either way.
  for (const s of springs) {
    const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a], dz = z[s.b] - z[s.a];
    const len = Math.hypot(dx, dy, dz);
    const ux = dx / len, uy = dy / len, uz = dz / len;
    const stretch = len - s.rest;
    const k = s.bend ? s.k : (stretch > 0 ? s.k : 0);
    const rel = (vx[s.b] - vx[s.a]) * ux + (vy[s.b] - vy[s.a]) * uy + (vz[s.b] - vz[s.a]) * uz;
    const F = k * stretch + C * rel;
    fx[s.a] += F * ux; fy[s.a] += F * uy; fz[s.a] += F * uz;
    fx[s.b] -= F * ux; fy[s.b] -= F * uy; fz[s.b] -= F * uz;
  }

  // Pass 3: the table, then the semi-implicit update. Part 3's floor was one
  // plane; the table is three surfaces: the top (a disk), the side (a round
  // wall), and the actual floor.
  for (let i = 0; i < N; i++) {
    // The table. Instead of three separate surface checks, find the
    // closest point on the solid cylinder and, if the mass is within
    // RADIUS of it, push it straight away from that point. Over the disk
    // the closest point is straight down; beside the wall, straight in;
    // outside both, the rim itself. The normal comes out of the geometry,
    // so the rim corner is rounded for free.
    const r = Math.hypot(x[i], z[i]);
    let px, py, pz;
    if (r < TABLE_R && y[i] > TABLE_H) {
      px = x[i]; py = TABLE_H; pz = z[i];
    } else if (r > TABLE_R && y[i] < TABLE_H) {
      px = TABLE_R * x[i] / r; py = y[i]; pz = TABLE_R * z[i] / r;
    } else if (r > TABLE_R && y[i] > TABLE_H) {
      px = TABLE_R * x[i] / r; py = TABLE_H; pz = TABLE_R * z[i] / r;
    } else {
      // Inside the solid: unreachable if the contact above works, but nudge
      // out through the shallower face instead of dividing by zero.
      if (TABLE_H - y[i] < TABLE_R - r) {
        px = x[i]; py = TABLE_H; pz = z[i];
      } else {
        const rr = Math.max(r, 1e-9);
        px = TABLE_R * x[i] / rr; py = y[i]; pz = TABLE_R * z[i] / rr;
      }
    }
    const dx = x[i] - px, dy = y[i] - py, dz = z[i] - pz;
    const dist = Math.hypot(dx, dy, dz);
    if (dist < RADIUS && dist > 1e-9) {
      const nx = dx / dist, ny = dy / dist, nz = dz / dist;
      const pen = RADIUS - dist;
      fx[i] += K_FLOOR * pen * nx;
      fy[i] += K_FLOOR * pen * ny;
      fz[i] += K_FLOOR * pen * nz;
      // Dashpot along the normal, only on the way in.
      const vn = vx[i] * nx + vy[i] * ny + vz[i] * nz;
      if (vn < 0) {
        fx[i] -= C_FLOOR * vn * nx;
        fy[i] -= C_FLOOR * vn * ny;
        fz[i] -= C_FLOOR * vn * nz;
      }
      // Friction: damp the velocity along the two directions tangent to
      // the surface. t1 is n x up (or n x x when the normal is vertical);
      // t2 completes the orthonormal pair.
      let tx, ty, tz;
      if (Math.abs(ny) < 0.9) {
        const L = Math.hypot(nx, nz);
        tx = -nz / L; ty = 0; tz = nx / L;
      } else {
        const L = Math.hypot(nz, ny);
        tx = 0; ty = nz / L; tz = -ny / L;
      }
      const t2x = ny * tz - nz * ty, t2y = nz * tx - nx * tz, t2z = nx * ty - ny * tx;
      const v1 = vx[i] * tx + vy[i] * ty + vz[i] * tz;
      const v2 = vx[i] * t2x + vy[i] * t2y + vz[i] * t2z;
      fx[i] -= C_SLIDE * (v1 * tx + v2 * t2x);
      fy[i] -= C_SLIDE * (v1 * ty + v2 * t2y);
      fz[i] -= C_SLIDE * (v1 * tz + v2 * t2z);
    }

    // The floor: a half-space, straight from Part 3, plus friction.
    const penF = RADIUS - y[i];
    if (penF > 0) {
      fy[i] += K_FLOOR * penF;
      if (vy[i] < 0) fy[i] -= C_FLOOR * vy[i];
      fx[i] -= C_SLIDE * vx[i];
      fz[i] -= C_SLIDE * vz[i];
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

The first change is the tension-only line in the spring pass: `const k = s.bend ? s.k : (stretch > 0 ? s.k : 0);`. This is the end of the compression cheat Part 1 named and Part 2 kept. The argument is one sentence: cloth is a lattice of fibers, and a fiber can pull but not push. When two masses get closer than the fiber's rest length, a real fiber buckles out of the way and the distance goes on shrinking; a linear spring would push them apart like a rod. The dashpot is deliberately left two-sided — the fold should still lose energy as it forms — so a compressed fiber is a dashpot with no spring. The bending springs stay two-sided: they *are* the fold resistance, and a fold that resists in only one direction isn't a fold.

The second change is the table itself. Part 3's floor was one plane and one test, `if (pen > 0)`. A cylinder has three faces, and three separate window checks (is it over the disk? beside the wall? near the rim?) get the rim wrong, because the rim is a corner where two surfaces meet. So the step doesn't check three surfaces. It finds the **closest point on the solid cylinder** — over the disk, straight down; beside the wall, straight in; outside both, the rim point — and if the mass is within `RADIUS` of that point, it gets a spring force, a one-way dashpot, and friction, all along the normal to that point. The normal falls out of the geometry, so the rim corner comes out rounded for free, and a mass sliding off the edge hands from the top's normal to the rim's normal to the wall's normal without a seam. The floor stays a separate half-space, Part 3's, with the same friction added.

The friction deserves a line of its own, because the cloth needs it in a way the cube didn't: the cube hit a floor and stayed put, but a cloth that touches the tabletop will slide on it. `C_SLIDE` damps the velocity along the two directions tangent to the surface it's touching — the two tangents are built from the normal, `n × up` (or `n × x` when the normal is vertical) and the cross product that completes the pair. Without it, the impact gives the cloth a net sideways shove and it slides off the table in one piece, a few hundred meters' worth of wrong physics that looks, from the camera, like the cloth decided to leave.

## The mouse: Part 3's, unchanged

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
    theta -= dx * 0.005;
    phi = Math.max(0.15, Math.min(1.5, phi - dy * 0.005));
    positionCamera();
  } else if (grabbed >= 0) {
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

Pick, drag on the camera-facing plane, orbit from empty space, the 40 m/s flick clamp, the 0.1 s dt clamp in the clock — all of it is Part 3's code, unchanged, and it works on the cloth as soon as the loop exists. One difference you'll feel when you play: the cloth is 225 masses and the drag moves one of them, so a grab feels like tugging a corner of a tablecloth rather than holding a cube.

## The clock starts the machine — and this time it costs

```js
// ---- The clock: fixed timestep, accumulator, readout ---------------------
const readout = document.getElementById("readout");
let accumulator = 0, last = null, t = 0;

function updateReadout() {
  // The headline number for a cloth is how much it stretches. The bending
  // springs fold when the cloth folds, so they are not part of the figure.
  let maxStretch = 0;
  for (const s of springs) {
    if (s.bend) continue;
    const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a], dz = z[s.b] - z[s.a];
    const e = (Math.hypot(dx, dy, dz) - s.rest) / s.rest;
    if (e > maxStretch) maxStretch = e;
  }
  readout.textContent =
    `${N} masses · ${springs.length} springs · max stretch ${(maxStretch * 100).toFixed(1)}%   t = ${t.toFixed(1)} s`;
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
  // The camera follows the cloth's center of mass, so a hard throw can't send
  // it out of view. While a mass is grabbed the camera holds still, so the
  // drag plane stays steady under the cursor.
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

The clock is Part 3's clock: fixed `STEP_DT = 1/240`, accumulator, 0.1 s clamp, camera follow, readout. One line changed in the readout, and it's a deliberate choice: Part 3's resting cube was mostly *compressed* (4.1% strain, 0.9% stretch), so its readout took the absolute deformation. A cloth at rest is stretched, so this one takes the plain max stretch over the structural and shear springs, and leaves the bending springs out — they fold when the cloth folds, and a fold is not a stretch.

Run it. The cloth falls for 0.30 s — the PREDICT's answer, measured 0.304 s at 2.98 m/s — and slaps onto the tabletop. The splat is gentler than the cube's: peak stretch 22.4% at 0.81 s, against the cube's 31.7% strain. Then it drapes. The center of the cloth stays over the table at 2.45 m, the four edges peel over the rim and hang down the wall, and the corners swing out and crumple onto the floor. By about 6.4 s it's under 1 cm/s, and by about 8 s it's still to a millimeter.

And the readout settles at a number the cube never had to show: **max stretch 12.7%**, and it stays there. The cloth is done moving and it is still 12.7% longer than its rest lengths, on the springs that carry its own weight.

## The readout says 12.7%: the price of stiffness

Here is what the resting cloth looks like, measured off the settled file:

- The center of the cloth sits at 2.45 m, on the tabletop, as it should.
- The midpoint of each edge hangs down the wall at y ≈ 0.2 m, in skin contact with the cylinder (its radius is 2.35 m = `TABLE_R − RADIUS`). Four edges, four straight hanging panels.
- The four corners are on the floor, at radius ≈ 5.08 m from the axis. That's far out — the corner's straight-line distance to the rim is 3.54 m — but it's not a stretch: the springs in the corner region are stretched at most 6.7%. The corner has crumpled and fanned out on the floor, and a fanned corner reaches further than a straight one. 40 of the 225 masses are on the floor, all in the four corner regions.
- The 12.7% is carried by the in-plane springs under the tabletop, where the overhanging edges hang the cloth's weight off the disk.

A real tablecloth, dropped on a real table, is not 12.7% stretched at rest. It stretches a percent, maybe two. The cloth in this file is a tablecloth the way the Part 3 cube was a jelly: the right topology, the wrong material law, and a readout that says so.

So do what the RECALL says to do: make the springs stiffer. The measured trade, at `STEP_DT = 1/240`:

| K_STRUCT | rest stretch | largest stable step |
|---|---|---|
| 100 | 12.7% | 1/60 (and it jiggles at 2.8 m/s there) |
| 200 | 9.9% | 1/60 |
| 300 | 8.8% | 1/60 |
| 400 | 8.1% | 1/100 (NaN at 1/80) |

Four times the stiffness buys 12.7% → 8.1%, and cuts the largest stable step from 1/60 to 1/100. The survey said it before you measured it: the mass-spring ODE "is stiff; that is, it possesses a wide range of eigenvalues," and an explicit scheme pays for it with "excessively small time steps." Part 3's ω·dt < 2 is the same sentence in your own notation, and the cube's 5.8× margin is exactly what this cloth spends. Past K ≈ 400 there is no stable step left that's fast enough to render at, and the stretch is still 8%. The force-based method has a wall, and the wall is the math, not the code: no choice of K moves it, because K *is* what sets ω.

There's a cost line in the same trade. With the table, a step is no longer 40 µs of spring arithmetic: the closest-point case split and the friction tangents run for every mass, every step, over 225 masses and 1202 springs instead of 125 and 780. Measured on this machine, `STEP_DT = 1/240`: **87.2 µs per step — 2.2× Part 3's step — 348.8 µs per frame at 60 fps, 2.1% of the budget.** That's the number Part 1's design note was pointing at — "when this series gets there, the accumulator comes back" — and it's the accumulator's first real job: the 0.1 s clamp now limits catch-up after a hiccup to 24 steps, 2.1 ms, instead of Part 3's 24 steps of almost nothing. On the PBD file below the same clamp limits catch-up to 24 × 322 µs = 7.7 ms, which is 46% of a frame, and the clamp is doing work you can see.

## A different step: Position Based Dynamics

The cloth needs to be both stiff and calm. Force-based can give you one at a time: stiff K means small dt, and small dt means a stiff cloth that still stretches 8% and jiggles at meters per second when you give it the step it deserves. So here's the rewrite, and it's the method Part 2 named but didn't use: **Position Based Dynamics**, from the 2007 paper that introduced it.

The paper's claim is short and it's the whole idea: a position-based approach "omits the velocity layer as well and immediately works on the positions," and its "main advantage" is "controllability — overshooting problems of explicit integration schemes in force based systems can be avoided." The step it gives, in the paper's own pseudocode, is:

```text
(5)  forall vertices i do vi ← vi + Δt·w_i·f_ext(x_i)
(6)  dampVelocities(v_1, ..., v_N)
(7)  forall vertices i do p_i ← x_i + Δt·v_i
(8)  forall vertices i do generateCollisionConstraints(x_i → p_i)
(9)  loop solverIterations times
(10) projectConstraints(C_1, ..., C_{M+M_coll}, p_1, ..., p_N)
(11) endloop
(12) forall vertices i
(13) vi ← (p_i − x_i)/Δt
(14) x_i ← p_i
(15) endfor
(16) velocityUpdate(v_1, ..., v_N)
```

Three passes, and if you squint they rhyme with the three passes you've used all series:

1. **Predict** (lines 5–7): the semi-implicit nudge from Part 3, verbatim — velocity first (the paper uses line 5 to add gravity, `vi ← vi + Δt·g`), then position. Every mass gets a predicted position `p_i`, and the old position is saved.
2. **Project** (lines 8–11): this is the new part. Constraints don't produce forces; they produce *position fixes*. A spring that's off its rest length moves its two ends part of the way back toward the rest length, and the solver does this for every constraint, `solverIterations` times, in a Gauss-Seidel pass — the same "iterate until things agree" move as the cloth survey's relaxation, and the same one Provot used in 1995 to tame stiff cloth, "much weaker stretching energies and then post-processing the cloth mesh at each time step, iteratively enforcing constraints."
3. **Differentiate** (lines 13–14): velocity is not integrated, it's *measured* — `v = (p − x)/dt`, where the mass ended up. The paper notes this "is in exact correspondence with a Verlet integration step," because Verlet stores the velocity implicitly as the difference between the current and the last position. Whatever the projections did, the new velocity agrees with the actual motion, and that one fact is why contact can never feed the cloth energy the way Act 1's contact springs can.

Two details from the paper land in the code as written. First, a constraint has "a stiffness parameter k_j ∈ [0..1] and a type of either equality or inequality," and "if the type is inequality, the projection is only performed if C(p1, ..., pn) < 0" — that's the tension-only line, formalized: the in-plane fibers are inequality constraints (a fiber shorter than its rest length is satisfied, so it's left alone to fold), the bending springs are equality. Second, if you project every spring by a fraction k of its error, ITER of them don't reach stiffness k — "the effect of k is non-linear," and the paper's fix is to project by

```text
k' = 1 − (1 − k)^(1/ITER)
```

instead, so that ITER projections land exactly on k. That's the `kp` in the next block, precomputed per spring.

The rewrite touches three places in the file. Replace the constants and arrays block — from `const MASS` through the `fx / fy / fz` line — with this:

```js
const G = 9.81;            // m/s^2, down
const DROP = 0.5;          // m, the cloth starts this far above the tabletop
const RADIUS = 0.05;       // m, contact "skin", carried over from Part 3
const ITER = 4;            // constraint iterations per step
const K_STRUCT = 1.0;      // 0..1, the in-plane fibers (tension-only)
const K_SHEAR = 0.5;       // 0..1, the diagonals (tension-only)
const K_BEND = 0.1;        // 0..1, the bending family (two-sided)
const FRICTION = 0.05;     // tangential velocity removed per step, in contact
const KDAMP = 0.02;        // damping toward the center of mass, per step
const STEP_DT = 1 / 240;   // s, the fixed physics step

const x = new Float64Array(N), y = new Float64Array(N), z = new Float64Array(N);
const vx = new Float64Array(N), vy = new Float64Array(N), vz = new Float64Array(N);
const ox = new Float64Array(N), oy = new Float64Array(N), oz = new Float64Array(N);
const cnx = new Float64Array(N), cny = new Float64Array(N), cnz = new Float64Array(N);
const onContact = new Uint8Array(N);
```

Read it as a change of units. `K_STRUCT` and friends are no longer N/m; they're fractions of the error that ITER projections should absorb, 0 for "no constraint" to 1 for "enforce exactly." `K_STRUCT = 1.0` says the in-plane fibers are enforced exactly — a fiber at 120% of rest length comes back to 100%, not to 108%. `ITER = 4` is the solver's patience, `FRICTION = 0.05` removes 5% of the tangential velocity of anything touching the table each step, and `KDAMP = 0.02` is damping toward the center of mass — Part 3's momentum lesson, kept: nothing in this scene absorbs a throw, so the damping eats only relative motion. And `MASS` is gone. Equal-mass PBD projection never divides by mass; a constraint correction splits evenly between its two ends, so the file no longer needs a mass at all. The new arrays are what the step needs that the force step didn't: `ox/oy/oz` save the pre-predict positions for the differentiation, and `cnx/cny/cnz` plus `onContact` remember where each mass touched, so friction can be applied in the velocity pass.

Then replace `addSpring` — four lines — with seven:

```js
function addSpring(a, b, k, shear, bend) {
  // kp is the stiffness of a single projection, chosen so that ITER of them
  // together reach the target stiffness k (the PBD paper's formula).
  springs.push({ a, b, k, shear, bend,
    rest: Math.hypot(x[b] - x[a], y[b] - y[a], z[b] - z[a]),
    kp: 1 - Math.pow(1 - k, 1 / ITER) });
}
```

The spring now carries `kp`, the per-iteration stiffness from the paper's formula. The rest of the data section — the lattice, the three spring families, the line rendering — is untouched and still correct: PBD doesn't care what a spring *means*, only what it *does*, and this one does `move the ends toward their rest length by a fraction of the error`.

And the step itself. Replace the whole force-based `step()` — from the banner `// ---- The step: the same three passes...` through its closing brace — with this:

```js
// ---- The step: predict, project, differentiate ---------------------------
function step(dt) {
  // 1. Predict. The semi-implicit nudge from Part 3, verbatim: velocity
  //    first (gravity), then position. Save where every mass was.
  for (let i = 0; i < N; i++) {
    ox[i] = x[i]; oy[i] = y[i]; oz[i] = z[i];
    vy[i] -= G * dt;
    x[i] += vx[i] * dt;
    y[i] += vy[i] * dt;
    z[i] += vz[i] * dt;
  }

  // 2. Project. Move the masses to where the constraints want them, ITER
  //    times. A constraint is a tiny position fix, not a force: a spring
  //    that is off its rest length pulls the two ends part of the way back.
  for (let it = 0; it < ITER; it++) {
    for (const s of springs) {
      const dx = x[s.b] - x[s.a], dy = y[s.b] - y[s.a], dz = z[s.b] - z[s.a];
      const d = Math.hypot(dx, dy, dz);
      const err = d - s.rest;
      // Tension-only, as in the force version: a fiber pulls but does not
      // push, so a shorter-than-rest length is left alone to fold. The
      // bending springs work both ways: they are what resists the fold.
      if (err < 0 && !s.bend) continue;
      const f = s.kp * 0.5 * err / d;
      x[s.a] += dx * f; y[s.a] += dy * f; z[s.a] += dz * f;
      x[s.b] -= dx * f; y[s.b] -= dy * f; z[s.b] -= dz * f;
    }
    for (let i = 0; i < N; i++) {
      onContact[i] = 0;
      // Contact is a projection, not a force: find the closest point on the
      // solid cylinder, and if the mass is inside the skin, move it to it.
      // The closest-point math is the force version's, unchanged.
      const r = Math.hypot(x[i], z[i]);
      let px, py, pz;
      if (r < TABLE_R && y[i] > TABLE_H) {
        px = x[i]; py = TABLE_H; pz = z[i];
      } else if (r > TABLE_R && y[i] < TABLE_H) {
        px = TABLE_R * x[i] / r; py = y[i]; pz = TABLE_R * z[i] / r;
      } else if (r > TABLE_R && y[i] > TABLE_H) {
        px = TABLE_R * x[i] / r; py = TABLE_H; pz = TABLE_R * z[i] / r;
      } else {
        if (TABLE_H - y[i] < TABLE_R - r) {
          px = x[i]; py = TABLE_H; pz = z[i];
        } else {
          const rr = Math.max(r, 1e-9);
          px = TABLE_R * x[i] / rr; py = y[i]; pz = TABLE_R * z[i] / rr;
        }
      }
      const dx = x[i] - px, dy = y[i] - py, dz = z[i] - pz;
      const dist = Math.hypot(dx, dy, dz);
      if (dist < RADIUS && dist > 1e-9) {
        x[i] += dx / dist * (RADIUS - dist);
        y[i] += dy / dist * (RADIUS - dist);
        z[i] += dz / dist * (RADIUS - dist);
        cnx[i] = dx / dist; cny[i] = dy / dist; cnz[i] = dz / dist;
        onContact[i] = 1;
      }
      // The floor, the same way.
      if (y[i] < RADIUS) {
        y[i] = RADIUS;
        cnx[i] = 0; cny[i] = 1; cnz[i] = 0;
        onContact[i] = 1;
      }
    }
  }

  // 3. Differentiate. Velocity is not integrated in PBD; it is measured:
  //    where the mass ended up, relative to where it was, over dt. Whatever
  //    the projections did to the positions, the velocities agree with the
  //    actual motion, so contact can never feed the cloth energy.
  for (let i = 0; i < N; i++) {
    vx[i] = (x[i] - ox[i]) / dt;
    vy[i] = (y[i] - oy[i]) / dt;
    vz[i] = (z[i] - oz[i]) / dt;
    // 4. Friction, for anything in contact: remove the inward normal
    //    component (an inelastic touch) and scale the tangent part.
    if (onContact[i]) {
      const vn = vx[i] * cnx[i] + vy[i] * cny[i] + vz[i] * cnz[i];
      if (vn < 0) {
        vx[i] -= vn * cnx[i];
        vy[i] -= vn * cny[i];
        vz[i] -= vn * cnz[i];
      }
      const v1 = vx[i] - (vx[i] * cnx[i]) * cnx[i] - (vy[i] * cny[i]) * cnx[i] - (vz[i] * cnz[i]) * cnx[i];
      const v2 = vy[i] - (vx[i] * cnx[i]) * cny[i] - (vy[i] * cny[i]) * cny[i] - (vz[i] * cnz[i]) * cny[i];
      const v3 = vz[i] - (vx[i] * cnx[i]) * cnz[i] - (vy[i] * cny[i]) * cnz[i] - (vz[i] * cnz[i]) * cnz[i];
      vx[i] = (vx[i] - v1) + v1 * (1 - FRICTION);
      vy[i] = (vy[i] - v2) + v2 * (1 - FRICTION);
      vz[i] = (vz[i] - v3) + v3 * (1 - FRICTION);
    }
  }

  // 5. Damping, relative to the center of mass. Part 3's lesson, by
  //    construction: nothing in this scene absorbs momentum, so the damping
  //    eats only relative motion, or a throw would never stop.
  let cvx = 0, cvy = 0, cvz = 0;
  for (let i = 0; i < N; i++) { cvx += vx[i]; cvy += vy[i]; cvz += vz[i]; }
  const inv = 1 / N;
  cvx *= inv; cvy *= inv; cvz *= inv;
  for (let i = 0; i < N; i++) {
    vx[i] = (1 - KDAMP) * vx[i] + KDAMP * cvx;
    vy[i] = (1 - KDAMP) * vy[i] + KDAMP * cvy;
    vz[i] = (1 - KDAMP) * vz[i] + KDAMP * cvz;
  }
}
```

Walk it against the pseudocode. Pass 1, **predict**: the old positions are saved into `ox/oy/oz`, gravity nudges `vy`, and every mass moves to its predicted position — the paper's line 7. Pass 2, **project**: `ITER` passes over the constraints. A spring with current length `d` and rest length `s.rest` has error `err = d − s.rest`; the correction moves each end `s.kp × 0.5 × err/d` of the way, which for equal masses is exactly the paper's equal-weight projection, and the `if (err < 0 && !s.bend) continue;` line is the inequality type — the fibers skip when they're slack, the bending springs work both ways. Contact is a constraint too: the same closest-point-on-cylinder math as Act 1, but instead of a spring force it *moves the mass to the skin* — `x[i] += dx/dist × (RADIUS − dist)` — and stores the normal, so the projection is exact and a mass can never be pushed through, which is the paper's line 8 turned into a loop. Pass 3, **differentiate**: `v = (x − ox)/dt`, the measured velocity. Then two things the pseudocode leaves to `velocityUpdate` (line 16) and `dampVelocities` (line 6), inlined: **friction** — for every mass in contact, the inward normal component of the measured velocity is removed (an inelastic touch) and the tangent part is scaled by `1 − FRICTION` — and the **COM-relative damping** from Part 3.

Run it. Same file, same drop, same table — the only things that changed are the constants, the arrays, `addSpring`, and `step()`. The cloth falls 0.30 s (measured 0.304 s at 2.97 m/s, the free-fall numbers again — the predict pass is the same free fall, so of course they agree), slaps down, and drapes. But the drape is a different cloth.

## The same scene, run again: stiff and calm

Measured, at rest:

- **Max stretch 1.3%** — against 12.7% for the force file. The readout settles at `225 masses · 1202 springs · max stretch 1.3%` and stays there.
- Peak stretch during the impact: **2.0%** at 0.90 s, against 22.4%. The slap is the same, the cloth is the different one.
- The drape: the center sits at 2.45 m on the tabletop; the edge midpoints hang the wall as a straight band at y ≈ 1.6–1.75 m (a stiffer cloth holds its edge higher than the stretchy one's y ≈ 0.2 m); the corners are a tighter fan on the floor at radius ≈ 4.7–4.9 m, against 5.08; 19 of the 225 masses touch the floor, against 40.
- Settling: under 1 cm/s by about 8.7 s, and it stays there. The readout's max speed floor is about 2 mm/s — a microjitter in the corner crumples — and the largest position swing in a 10 s window is 17 mm, less than one pixel at the camera's distance. Still to the eye, to the millimeter, forever.

Why is it stiff? Because stiffness stopped being a force and became a fraction. A constraint that corrects a fraction k′ of its error each iteration can only *reduce* the error — the correction is along the error, by less than its length — so there is no overshoot, no ω, no `ω·dt < 2`. The RECALL's limit is not small at K = 1.0; it's gone. The stability story of Parts 1–3 — the eigenvalue, the matrix, the 5.8× margin — applies to a method that no longer exists in this file.

Why is it calm? Two reasons, both in the differentiate pass. Contact is a projection, so the table can take energy out (a mass is moved to the skin and its measured velocity agrees with the move) but can never put energy in — the slingshot failure mode of Act 1's contact springs, the one that needed the closest-point rewrite to fix, is structurally impossible. And the damping is COM-relative, so it damps the cloth *against itself* — exactly the wrinkle-and-ring energy that needs killing — without touching the throw.

Now the PREDICT you should have been making since the stability section: **how big can `STEP_DT` get before the PBD cloth misbehaves?** You have no ω to bound, so predict from the geometry instead. Measured:

- `1/60`: stable. The drape holds, the readout settles to 7.7% stretch with a 5.9 cm/s jiggle — a coarser cloth that never relaxes, because the projection at 1/60 corrects less per second than it stretches.
- `1/30`: still stable — no NaN, no explosion — but the cloth **falls through the table** and pancakes onto the floor, 18.6% stretched. The step is 33 mm long, and a mass moving at 3 m/s moves a whole step in one step: the closest-point check samples once per step, and the mass was over the disk at the start of the step and on the floor at the end.
- `1/10`: stable, and flat on the floor in a second.

So PBD removed the *stability* limit and left the *resolution* limit: the step still can't be longer than the distance a fast mass travels in a step, or contact samples between the table's faces. 1/240 in the file is not a stability margin — it's the same anti-tunneling choice Part 3 made as a margin, now doing the job itself.

And the cost, the number this part has been circling since "the clock starts the machine": 321.9 µs per step at 1/240 — **3.7× the force step** (four constraint passes over 1202 springs plus contact, against one force pass) — so 1287.6 µs per frame at 60 fps, 7.7% of the budget. The honest version of the trade: PBD is a slower step that tolerates a bigger step. At 1/60 it costs 328.6 µs per frame, the same as the force file costs at 1/240, and it's the stable, calm, 1.3% cloth. That's the trade in one line: you pay for stiffness in iterations per step, and you get the step size back.

## Checkpoint

> [!PREDICT]
> With the file closed, from memory: when does the first mass touch the table, what does the impact peak at, and what does the readout hold at rest — in both file states?

**Run this to verify your work so far:** build the file from the blocks in the sections above, in order — S1, S2a, S3a, S4, S5 for the force cloth, then the three replacements (constants and arrays, `addSpring`, `step()`) for the PBD cloth. Open it in a browser (double-click it, or `python3 -m http.server` from its folder and visit the page).

Expected, force file: first contact at 0.304 s at 2.98 m/s, peak stretch 22.4% at 0.81 s, settled by about 8 s, readout holding at `225 masses · 1202 springs · max stretch 12.7%`. Expected, PBD file: first contact at 0.304 s at 2.97 m/s, peak 2.0% at 0.90 s, settled by about 9 s, readout holding at `... max stretch 1.3%`.

The PREDICT answers: 0.30 s and 2.97 m/s in both files (the free-fall of 0.45 m, measured 0.304 s / 2.97–2.98 m/s); 22.4% force, 2.0% PBD; 12.7% force, 1.3% PBD.

Play: grab a hanging edge mass and lift it — the PBD cloth holds its shape while you lift it, and drops with a 2% ripple when you let go; the force file stretches around your hand as you drag. Flick a corner sideways and the PBD cloth rings for a second and is still; the force file's ripple takes longer and the readout peaks higher. Throw a corner hard: both files keep the throw's momentum (the damping is COM-relative in both), and the camera follows the cloth in both.

**Likely errors:**
- If the PBD file's readout holds near 10–15% and the cloth shimmers, the tension-only line is two-sided — `if (err < 0 && !s.bend) continue;` became `if (err < 0) continue;` or vanished. A bending spring that only pulls is a fold that can't crease, and the cloth braces instead of draping.
- If the PBD cloth slides off the table as one piece, `FRICTION` is 0 or the `onContact` block is missing: the impact's sideways shove has nothing to eat it, the same failure the force file fixed with `C_SLIDE`.
- If the PBD file blows up to NaN, `k'` is wrong — a decimal slip made `kp` exceed 1, and a projection that corrects *more* than its error overshoots, which is how you get a force-based instability back into a position-based step.
- If the force file's cloth slides off the table in one piece within a second, `C_SLIDE` is 0: the table has no tangential grip, and the impact's net sideways shove wins.
- If the force file's readout spikes past 90% on landing, `K_STRUCT` is still 100 with `STEP_DT` at 1/60 (or a decimal slip put it there): the stability section is why the file takes 1/240, and K = 400 at 1/80 is the NaN version of the same error.
- If the page freezes for seconds after a tab switch, the 0.1 s clamp is gone: the accumulator is catching up the whole hidden interval, and at 322 µs per PBD step a 1 s hiccup is 240 steps, 77 ms of catch-up, in a single frame.

The two one-shot render calls are now redundant — the loop draws every frame. They're harmless; leave them or delete them.

## What's next

This is the end of the series, and the four parts are one argument. Part 1: a pile of springs pretending to be matter, and the one number that decides whether the pretending works — `ω·dt < 2`. Part 2: two lists, three passes, and a sheet that taught you which springs do what. Part 3: the same step in 3D, a free body, and the accumulator coming back for stability. Part 4: the same scene computed two ways, and the wall the force-based method hits — stiffness against time — and the method that walks around it by making springs into position fixes.

The tools are all in the file: two flat arrays, a list of constraints, a step, a clock, and a readout that tells you when the pretending is failing. A rope is a line of masses with bending turned off. A jelly is a shell of them. A soft body is a volume, Part 3's lattice with a third axis. And the round table is the smallest curved-surface contact there is — the closest-point trick is the first step of the whole collision-detection field, and it's eleven lines in `step()`.

If you take one thing from the series, take the readout. Every part has a number that says *the model is lying to you* — 4.1% strain in the cube, 12.7% stretch in the cloth — and the skill is knowing which number, and measuring it, before the model gets a chance to look convincing.

## Exercises

All one-constant changes, all in the PBD file unless noted.

- [ ] **Half-stiff fibers.** `K_STRUCT = 0.5`. The cloth still drapes, but the tabletop region sags into a shallow dome and the readout holds at 2.4% against 1.3%. The fibers are now an *approximation* of the constraint, enforced 50% per step — and ITER of them land exactly on 0.5, which is the `k'` formula doing its job.
- [ ] **No shear springs.** `K_SHEAR = 0`. The in-plane fibers can't resist the diagonals, and the four corners swing out to radius ≈ 5.48 m — further than the baseline's 4.88 — while the readout climbs to 31.4%: the load that the diagonals carried in the baseline is now carried by the edge fibers at full stretch. "A square with no center," Part 2's thought experiment, arriving in production.
- [ ] **One iteration.** `ITER = 1`. The projections get one pass instead of four, and the cloth relaxes to 2.2% instead of 1.3% — the same cloth, less enforced, because each constraint is only partially satisfied per step. The `k'` formula is why this is a clean factor and not a blow-up: with `kp = 1 − (1 − k)^(1/1) = k`, one iteration lands exactly on k.
- [ ] **A smaller table.** `TABLE_R = 1.8`. The overhang goes from 1.8 m per edge to 2.4 m, the drape deepens (the cloth's center of mass drops to y = 1.22 m from 1.50), and the corner fan reaches out to 5.75 m. Same cloth, more overhang, more of the cloth on the floor.
- [ ] **A harder drop.** `DROP = 2.0`. First contact at 0.629 s (the free fall of 1.95 m, the PREDICT's arithmetic from the Part 3 checkpoint, now in cloth) at 6.15 m/s — four times the 0.5 m impact speed — and the peak stretch is 2.4%, barely above the 2.0% the 0.5 m drop makes. The PBD cloth takes a four-times harder slap and peaks at barely more. In the force file the same drop peaks at 9.6% and rests at 3.8%: stiffer-feeling, less stretch, because the harder slap loads the tension-only fibers and the drape that results is tauter — the force method's answer to "harder" is "stretchier," up to the wall of the stability section.
- [ ] **No gravity.** `G = 0`. The cloth floats where you drop it, and a flick gives the whole cloth a center-of-mass velocity that never changes — measured: 0.0089 m/s before, 0.0089 m/s eight seconds after. The COM-relative damping is doing exactly the job Part 3 assigned it: eat the ripples, keep the throw.

## Sources

Physics and simulation:

1. [Müller, Heidelberger, Hennix, Ratcliff, "Position Based Dynamics," SCA 2007](http://www.cs.toronto.edu/~jacobson/seminar/mueller-et-al-2007.pdf) — the PBD step (predict, project, differentiate); constraints with stiffness k ∈ [0..1] and equality/inequality types; the per-iteration stiffness `k' = 1 − (1 − k)^(1/ITER)`; collision as projection; the Verlet correspondence.
2. [Nealen, Beer, Teschner, "Physically Based Deformable Models for Computer Animation and Virtual Environments" (state-of-the-art report), Eurographics 2006](http://www.nealen.net/papers/pbdm.pdf) — cloth anisotropy ("resists bending weakly, but has a relatively strong resistance to stretching"); the stiff ODE and "excessively small time steps"; Provot's iterative relaxation; damping along the spring direction, not against the world.

Carried over from earlier parts (same URLs, same role):

3. [Cloth simulation — Wikipedia](https://en.wikipedia.org/wiki/Cloth_simulation) — the particle-mesh and mass-spring model families (Part 2's source for the three spring families).
4. [Hauth et al., "Cloth Animation and Rendering," Eurographics 2002 Tutorial T3](http://www.mirkosattler.de/publications/2002_tutorial_eg.pdf) — structural, shear, and bending spring families (Part 2's source for the families and the load-sharing quote).
5. [Semi-implicit Euler — Wikipedia](https://en.wikipedia.org/wiki/Semi-implicit_Euler_method) — the integrator behind the predict pass (Part 3's source).
6. [Center of mass — Wikipedia](https://en.wikipedia.org/wiki/Center_of_mass) — the COM-relative damping (Part 3's source).
7. [Fix Your Timestep — Glenn Fiedler (Gaffer on Games)](https://gafferonggames.com/post/fix_your_timestep/) — the accumulator and the spiral of death (Part 1's design note).
