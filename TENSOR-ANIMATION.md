# Building the Tensorfy tensor-field animation

A complete build spec for the hero animation on `tensorizeai.com` — a canvas-2D
tensor field that opens inside a single tensor, pulls back to reveal 616 of them,
orbits the field, then flies into a different one and resolves.

Everything here is 2D canvas. No WebGL, no libraries, no dependencies. The whole
thing is roughly 340 lines and runs at 60fps on a laptop.

---

## 1. The idea

The animation is an argument, not decoration. It says, in order:

1. **This is a tensor.** A single object, filling the screen.
2. **It is one of many.** Pull back; the field is enormous.
3. **They are enumerable.** Orbit; the count is finite and stated.
4. **We can address any one of them.** Fly to a specific tensor and resolve it.

If you build the motion without that argument, you get a screensaver. The
sequencing is the point.

---

## 2. The score

Four acts on a shared clock `T` (seconds since start). Three constants define
every boundary:

```js
var T_OUT = 4.6;    // pull-back completes
var T_ORBIT = 7.6;  // orbit completes, dive begins
var T_IN = 10.6;    // arrival; everything settles
```

| Window | Act | Camera distance | Field opacity | HUD label |
|---|---|---|---|---|
| 0 – 1.8 | Inside one tensor | 2.55 → ~12 | 0 | `TENSOR` |
| 1.8 – 4.6 | Pull back, field appears | → 34 | 0 → 1 | `ENUMERATE n=616` |
| 4.6 – 7.6 | Orbit the field | 34 (held) | 1 | `SURVEY 616 TENSORS` |
| 7.6 – 10.6 | Dive to the target | 34 → 4.6 | 1 → 0.12 | `ISOLATE 418/616` |
| 10.6+ | Settled | 4.6 | 0.12 | `RESOLVED` |

Expressed as keyframe tracks:

```js
var CAM   = [[0, 2.55], [T_OUT, 34], [T_ORBIT, 34], [T_IN, 4.6]];
var FIELD = [[0, 0], [1.2, 0], [3.4, 1], [T_IN - 1.0, 1], [T_IN, 0.12]];
var FOCUS = [[0, 0], [T_ORBIT, 0], [T_IN, 1]];   // 0 = hero tensor, 1 = target tensor
var FOCAL = 1.6;
```

`FIELD` holding at 0 until T=1.2 is deliberate. The field must not fade in while
the camera is still inside the first tensor, or the reveal has no snap.

---

## 3. Geometry

### The unit tensor — built once, reused by every box

A 7×7×7 lattice, so 343 nodes. Built once at init and re-projected each frame:

```js
var N = 7;
var LOC = [];
var lidx = function (x, y, z) { return (x * N + y) * N + z; };

for (var gx = 0; gx < N; gx++)
for (var gy = 0; gy < N; gy++)
for (var gz = 0; gz < N; gz++) {
  var u = gx / (N - 1) - 0.5,
      v = gy / (N - 1) - 0.5,
      w = gz / (N - 1) - 0.5;
  LOC.push({
    bx: u * 2, by: v * 2, bz: w * 2,
    // per-node phase, so the scalar each node displays drifts independently
    vph: ((gx * 31 + gy * 17 + gz * 7) % 100) / 100 * 6.2832,
    x: 0, y: 0, z: 0, px: 0, py: 0, pz: 0, vis: false
  });
}
```

The multipliers `31 / 17 / 7` are coprime, which is what stops the phase pattern
from visibly repeating along any axis.

### The field — 616 tensors on a jittered lattice

```js
var FX = 11, FY = 8, FZ = 7, SP = 6.0;   // 11 * 8 * 7 = 616
var boxes = [];

// deterministic jitter — a perfect grid reads as wallpaper
function jit(a, b, c, m) {
  return ((((a * 7 + b * 13 + c * 29) * m) % 23) / 23 - 0.5) * 1.5;
}

for (var i = 0; i < FX; i++)
for (var j = 0; j < FY; j++)
for (var k = 0; k < FZ; k++) {
  boxes.push({
    ox: (i - (FX - 1) / 2) * SP + jit(i, j, k, 3),
    oy: (j - (FY - 1) / 2) * SP + jit(i, j, k, 11),
    oz: (k - (FZ - 1) / 2) * SP + jit(i, j, k, 19),
    ph: ((i * 31 + j * 17 + k * 7) % 100) / 100 * 6.2832,  // spin phase
    sp: 0.09 + (((i * 13 + j * 7 + k * 11) % 7) / 7) * 0.15, // spin rate
    i: i, j: j, k: k, cx: 0, cy: 0, cz: 0, r: 0
  });
}
```

