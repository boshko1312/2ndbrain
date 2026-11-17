# Particle field — audit of v1 and of the v2 rebuild

Deliverables in this folder:

| File | What it is |
|---|---|
| `particle-field-v5.html` | **Current version.** Nervous system audited and polished: domain-based layout, colour by domain, symmetric nerve tree, flowing fibres. See §9. |
| `particle-field-v4.html` | First nervous-system layout. See §8. |
| `particle-field-v3.html` | Dense-dust rebuild made after the reference screenshot arrived. See §7. |
| `particle-field-v2.html` | First rebuild: CPU N-body-style simulation, bright bloomed look. Kept for comparison. |
| `screenshots/` | Renders of v2 (`01`–`05`) and v3 (`v3-01`–`v3-05`). |
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

---

## 7. v3 — after the reference screenshot

The reference (a Mindcorpus clip) showed that v2 was aiming at the wrong look.
What the reference actually is, and where v2 missed:

| Reference | v2 |
|---|---|
| One cohesive, sculptural mass | A spread-out web of separate bright nodes |
| Hundreds of thousands of tiny grains; form comes from density | ~80k larger sprites; form comes from glow |
| Matte, muted leaf green, almost no bloom | Saturated cyan/green, heavy bloom, white cores |
| A "form ↔ layout" control | No equivalent |

The reference's actual shape is a head-and-torso figure, which is the thing you
said looked creepy, so v3 copies the *material* and not the silhouette.

### What changed

- **Motion moved to the GPU.** Each particle's position is a closed-form
  function of time evaluated in the vertex shader (a breathing swirl about its
  cluster axis, or a stream along a braided strand, plus a turbulence
  displacement). Nothing is simulated per particle on the CPU, so the default
  is 400k units on desktop (120k on phones) and the slider goes to 1M.
- **The cluster graph is still force-directed and live** on the main thread
  (72 nodes, exact O(C²)), driving the Repulsion and Link-tension sliders.
- **Form slider.** Scales the cluster layout from a merged single mass
  (`mass`) to the separated graph (`web`). Default 0.3.
- **Sculpted clouds.** Clusters are clouds, disks or rings, shaped by domain
  warping and carved with soft voids, plus a loose dust medium around each
  cluster so neighbours merge into one body.
- **Matte rendering.** Grain-sized sprites, flatter brightness distribution, a
  muted moss → leaf → teal palette, bloom at a low default, and a tonemap with
  a soft ceiling so density reads as tone instead of white-out.
- **Picking without a CPU copy of the positions.** The shader maths is mirrored
  in JS (`unitPos`) and evaluated only for the particles that need it: the
  clicked cluster's members on click, and the selected unit plus its links per
  frame.

### What was given up

This is a real trade, not a free upgrade:

- **No per-particle physics.** v2's particles orbited under gravity and tides;
  v3's follow authored motion. Core gravity, Web pull and Spiral arms are gone
  as sliders; Swirl, Turbulence, Strand flow and Cloud size replace them. Only
  Repulsion and Link tension are still genuine forces.
- **No per-unit hover.** Hover identifies clusters only; units are picked on
  click or from the inspector lists.

### Measurements (same software-rendered container as §3)

- 400k units at 1920×1080: 23–57 fps on the software rasteriser, against
  13–20 fps for v2 at 80k. Real GPU frame rate is still unmeasured.
- One-off particle generation: about 220 ms for 300k, 340 ms for 400k, 460 ms
  to grow to 600k. It runs on the main thread, so changing density causes a
  brief hitch.
- Background: all four corners `0,0,0`, 68% of pixels exactly black in the
  1920×1080 home render.
- Shader/JS mirror: for sparse outer-halo particles, the JS-predicted screen
  position landed on a lit pixel in 76 of 88 and 100 of 105 cases, against 2
  and 5 for a control point 11 px away. The misses are particles too dim to
  clear the detection threshold.
- Headless checks passed with no app errors: hover tooltip, click select, unit
  pick, unit selection from the list with link lines, orbit and zoom, search,
  density change to 600k with selection kept, new field, 390×844 touch
  viewport with tap-select.

### Where v3 still falls short of the reference

1. **Close-ups are grainy.** At home distance the mass reads as dense and
   sculpted. Zoomed well inside it, the grains separate and it looks like
   speckle with less contrast than the reference (`v3-03-closeup.png`). More
   particles help; the proper fix is depth-aware sizing or a density pre-pass.
