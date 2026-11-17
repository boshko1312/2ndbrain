# Particle field — audit of v1 and of the v2 rebuild

Deliverables in this folder:

| File | What it is |
|---|---|
| `particle-field-v2.html` | The rebuilt prototype. One file, no dependencies, opens straight from disk. |
| `screenshots/` | Renders of v2 (home view, close-up, inspector, cool colour temperature, mobile). |
| `README.md` | How to run it, controls, URL parameters. |
| `AUDIT.md` | This document. |

**A caveat up front.** `particle-field.html` (v1) was not in the repository — the
repo was empty when this work started. Part 1 is therefore an audit of the v1
*approach as it was described* (Three.js r128, CPU force loop, additive
`THREE.Points`, galaxy seed, three sliders), not a line-by-line code review.
Anything specific to v1's actual source is unverified.

---

## 1. Audit of the v1 approach

| Area | Finding | Why it hurt |
|---|---|---|
| **Structure** | One central attractor plus an orbital kick. | Every particle falls into the same well, so the only shape available is a single blob or disk. There is nothing for the eye to read as clusters, links or hierarchy, and nothing to select. This is the root cause of "no real graph structure". |
| **Repulsion** | "Cheap neighbour repulsion" on the main thread. | Either O(n²) (unusable past a few thousand) or sampled (noisy, and it visibly jitters). It also fights the attractor, so particles settle into a uniform fuzz instead of dense cores with sparse halos. |
| **Performance** | Simulation and rendering share the main thread. | Every simulation millisecond is stolen from the frame budget, and input handling stalls with it. 50k particles in a JS loop with per-particle `sqrt` and neighbour lookups cannot hold 60 fps this way. |
| **Glow** | Additive blending straight into an 8-bit canvas. | Additive blending on an LDR target clips: overlapping particles saturate to flat white/green almost immediately, so dense regions lose all internal detail, and you cannot turn brightness up without making that worse. There is no real bloom — "glow" is only the sprite's own falloff. This is why density and glow both "feel insufficient" at once. |
| **Density** | World-sized sprites, uniform brightness. | Uniform brightness means no depth cue and no sparkle. World-sized sprites become large blobs when zoomed in and vanish when zoomed out. |
| **Colour** | Flat palette. | No relationship between colour and data; no temperature control. |
| **UX** | Three sliders, no selection, no inspector, no touch handling. | It is a screensaver, not a knowledge tool. |
| **Code** | Three.js r128 from a CDN for what is one draw call. | ~600 KB dependency and a network requirement for a file meant to be opened directly; r128 is also several years old. |

Verdict: the weaknesses were architectural (one attractor, LDR additive
rendering, main-thread physics), not tuning problems. Rebuilding was cheaper
than patching.

---

## 2. What v2 is

### Data model
A synthetic knowledge graph: 6 domains → 72 topic clusters (Zipf-distributed
sizes, one large cluster per domain) → N units. Clusters are linked by ~107
edges (per-domain minimum spanning tree, local extras, cross-domain bridges).
Every particle is one unit and belongs to exactly one of three groups: a
cluster, a filament (a "bridge note" between two linked clusters), or the
unfiled outer halo.

### Simulation (Web Worker)
A two-level hierarchical approximation rather than a full Barnes–Hut tree:

1. **Cluster graph — exact O(C²) force-directed layout**, running live:
   pairwise repulsion, springs along edges, weak centring. This is what the
   *Repulsion* and *Link tension* sliders drive.
2. **Bound particles — softened N-body against cluster monopoles.** Each
   particle orbits its own cluster in a softened (Plummer) potential and feels
   the tidal field of the four most influential neighbouring clusters. Clusters
   act as invisible dark-matter-halo attractors. Additional terms:
   - radial-velocity damping, which circularises orbits while conserving
     angular momentum, so disks do not collapse;
   - a rotating two-armed logarithmic spiral potential on disk particles, which
     keeps spiral arms alive instead of letting them wind up;
   - damping along the disk normal to keep disks thin;
   - a weak spring toward each particle's home orbital radius (see §4).