**Use deterministic jitter, not `Math.random()`.** The jitter must be identical
on every page load, otherwise the target tensor lands somewhere different each
time and the framing of the final shot is luck.

### Choosing the two tensors that matter

```js
// HERO — the one the camera starts inside: nearest the origin
var HERO = 0, bd = 1e9;
for (var b = 0; b < boxes.length; b++) {
  var B = boxes[b], d2 = B.ox * B.ox + B.oy * B.oy + B.oz * B.oz;
  if (d2 < bd) { bd = d2; HERO = b; }
}

// TARGET — the one it flies to: a fixed grid address, off-centre
var TARGET = HERO;
for (var b2 = 0; b2 < boxes.length; b2++) {
  if (boxes[b2].i === 7 && boxes[b2].j === 3 && boxes[b2].k === 4) { TARGET = b2; break; }
}
if (TARGET === HERO) TARGET = (HERO + 41) % boxes.length;
```

Picking the target by grid address `(7,3,4)` rather than by index keeps it
meaningful if you change the field dimensions. The fallback matters — if hero and
target collide, the dive has nowhere to go and the last act plays as a dead hold.

---

## 4. Camera

A hand-rolled yaw/pitch camera. Translate to the focus point, rotate, push back
by the camera distance, then perspective-divide.

```js
function cam(wx, wy, wz, out) {
  var x = wx - fx, y = wy - fy, z = wz - fz;   // fx,fy,fz = focus point
  var x1 = x * cY - z * sY;                    // yaw   (about Y)
  var z1 = x * sY + z * cY;
  var y1 = y * cP - z1 * sP;                   // pitch (about X)
  var z2 = y * sP + z1 * cP + camD;            // push back
  out[0] = x1; out[1] = y1; out[2] = z2;
}

// project
var SS = Math.min(W, H) * 0.5, cx = W / 2, cy = H / 2;
var f = FOCAL / v[2];
var screenX = cx + v[0] * f * SS;
var screenY = cy + v[1] * f * SS;
```

Scaling by `min(W,H) * 0.5` rather than by width is what makes the composition
survive a portrait phone. Scale by width and the field overflows vertically on
tall screens.

### The focus point glides between two tensors

```js
var fmix = key(T, FOCUS);      // 0 → 1 across the dive
var hb = boxes[HERO], tb = boxes[TARGET];
var fx = hb.ox + (tb.ox - hb.ox) * fmix;
var fy = hb.oy + (tb.oy - hb.oy) * fmix;
var fz = hb.oz + (tb.oz - hb.oz) * fmix;
```

The camera never "travels" in the sense of a path. The point it orbits slides
from one tensor to another while the distance collapses. That reads as flight
and costs three lerps.

---

## 5. Interpolation — and the one thing that matters

Two interpolators. Using the wrong one is the single most common way this
animation ends up feeling wrong.

```js
function key(T, ks) {          // linear, smoothstepped — for opacity, rates, mixes
  if (T <= ks[0][0]) return ks[0][1];
  for (var i = 1; i < ks.length; i++) {
    if (T <= ks[i][0]) {
      var a = ks[i-1], c = ks[i], u = (c[0] - a[0]) ? (T - a[0]) / (c[0] - a[0]) : 1;
      u = u * u * (3 - 2 * u);
      return a[1] + (c[1] - a[1]) * u;
    }
  }
  return ks[ks.length - 1][1];
}

function keyLog(T, ks) {       // geometric — for camera distance, ONLY
  if (T <= ks[0][0]) return ks[0][1];
  for (var i = 1; i < ks.length; i++) {
    if (T <= ks[i][0]) {
      var a = ks[i-1], c = ks[i], u = (c[0] - a[0]) ? (T - a[0]) / (c[0] - a[0]) : 1;
      u = u * u * (3 - 2 * u);
      return a[1] * Math.pow(c[1] / a[1], u);
    }
  }
  return ks[ks.length - 1][1];
}
```