2. **Faint stripe artefacts** in some small disk clusters when seen close,
   from the periodic function used to carve voids.
3. **The mass is lumpier and flatter than the reference's single strong
   form.** It is a merged graph, not a designed sculpture. A deliberate
   abstract target shape (a torus knot, a warped shell) that the clusters
   morph into would get closer, at the cost of the layout meaning less.
4. **Small clusters** still show as brighter smudges inside the mass.
5. Limitations 2, 3, 6, 7 and 8 from §6 still apply.

---

## 8. v4 — nervous-system layout

Requested explicitly, which overrides the earlier "no literal anatomy" brief.
v4 is v3 with one addition: an **Anatomy** slider that morphs the whole field
between the graph layout and a human nervous system. It opens at "nerves".

### How it maps onto the data

- A nerve tree is generated: brain tracts, spinal cord, rib nerves, a fan of
  nerves below the cord, four nerves per limb, digit branches, and three
  generations of fine side branches (about 1,460 of them).
- **Clusters sit on the tree.** The 14 largest become brain lobes (12 cerebral,
  2 cerebellar), the next 12 become ganglia along the spinal cord, and the
  remaining 46 are scattered over the peripheral nerves.
- **Links become fibres.** Each link's strand particles are laid along the tree
  path between its two clusters, so cross-body links run through the spinal
  cord. That is why the cord is the brightest structure.
- Peripheral clusters put most of their units along their nearest nerve
  branches rather than in a ball, so selecting one lights a region of nerves.
- Every particle stores a second, anatomical home position; the shader blends
  between the two with a per-particle stagger so the morph ripples.

There is no face, skull, skin or body outline — only nerves and a brain.

### Checks

- Same headless interaction suite as v3 passed in anatomy mode: hover, click
  select, unit pick, list selection with link lines, orbit and zoom, search,
  density change to 600k, new field, phone-sized touch viewport.
- Shader/JS mirror in anatomy mode: outer-halo particles hit a lit pixel at the
  predicted position in 96 of 105 cases (control 8) fully morphed, and 80 of 89
  (control 5) half-morphed.

### Limitations

1. **It is a stylised figure, not an anatomical atlas.** Nerve routes are
   plausible, not accurate; there are no cranial nerves and no named plexuses.
2. **Strand flow stops in anatomy mode.** Fibres hold their position along the
   nerve; only the brightness pulse and the turbulence shimmer still move.
3. **The brain is a smooth ovoid** with a central fissure and light surface
   folding, not convincing gyri.
4. **Which cluster lands on which body part is arbitrary** beyond size
   (largest to brain). A real mapping, such as domain to region, is a design
   decision that has not been made.
5. **Left and right are not symmetric** in the fine branches, and the figure
   can look ragged at the hands and feet.
6. All v3 limitations still apply.

---

## 9. v5 — audit of the v4 nervous system, and the polish pass

### Audit of v4

| Area | Finding in v4 | What v5 does |
|---|---|---|
| **Meaning** | Which cluster landed on which body part was arbitrary beyond "largest go to the brain". The picture could not be read. | A layout rule (below). Every region now means something, and a legend explains it. |
| **Colour** | One green family. Colour carried no information. | One hue per domain, so the body reads as coloured bands and the brain as coloured lobes. |
| **Brain** | A smooth ovoid with flat patches; from the front, a disc. Cerebellum was a detached blob. | A folded cortical shell with carved grooves, a narrower frontal pole, lower-hanging temporal lobes, a flat underside, and a striped cerebellum tucked beneath. |
| **Symmetry** | Left and right were grown independently, so fine branches differed and hands and feet looked ragged. | The midline and the right side are grown, then mirrored. |
| **Nerve tree** | Rib nerves, limb nerves and little else; empty torso and neck. | Adds dorsal root ganglia, cervical/brachial/lumbosacral plexus roots, a sympathetic chain, a vagus with cardiac and solar plexus, paired digital nerves, and a fourth generation of fine branches (about 2,500 side branches in total). |
| **Motion** | Fibres froze in anatomy mode; only brightness pulsed. | Fibres stream along their nerve route between clusters, and a signal wave travels from the brain down the cord and out to the extremities. |
| **Hover** | Picked clusters by centre point, so hovering a nerve belonging to a peripheral cluster did nothing. | Picks against 80 sampled member units per cluster, so nerves and lobes respond where they actually are. |
| **Controls** | Graph-only sliders (Form, Cloud size, Swirl, Repulsion, Link tension) stayed live but did nothing in anatomy mode. | They are shown as inactive in anatomy mode. |
| **Inspector** | Said "spiral disk" for a cluster drawn as a nerve. | Says where the cluster is: "brain, left hemisphere", "spinal ganglion, chest", "forearms & hands, right". |
| **Depth** | Flat: front and back of the figure equally bright. | The near side is brighter. |
| **Phone** | Header overlapped the brain once it grew. | Phone portrait reframes the figure below the header. |
| **Selection lines** | Link lines were heavier than the particles they connected. | Dimmed. |
| **Motion preference** | Auto-rotation ignored the OS reduced-motion setting. | Off by default when reduced motion is requested. |