3. **Filament particles** stream back and forth along braided, bowed strands
   between linked clusters, with a brightness pulse travelling along each edge.
4. **Turbulence** is a divergence-free (ABC) flow. Bound particles feel only
   the shear relative to their cluster centre.

The worker is created from a blob URL, so it works from `file://`. If workers
are unavailable the same code runs on the main thread. Position buffers are
transferred, not copied, in both directions.

### Rendering (raw WebGL2, no library)
1. Point sprites, additive, into an **RGBA16F HDR** target. Gaussian core plus
   soft skirt; power-law brightness so a few units sparkle and most form dust;
   per-particle twinkle; sub-pixel particles keep energy instead of size;
   sprite size grows sub-linearly on zoom so close-ups stay fine-grained.
2. **Dual-filter bloom** (3–6 mip levels chosen from buffer size) for the soft
   volumetric glow.
3. Composite: hue-preserving exponential tonemap, so dense cores stay
   green/cyan and only the very densest roll off toward white.

Falls back to RGBA8 if `EXT_color_buffer_float` is missing.

### Interaction
Orbit / pan / zoom with inertia (mouse, touch, pinch); hover tooltip for units
and clusters; click to select a cluster and fly to it; inspector with stats,
connected clusters, sample units, and per-unit link lines; search; labels;
keyboard shortcuts. Highlighting is a one-texel-per-group state texture read in
the vertex shader, so selection costs nothing per frame.

---

## 3. Measurements

Environment: a 4-core cloud container with **no GPU**. Rendering here goes
through SwiftShader (a software rasteriser), so the frame rates below are not
representative of a real machine and I have not measured GPU frame rate on real
hardware. The simulation timings are real CPU numbers and do transfer.

**Simulation step, single thread, Node, default sliders (mean of 60 steps):**

| Units | ms / step |
|---|---|
| 20,000 | 3.7 |
| 60,000 | 5.5 |
| 100,000 | 8.8 |

So the simulation itself fits a 60 Hz budget up to roughly 100k units on this
(modest) CPU. An earlier version of the step cost 14.8 ms at 60k; replacing
`Math.sin/cos` in the turbulence, filament and spiral terms with a 4096-entry
table and skipping the tidal sum for core particles brought it to 7.4 ms, and
the later differential-turbulence change to 5.5 ms.

**In the software-rendered browser (not representative):** 80k units at
1920×1080 ran at roughly 13–20 fps, with the worker step at 11–13 ms because
the rasteriser was competing for the same four cores.

**Stability, 30k units, 6 simulated minutes, default sliders:** median particle
radius per cluster went from 0.52–0.59 to 0.46–0.53 scale radii, 90th
percentile from ~2.4 to 2.65–3.0. With gravity, web pull, turbulence and spiral
all at maximum, clusters stay bound (90th percentile 8–12 scale radii, no
escapes, no NaNs). The cluster web radius stayed at 140–144 units throughout.

**Background:** in the 1920×1080 home render, all four corner pixels are
exactly `0,0,0` and about 60% of all pixels are exactly black.

**Checks run headlessly (all passed, no console errors):** hover tooltip, click
select, inspector, unit selection with link lines, orbit / wheel zoom /
right-drag pan, search and Enter-to-select, density slider re-seed (selection
preserved), colour temperature, new field, pause, 390×844 touch viewport with
tap-select and the Tune sheet, worker-blocked fallback, LDR fallback.

---

## 4. Problems found in my own first passes, and what changed