**Camera distance must interpolate in log space.** The pull-back goes 2.55 → 34,
a 13× change. Interpolated linearly, the first half of the animation covers
2.55 → 18 and the second covers 18 → 34 — but perceived zoom is proportional, so
that reads as a violent rush followed by a stall. Geometric interpolation changes
distance by a constant *ratio* per frame, which is what "constant zoom speed"
actually means to an eye.

Everything else — opacity, rotation rate, blend factors — uses plain `key()`.

### Rotation is nearly frozen during the dive

```js
var rate = key(T, [
  [0, 0.11], [1.8, 0.17], [T_OUT, 0.30], [T_ORBIT, 0.30],
  [T_ORBIT + 0.7, 0.035],   // slam the brakes as the dive starts
  [T_IN, 0.025],
  [T_IN + 2.5, 0.10]        // ease back to a gentle drift
]);
yaw += rate * dt * (reducedMotion ? 0 : 1);
```

This was learned the hard way. Rotating at survey speed *while* diving makes the
target swing across frame and the shot feels like a camera operator losing
control. Cut rotation to ~12% of survey speed the moment the dive begins.

---

## 6. Rendering — level of detail

Four tiers, selected per box by projected radius `r = (FOCAL / z) * SS`:

| Condition | What is drawn | Cost |
|---|---|---|
| `z < 1.7` or off-screen | nothing | culled |
| `r < 2` | one dot, radius 0.7 | 1 arc |
| `r < 7` | one dot, radius `r * 0.32` | 1 arc |
| `r >= 7` | 12-edge wireframe cube | 8 projections, 12 lines |
| focus tensor, `fscale > 55` | full 7³ lattice, 343 nodes + scalars | ~1000 lines |

The margin test uses a radius-aware bound so large near boxes are not culled
while still partly visible:

```js
var mg = 70 + Bx.r * 2;
if (Bx.cx < -mg || Bx.cx > W + mg || Bx.cy < -mg || Bx.cy > H + mg) { Bx.r = 0; continue; }
```

### Cross-fading wireframe into lattice

The focus tensor is drawn twice during the handover — once as a wireframe box in
the field pass, once as a full lattice. Fade the wireframe out by exactly the
amount the lattice fades in:

```js
var fullMix = clamp((fscale - 55) / 50, 0, 1);
...
var dep = Math.min(v[2] / 60, 1);
var o = Math.pow(1 - dep, 1.6) * fieldA;
if (n === focusIdx && fullMix > 0) o *= (1 - fullMix);   // MUST come after `var o`
```

> **The bug to avoid.** If that multiply is written above the `var o` line, `var`
> hoisting makes it multiply `undefined` (yielding `NaN`), and the declaration
> then overwrites it. No error is thrown, the fade silently does nothing, and the
> handover pops. Order matters.

---

## 7. Performance: alpha-bucketed Path2D

The naive version issues one `stroke()` per edge. At 616 boxes × 12 edges that is
7,392 draw calls per frame and it will not hold 60fps.

Instead, quantize opacity into 7 levels, accumulate every line of a given level
into one `Path2D`, and issue 7 strokes total.

```js
var LV = 7, buckets = [], dots = [];
function resetBuckets() { for (var i = 0; i < LV; i++) { buckets[i] = null; dots[i] = null; } }

function bAdd(o, x1, y1, x2, y2) {
  var l = Math.floor(o * LV); if (l < 0) return; if (l > LV - 1) l = LV - 1;
  var p = buckets[l] || (buckets[l] = new Path2D());
  p.moveTo(x1, y1); p.lineTo(x2, y2);
}

function bFlush(rgb, max) {
  for (var i = 0; i < LV; i++) {
    if (!buckets[i]) continue;
    ctx.strokeStyle = "rgba(" + rgb + "," + (((i + 0.6) / LV) * max).toFixed(4) + ")";
    ctx.stroke(buckets[i]);
  }
  resetBuckets();
}
```

`dAdd` / `dFlush` do the same for dots using `arc()` and `fill()`.