### The layout rule (recommendation, implemented as the default)

Two questions decide where a cluster goes:

1. **How central is it?** decides how high up the hierarchy it sits.
   - The two **largest** clusters of each domain become **brain lobes**, one in
     each hemisphere. The brain is the set of hubs.
   - The two clusters of each domain with the **most links to other domains**
     become **spinal ganglia**. The cord is the set of connectors, and it is
     also where cross-body fibres physically run.
   - The remaining eight spread over the **peripheral nerves**, four on each
     side. These are the specialised topics.
2. **Which domain is it in?** decides which body band it lives in, the way a
   dermatome map assigns skin to spinal segments. Each domain owns one band,
   its spinal ganglia sit at that band's cord level, and its lobes share its
   colour.

| Band (head to toe) | Colour | Demo domain |
|---|---|---|
| Shoulders & upper arms | cyan | Systems & Code |
| Forearms & hands | violet | Craft & Design |
| Chest | emerald | Life & Ideas |
| Abdomen | gold | Work & Strategy |
| Hips & thighs | coral | Science |
| Lower legs & feet | magenta | Mind & Learning |

Unfiled units collect in the cerebellum and the outer dust.

Why this rule: it needs nothing that real notes do not already have (size,
links, a folder or tag), so it will carry over to real data unchanged; it keeps
related things together; and isolating a domain shows a complete circuit
(band, ganglia, lobes, and the fibres between them).

**What is still a judgement call:** which domain gets which band. The table is
a metaphor chosen for the demo (making things in the hands, and so on). With
real data I would order domains by size and give the largest the torso, unless
you want to assign them by hand.

### Colour

- **Colour range** slider: from the original mono green to one hue per domain.
  Default is nearly full.
- **Hue shift** rotates the whole palette.
- Darker hues are lifted so no domain looks weaker than another.
- Legend chips (head-to-toe order) isolate a domain when clicked.

### Measurements (same software-rendered container as §3; GPU frame rate still unmeasured)

- 400k units at 1920×1080: about 42 fps on the software rasteriser.
- Particle generation: about 450 ms for 400k; growing from 150k to 600k took
  760–880 ms. Slower than v4 because brain units are placed by rejection
  sampling. Still one main-thread hitch per density change.
- Background: corners `0,0,0`; 88.5% of pixels exactly black at home view.
- Shader/JS mirror, 10k units, 37 s of simulated time so fibres had moved:
  predicted positions landed on lit pixels for 97% of cluster units, 97% of
  fibre units and 90% of halo units fully morphed (control point 17 px away:
  54%, 44%, 28%). Half-morphed and graph mode gave similar margins.
- Interaction suite passed with no console warnings or errors: hover on a
  peripheral nerve, click select, unit pick, list selection, legend domain
  isolation, orbit and zoom, search, slider changes (anatomy off, mono colour,
  600k), new field, and a 390×844 touch viewport including tap-select and
  legend tap.

### Limitations

1. **Neighbouring clusters share nerves.** The four clusters on one side of a
   band are interleaved along the same branches, so hovering a nerve picks
   whichever has the nearest sampled unit. In the test, hovering a unit of
   "Writing" reported its neighbour "Game Design".
2. **Band assignment is a metaphor**, as above.
3. **Still stylised.** Routes are plausible, not atlas-accurate. There are no
   cranial nerves by design, since they would draw a face.
