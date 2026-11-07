# WebGPU loudspeaker cabinet design API

A modern Web API implementation using WebGPU for high-performance, low-latency processing editing and FEM/BEM computation in the browser using GPU acceleration.

## Features

- GPU-accelerated computation
- GPU accelerated 2D design and 3d overview manipulation
- Real-time synthesis
- WebGPU-based compute shaders for acoustic simulations

## Getting Started

The app has no npm dependencies and no `package.json`. The tools are plain Node.js scripts that use only built-in modules.

Run the app locally from the `FFS/` folder:

```bash
cd FFS
node tools/serve.mjs          # serves src/ at http://127.0.0.1:8080/
```

Rebuild the single-file app (`FFS/FFS-Lite.html`) after editing anything under `src/`:

```bash
cd FFS
node tools/build.mjs
```

## FFS

As example for code and styling i have included compiled web based app called FFS, Loudspeaker enclosure designer and acoustic simulator.
FFS lacks GPU acceleration and is using the CPU, simple multithreading max 2 to 3 cpu's. for tasking FEM,BEM and MODES simulations.

**FFS directory**
- `/FFS/FFS-Lite.html` is an example program that used wasm, webworkers and has build in 2d / 3d designer and instant preview of cabinet behaviour'
- `/FFS/Designs/` contains examples designs bases on FFS data (json) structure, for example BR-118-60x60x70.ffs contains a simple Bass Reflex subwoofer. (*.ffs file consist of json data)




## OUR GOAL: FFS GPU accelerated Extension (API)
The goal is to extend FFS that takes the design (json) file and use webgpu for better and faster workings.
Or the better option would be a direct connection to FFS, but in its own 'sandbox' (a tab, iframe, panel or other means) to extend FFS as plugin for alternative GPU accelerated loudspeaker enclosure design and other useful acoustic simulations for example.

Right now the WASM API returns magnitude-only results (splDb, impedanceOhms, excursionMm) plus one derived value (excessGroupDelayMs), and the field solve (lite_field_solve) returns only scalar values at a single fixed mic point. That's enough for the current SPL/impedance/excursion/off-axis graphs, but it blocks a few visualizations that need either phase or spatial data the core likely already computes internally but doesn't surface.

**But we do have the `/FFS/Designs/` source *.ffs files which contains all the data for the enclosure. Which we can use**

Complex (phase) frequency response. For each swept frequency, in addition to the existing magnitude, return the phase (degrees, or real/imag parts) of both the acoustic response and impedance. This alone would unlock:
A real phase plot (currently only the group-delay derivative is exposed, not phase itself)
Impulse response, step response, waterfall/CSD and spectrogram views — all of these need a complex transfer function to inverse-FFT into the time domain; with magnitude only, there's nothing to build them from on the JS side.
Multi-angle / multi-point sampling in the 3D field solve. lite_field_solve(f, mode) currently evaluates one fixed mic location per call. Letting the caller pass a mic position/angle (or a small batch of them) per solve would let us build real polar/directivity plots and spherical radiation patterns from the FEM/BEM solve, instead of only the existing 7-point horizontal off-axis approximation in the 1D model.
Per-node/per-element spatial pressure (and ideally velocity) output from the field solve, sampled on the mesh surface — and if feasible, on a coarse grid around the cabinet — at a chosen frequency. This is the one needed for a pressure/radiation heatmap like the marketing renders (rainbow colormap over the 3D mesh). Wall Modes already does the equivalent for structural displacement (modes.amp per node); the field solve has no analogous per-node output today.

If exposing full per-node pressure fields is too heavy for now, (1) and (2) alone would already unlock most of the plot types (phase, group delay as a proper curve already works, impulse response, waterfall, spectrogram, and horizontal/vertical polar).  for the heatmap-style visualization specifically.

And ofcourse much faster computation of BEM, FEM and Modes simulation!


## FUNCTIONAL

Proposed shapes, matching the existing calling convention

All current exports take a JSON-stringified doc/args in, and return JSON out (via the lite_alloc/lite_result_ptr/lite_result_len pattern already used everywhere). The additions below are designed to slot into that same pattern and be additive — existing callers keep working untouched.

1. Phase — extend lite_compute's existing result object, no new export needed

lite_compute already has to produce the complex response internally to derive excessGroupDelayMs, so this is "stop discarding the imaginary part" rather than new computation. Add alongside the existing magnitude arrays in the same result JSON:

splPhaseDeg: number[] // parallel to splDb, one per swept frequency
impedancePhaseDeg: number[] // parallel to impedanceOhms
driverImpedancePhaseDeg: number[][] // parallel to driverImpedanceOhms

2. Multi-point / angle sampling for the field solve

Current: lite_field_solve(f, mode) → scalar result at one implicit mic point.Proposed: accept an optional array of sample points/angles so JS can build directivity/polar plots without re-meshing per angle:

lite_field_solve(f, mode, pointsJsonPtr, pointsJsonLen)
// input JSON: { points: [{ az?, el?, x?, y?, z? }, ...] } // whichever coordinate form fits the core's math
// output JSON: { points: [{ splDb, phaseDeg, impedanceOhms }, ...] } // one per input point, same order
Backward-compatible option: if pointsJsonLen === 0, behave exactly as today (single implicit mic point) — old call sites need no change.

3. Per-node spatial pressure, mirroring how lite_wall_modes already returns per-node data

lite_wall_modes already returns { nodes, groups, modes: [{ hz, amp per node, mot per node }] }. A sibling export reusing the same nodes indexing from lite_field_mesh would let the field solve return a colorable field the same way Wall Modes does:

lite_field_sample(f, mode)
// output JSON: { nodes (same indexing as lite_field_mesh's), pressureDb: number[], phaseDeg: number[] }
This is the one that would need the most new work on his side (spatial sampling, not just evaluating the existing point result) — worth flagging as optional/lower-priority relative to (1) and (2).


##API

It would be nice that more 'plugins' will come from other authors, So giving them all the data from API reading points. Would be a good addition.


## License

MIT


## Author

GIT @PVH-Archifact