Seven brightness levels is enough that banding is invisible against a near-black
background. Below about five it becomes visible as depth "steps".

---

## 8. The details that sell it

### Target brackets

Corner brackets, not a full rectangle — a full box reads as a UI element, corners
read as a reticle. Computed from the exact projected extent of the 8 cube
vertices, so they track the tensor as it spins:

```js
var bb = boxes[TARGET].bb, pad = 7;
var qx0 = bb[0] - pad, qy0 = bb[1] - pad, qx1 = bb[2] + pad, qy1 = bb[3] + pad;
var L = Math.max(Math.min(qx1 - qx0, qy1 - qy0) * 0.30, 5);
var bo = (T < T_ORBIT ? 0.66 : 0.66 * (1 - Math.pow((T - T_ORBIT) / (T_IN - T_ORBIT), 2)));
```

Bracket opacity decays quadratically through the dive, so it has faded out by
arrival rather than sitting on top of the resolved tensor.

### Travel streaks

Only while the camera is genuinely moving. Draw from the previous frame's
projected centre to the current one:

```js
var dive = (T > T_ORBIT && T < T_IN)
  ? Math.sin(Math.PI * (T - T_ORBIT) / (T_IN - T_ORBIT)) : 0;

if (dive > 0 && Bx.seen) {
  var sdx = Bx.cx - ocx, sdy = Bx.cy - ocy, sv = sdx * sdx + sdy * sdy;
  if (sv > 9) {
    var so = Math.min(Math.sqrt(sv) / 70, 1) * o * dive * 1.7;
    if (so > 0.02) bAdd(Math.min(so, 0.999), ocx, ocy, Bx.cx, Bx.cy);
  }
}
```

The `sin` envelope means streaks build and fade rather than switching on.

### Every node carries a number

This is what makes it read as a *tensor* rather than a lattice — the thing is
populated, not decorative. Gated on zoom so the labels never appear as noise:

```js
if (fullMix > 0.5 && fscale > 120) {
  ctx.font = "400 7.2px ui-monospace, SFMono-Regular, Menlo, monospace";
  ctx.textAlign = "center"; ctx.textBaseline = "middle";
  for (var g = 0; g < LOC.length; g++) {
    var gp = LOC[g]; if (!gp.vis) continue;
    var gd = clamp((gp.pz - near) / (far - near), 0, 1), go = 1 - gd;
    if (go < 0.16) continue;
    ctx.fillStyle = "rgba(159,182,216," + (0.16 + go * go * 0.78 * fullMix).toFixed(3) + ")";
    ctx.fillText((0.5 + 0.5 * Math.sin(gp.vph + T * 0.42)).toFixed(1), gp.px, gp.py - 7.5);
  }
}
```

> **The bug to avoid.** `gp.vph` must exist on every node from construction. If
> the render code references a property the constructor never set, every label
> renders `NaN` — and it renders 343 times, so it is impossible to miss visually
> but easy to miss in a test that only checks whether the *usage* is present.
> Verify the constructor, not the call site.

---

## 9. Lifecycle and accessibility

Three guards, all required:

```js
// 1 — respect the OS setting. RM freezes T at 999, i.e. the settled end state.
var RM = matchMedia("(prefers-reduced-motion: reduce)").matches;
var T = RM ? 999 : (now - t0) / 1000 - hidden;

// 2 — stop rendering when scrolled away
new IntersectionObserver(function (es) { visible = es[0].isIntersecting; },
  { threshold: 0 }).observe(cv);

// 3 — pause the clock on a hidden tab, so returning does not skip the whole score
var awayAt = 0;
document.addEventListener("visibilitychange", function () {
  if (document.hidden) awayAt = performance.now();
  else if (awayAt) { hidden += (performance.now() - awayAt) / 1000; last = 0; }
});
```

The third is the one people forget. Without it, switching tabs for 30 seconds
means the animation is over before the reader looks back.

Also clamp delta time — a long frame otherwise teleports the camera:

```js
var dt = last ? Math.min((now - last) / 1000, 0.05) : 0.016;
```

### Canvas sizing

```js
function size() {
  DPR = Math.min(window.devicePixelRatio || 1, 2);   // cap at 2; 3 costs 2.25× fill for no gain
  W = cv.clientWidth; H = cv.clientHeight;
  cv.width = Math.round(W * DPR);
  cv.height = Math.round(H * DPR);
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);            // now draw in CSS pixels
}
```