| Pass | Problem | Fix |
|---|---|---|
| 1 | Blown-out white cores, grey-green fog over the whole frame. | Lower gain, hue-preserving tonemap, far-field bloom cut. |
| 2 | Over-corrected: looked like ~5k sparse dots, because most sub-pixel particles faded to nothing. | Energy floor for sub-pixel sprites, flatter brightness distribution, so the faint majority adds up to dust. |
| 3 | Close-ups showed large soft balls, not fine particles. | Sprite size grows as zoom^0.42 instead of linearly. |
| 3 | Most clusters were featureless spheroids. | 80% of clusters are now disk-dominated; 20% remain spheroids for variety. |
| 4 | Simulation was trig-bound (14.8 ms at 60k). | Sine table, tides skipped inside 1.2 scale radii. |
| 5 | Long-run test: cores slowly contracted (median 0.57 → 0.24 in 6 min), and extreme sliders dissolved clusters entirely. | Turbulence now applies only shear relative to the cluster centre; added the home-radius spring; removed the spiral term's tangential torque. |
| 6 | On small / low-resolution buffers the widest bloom level covered the screen as a haze band. | Bloom level count now scales with buffer size. |
| 6 | Unit link lines were heavy and dominated the cluster. | Lower intensity. |
| 7 | Bloom cut left a visible hard contour around structures. | Smooth fade instead of a hard subtract. |

---

## 5. Against the references

**Galactic structure — good.** Edge-on disks read as bright streaks, inclined
disks as ellipses, face-on disks show two-armed spirals, and the largest
clusters have distinct core / disk / halo. Limitation: the arms are driven by
an imposed pattern, not self-gravity, so every spiral is a tidy two-armed
grand-design type. No bars, no flocculent arms, no mergers.

**Cosmic web / dark-matter halo — good at mid zoom.** Nodes, filaments and
voids are clearly legible in the close-up. Limitation: the filaments are
kinematic strands along graph edges. They do not form by gravitational
collapse, so there are no sheets or walls and the web is only as rich as the
edge list.

**Neural field — partial.** Braided strands and travelling pulses give a
signalling feel, but nothing actually propagates through the graph: pulses are
per-edge and independent.

**Dense particle-cloud videos — partial.** At 80k units the cores and filaments
are dense, but the space between structures is sparse by design. Reference
clips of that style typically use hundreds of thousands to millions of
GPU-simulated particles; 80k CPU-simulated units cannot fill volume the same
way. See limitation 1.

**Non-creepy — yes.** Nothing in the generator can produce bilateral symmetry,
a silhouette or a face: cluster positions come from a force-directed layout of
a random graph, and disk orientations are random.

---

## 6. Remaining limitations

1. **Particle count ceiling.** The CPU worker is comfortable to ~100k. Going to
   500k+ for the truly volumetric look needs the simulation moved to the GPU
   (transform feedback or WebGPU compute). The force model was kept
   per-particle-independent partly so that port is straightforward.
2. **GPU frame rate is unmeasured.** Everything here ran on a software
   rasteriser. The adaptive resolution scaler (steps down below 27 fps, never
   back up) is the safety net; it should be checked on a real phone and a real
   integrated GPU.
3. **The data is synthetic.** Cluster names, unit titles and links are
   generated; duplicate titles occur. There is no importer yet. The natural
   next step is loading `{nodes, links}` JSON and deriving clusters by
   community detection.
4. **Not a true Barnes–Hut N-body.** Particles do not attract each other; they
   respond to cluster monopoles. That is why it is fast and stable, and also
   why there is no emergent clumping below cluster scale.
5. **Unit picking is a linear scan** of all particles per hover frame on the
   main thread. Fine at 80k; would need a spatial index or GPU pick buffer at
   much higher counts.
6. **Cluster count is fixed at 72** regardless of density, so at 5k units
   clusters are thin (~50 units each).
7. **Sorting / occlusion.** Everything is additive with no depth, which is
   right for this look but means a foreground cluster cannot hide one behind it.
8. **Accessibility.** The canvas has no keyboard path to selection other than
   search, and there is no reduced-motion handling.