4. **Sulci are a pattern, not anatomy.** They are regular wavy grooves and do
   not match real fissures; at home distance they are barely visible.
5. **The brain is exposed down by a fixed factor** to keep lobes readable. At
   very low particle counts it can look dim.
6. **The signal wave is geometric**, based on distance from the brain. It does
   not follow real link structure.
7. **Fixed six bands.** A dataset with more or fewer than six domains needs the
   bands subdivided or merged; that is not built.
8. **Full colour range is busy.** If it reads as too much, the Colour range
   slider brings it back toward green.
9. All v3 limitations still apply (synthetic data, close-up grain, no importer).

## 10. Dashboard mockup — Body, forces, and a third audit pass

Added to `second-brain-os.html`:

- **Body shape** (eighth shape, works in Whole, Groups and Table, and is one body in all three). Brain, cerebellum, brainstem, 30 vertebrae, arm nerves from the neck, ribs from the chest, leg nerves from the base. Time runs down the body.
- **Forces group**: swirl (rigid for Neuron, Orbit, Shell; faster near the centre for Galaxy, Globe, Nebula), turbulence, breathe, spread. Every sprite carries its body's centre and radius, and the forces are computed in the vertex shader. The same maths runs in JavaScript for hover, selection ring, labels and link endpoints.
- **Slice viewer** for Scan: the slab drawn straight on in a small panel, with count and date range.

Checked in this pass (headless Chromium, software GL):

| Check | Result |
|---|---|
| All 24 shape × layout views render, no console errors | Pass |
| Predicted item positions match drawn dots under swirl 3, turbulence 3, breathe 3, spread 1.3 (160 items, frozen clock) | 128–151 of 160 predicted positions have a dot; 7–13 of 160 for an offset control |
| Tooltip appears on hover under maximum forces | 23 of 23 positions tried (Galaxy, Neuron, Globe groups) |
| Reset look restores force defaults | Pass |
| Scan switches itself off when leaving the Body | Pass |
| Full user flow (`utest.js`) and record-size flow (`gtest.js`) | Pass |
| Phone layout: Body view, Forces in the sheet, no horizontal overflow | Pass |

Fixed in this pass:

- Neuron and Body in Table were burnt out into blobs. Each view now measures how tightly it is packed and lowers brightness to match (Neuron table to 0.39, Body table to 0.65; most views unchanged).
- The first Body draft looked like a dome on a feather. Rebuilt with a fissure, cerebellum, brainstem, ribs and limb nerves.
- The slice viewer drew only the first 6,000 items it found, which filled one half of the brain. It now draws an even sample.
- Auto-scan applied to Body in Groups and Table, where one plane across many bodies means little. It now applies to the whole Body only, and only for records above 5,000 items.

Still rough:

- In dense areas hover can pick a neighbour of the item you meant.
- While scanning, hover only picks items inside the slice, so most of the Body does not respond to hover until Scan is off.
- Swirl does nothing on Body or Block; the panel says so, but the slider stays enabled.
- Scan position ignores the forces: with high spread or turbulence the lit slab is slightly off the drawn plane.
- Frame rate is still unmeasured on a real GPU.

### Follow-up: one body, not many

Body in Groups and Table first drew one small figure per group (8 figures, or 35 in the table). That contradicts the idea of a second brain, so both now draw a single body:

- **Groups**: groups are ordered by size (years by date). The leading groups whose share is closest to 40% become left-to-right slices of the brain; the rest each take a stretch of the spine in proportion to their share, newest item at the top of the stretch.
- **Table**: a flat outline (head, neck, shoulders, waist, legs). Rows take height in proportion to their share of the outline's area; columns are nested bands from the spine outward, mirrored left and right. Rows thinner than a label are named in the legend only.

The other shapes still draw one body per group in Groups and Table.

### Follow-up: Sunflower, colour by, swap

- **Sunflower shape** (ninth). Vogel's phyllotaxis model: item k (in date order) at angle k × 137.508° and radius √k. Equal area per item, no gaps or clumps at any record size, and a new item always lands on the rim, so growth playback fills it from the centre outward. Year rings are drawn at √(share of items up to that year). Thread lines join k to k + F for consecutive Fibonacci F, each family only in the radius band where those two items are nearest neighbours (around k ≈ F² / 5.6), which gives the alternating spiral bands of a real seed head.
- **Colour by**: colour can now stand for a different property than the grouping.
- **Swap** rows and columns in Table (disabled when the pair cannot be swapped, such as Project rows or Month-of-year columns).