---

## 10. Markup and CSS

```html
<section id="hero">
  <canvas id="lattice" aria-hidden="true"></canvas>
  <div class="hero-fade" aria-hidden="true"></div>
  <div class="wrap"><!-- hero copy sits above --></div>
</section>
```

```css
#lattice {
  position: absolute; inset: 0;
  width: 100%; height: 100%;
  z-index: -1;
  opacity: 0;
  transition: opacity 1.1s cubic-bezier(.16,1,.3,1);
}
#lattice.on { opacity: .6 }   /* .6, not 1 — text has to win */
```

`aria-hidden="true"` is not optional. The canvas carries no information a screen
reader can use, and the label text is painted, not in the DOM.

---

## 11. Tuning guide

| Want | Change |
|---|---|
| More / fewer tensors | `FX, FY, FZ` — count is their product |
| Denser field | lower `SP` (spacing) |
| More detail per tensor | `N` — cost is O(N³), 7 is already 343 nodes |
| Slower reveal | raise `T_OUT` |
| Longer orbit | raise `T_ORBIT` |
| Slower dive | raise `T_IN` |
| Wider pull-back | raise the peak in `CAM` (34) |
| Closer final framing | lower the last `CAM` value (4.6) |
| Field brighter | the `max` argument to `bFlush` (0.62) |
| Earlier lattice handover | lower the `55` in `fullMix` |
| Calmer spin | scale down `sp` in the box constructor |
| Less mouse parallax | the `0.7` / `0.42` multipliers on `mx` / `my` |

---

## 12. Failure modes, in the order they will bite you

1. **`NaN` on every node label.** A property referenced in render but absent from
   the constructor. Check the constructor.
2. **The wireframe→lattice handover pops.** The `o *= (1 - fullMix)` line is
   above `var o`. Hoisting. Move it below.
3. **Zoom feels like it stalls then rushes.** Camera distance is on `key()`
   instead of `keyLog()`.
4. **The dive feels out of control.** Rotation rate was not cut at `T_ORBIT`.
5. **The field looks like wallpaper.** Jitter is missing, or is `Math.random()`
   and therefore differs per load.
6. **Frame rate collapses.** One `stroke()` per edge. Bucket by alpha.
7. **Animation is over before the reader returns.** No `visibilitychange` clock
   pause.
8. **Composition breaks on mobile.** Projection scaled by `W` instead of
   `min(W, H)`.
9. **Camera teleports after a stall.** `dt` not clamped.
10. **Text is unreadable over the field.** Canvas opacity at 1. Drop it to ~0.6
    and put a gradient scrim between canvas and copy.

---

## 13. Minimal working version

Roughly 90 lines — field, camera, LOD, bucketing. Drop it in a file and open it.

```html
<!doctype html>
<meta charset="utf-8">
<style>
  html,body{margin:0;height:100%;background:#050506;overflow:hidden}
  canvas{position:fixed;inset:0;width:100%;height:100%;opacity:.75}
</style>
<canvas id="c"></canvas>
<script>
var cv=document.getElementById("c"),ctx=cv.getContext("2d"),W,H,DPR;
function size(){DPR=Math.min(devicePixelRatio||1,2);W=cv.clientWidth;H=cv.clientHeight;
  cv.width=W*DPR|0;cv.height=H*DPR|0;ctx.setTransform(DPR,0,0,DPR,0,0);}
size();addEventListener("resize",size);

var FX=11,FY=8,FZ=7,SP=6,boxes=[];
function jit(a,b,c,m){return ((((a*7+b*13+c*29)*m)%23)/23-.5)*1.5;}
for(var i=0;i<FX;i++)for(var j=0;j<FY;j++)for(var k=0;k<FZ;k++)boxes.push({
  ox:(i-(FX-1)/2)*SP+jit(i,j,k,3), oy:(j-(FY-1)/2)*SP+jit(i,j,k,11),
  oz:(k-(FZ-1)/2)*SP+jit(i,j,k,19),
  ph:((i*31+j*17+k*7)%100)/100*6.2832, sp:.09+(((i*13+j*7+k*11)%7)/7)*.15});

var V=[[-1,-1,-1],[1,-1,-1],[1,1,-1],[-1,1,-1],[-1,-1,1],[1,-1,1],[1,1,1],[-1,1,1]];
var E=[[0,1],[1,2],[2,3],[3,0],[4,5],[5,6],[6,7],[7,4],[0,4],[1,5],[2,6],[3,7]];

var LV=7,bk=[];
function bAdd(o,x1,y1,x2,y2){var l=Math.min(LV-1,Math.max(0,o*LV|0));
  (bk[l]||(bk[l]=new Path2D())).moveTo(x1,y1);bk[l].lineTo(x2,y2);}
function bFlush(){for(var i=0;i<LV;i++){if(!bk[i])continue;
  ctx.strokeStyle="rgba(159,182,216,"+((i+.6)/LV*.62).toFixed(3)+")";ctx.stroke(bk[i]);}bk=[];}

function keyLog(T,ks){if(T<=ks[0][0])return ks[0][1];
  for(var i=1;i<ks.length;i++)if(T<=ks[i][0]){var a=ks[i-1],c=ks[i],
    u=(T-a[0])/(c[0]-a[0]);u=u*u*(3-2*u);return a[1]*Math.pow(c[1]/a[1],u);}
  return ks[ks.length-1][1];}

var t0=performance.now(),yaw=0,last=0,FOCAL=1.6;
requestAnimationFrame(function f(now){
  requestAnimationFrame(f);
  var dt=last?Math.min((now-last)/1000,.05):.016; last=now;
  var T=(now-t0)/1000, camD=keyLog(T,[[0,2.55],[4.6,34],[7.6,34],[10.6,4.6]]);
  yaw+=.22*dt;
  ctx.clearRect(0,0,W,H);
  var cY=Math.cos(yaw),sY=Math.sin(yaw),cP=Math.cos(.17),sP=Math.sin(.17),
      SS=Math.min(W,H)*.5,cx=W/2,cy=H/2,o3=[0,0,0];
  function cam(x,y,z){var x1=x*cY-z*sY,z1=x*sY+z*cY;
    o3[0]=x1;o3[1]=y*cP-z1*sP;o3[2]=y*sP+z1*cP+camD;}
  for(var n=0;n<boxes.length;n++){
    var B=boxes[n]; cam(B.ox,B.oy,B.oz);
    if(o3[2]<1.7)continue;
    var r=FOCAL/o3[2]*SS, px=cx+o3[0]*FOCAL/o3[2]*SS, py=cy+o3[1]*FOCAL/o3[2]*SS;
    var mg=70+r*2; if(px<-mg||px>W+mg||py<-mg||py>H+mg)continue;
    var op=Math.pow(1-Math.min(o3[2]/60,1),1.6);
    if(op<.02)continue;
    if(r<7){bAdd(op,px,py,px+.8,py);continue;}
    var a=B.ph+T*B.sp,c2=Math.cos(a),s2=Math.sin(a),sc=[],ok=1;
    for(var q=0;q<8;q++){var v=V[q],lx=v[0]*c2-v[2]*s2,lz=v[0]*s2+v[2]*c2;
      cam(B.ox+lx,B.oy+v[1],B.oz+lz);
      if(o3[2]<.2){ok=0;break;}
      sc.push(cx+o3[0]*FOCAL/o3[2]*SS, cy+o3[1]*FOCAL/o3[2]*SS);}
    if(!ok)continue;
    for(var e=0;e<12;e++)bAdd(op,sc[E[e][0]*2],sc[E[e][0]*2+1],sc[E[e][1]*2],sc[E[e][1]*2+1]);
  }
  ctx.lineWidth=1; bFlush();
});
</script>
```

That gives you the field, the camera, the log-space pull-back and the bucketing.
Add the focus-point lerp, the lattice handover, the brackets and the node scalars
on top of it in that order — each one is independently testable, and building
them in that order means you always have something on screen that works.

---

## 14. Where the real thing lives

`public/index.html` — search for `function lattice()`. About 340 lines, self
contained, no imports. The HUD readout in the bottom-right (`SURVEY 616 TENSORS`)
is painted onto the same canvas at the end of each frame.