Checked: all 27 shape × layout views render without errors; colour by Source changes the legend to Source, a legend row filters by Source, and the filter survives a change of grouping; swap exchanges Area × Year.

Rough: at the full 80,000 items the seed lattice is too fine to see until you zoom in; at that size the picture reads through the year rings, the spiral bands and colour. Size variety is reduced to 30% on this shape so the lattice shows.

### Follow-up: the Body did not look human

The first Body was built from formulas (a dome brain, hoop ribs, drooping stubs for limbs) and read as a mushroom on a fish; its flat Table version read as a figurine. It is now built on the nerve tree from `particle-field-v5.html`: brainstem and cord on the midline, twelve rib nerves, neck, shoulder, four strands down each arm, a pair of nerves per finger, hip, leg and toes, the chain beside the spine, and four rounds of finer branches, grown for the right side and mirrored. The brain is scaled to the figure (it was about twice too large).

- Items sit along the nerves. The tree is cut into short pieces sorted from head to toe, each weighted by length and thickness, so items spread evenly and any share of the record maps to a stretch of the body.
- Table no longer uses a separate flat outline. It uses the same body and the same stretches as Groups; inside a stretch, pieces are ordered left to right and split by the second property, and colour follows that property.
- While scanning, the unlit part of the Body stays at 20% in its own colour instead of 3% grey, so the figure does not disappear.

Checked: Whole, Groups and Table render on desktop and phone, and the full click-through flows pass.

### Follow-up: pick to isolate, chips, To AI, suggested links

Four interaction features, and a user audit of them.

- **Pick to isolate.** A click picks an item instead of opening it. Unrelated items drop to 4% brightness, picked items burn at 3.4× and connected ones at 2.4×, and rotation pauses while anything is picked. Dimmed items can still be clicked, so a selection can grow.
- **Chips.** Up to eight picked items show as chips beside the count; each can be removed, and Clear or Esc removes all. Filters come back when the selection is cleared.
- **To AI.** The toolbar on the last pick has Open, To AI, Zoom and remove. To AI fills an Ask tray with the picked items. Where the page runs as a Claude artifact, Ask sends titles, summaries and links to Claude on the viewer's account and streams the answer; elsewhere only Copy prompt is offered.
- **Suggested links.** Dashed lines to items that are not linked but probably related. In the mockup the rule is: same area, another project, written within about ten weeks. The item page lists them under "Might be related" and says how they were chosen.

Checked in a headless browser:

| Check | Result |
|---|---|
| Pick, second pick, remove a chip, Esc | Counts read "1 picked · 4 connected", "2 picked · 9 connected", back to "80,000 items" |
| Toolbar Open, Zoom, remove | Pass; Zoom moves the camera from 222 to 34 and the toolbar stays on the item |
| Suggested links toggle | Line count drops and returns |
| Pick with a legend filter active, then Esc | Filter count restored |
| Sunflower, Block groups, Globe table | Pick works in each |
| Ask with a stand-in model | Shows "Thinking…", streams, shows the answer; prompt contains the item title |
| Ask when rate limited / declined | Message shown and button kept / button hidden and copy offered |
| Phone: tap, toolbar, tray, item sheet | Pass, no horizontal overflow |
| Full regression flows | Pass |

Found and fixed during the audit:

- **Links ran off screen at small record sizes.** With the record shrunk (440 items), an item's links still pointed at items that are not in the record yet, whose positions are stale. Links now only go to items in the record. At 440 items the longest link is 17 units inside a body 28 units tall, and no link ends on a hidden item.
- With a selection active, nothing else could be clicked, because dimmed items were not pickable.
- The toolbar jittered with every small drift of its item. It now rests until the item moves more than 46 px.
- Only the last picked item had a ring. Every picked item has one now.
- The shape's own thread lines competed with the link lines; they dim to 16% while anything is picked.

Not checked: a real call to Claude from the published page. That needs a signed-in viewer to press Ask and accept the consent prompt.

Still rough: suggested links come from a rule, not from what the items say. On a phone the toolbar can cover a nearby label. In the densest areas a click can land on a neighbour of the item you meant.

