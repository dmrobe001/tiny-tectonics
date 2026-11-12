# Tiny Tectonics: implementation plan

Status: **provisional.** The LaTeX algorithm document has not reached the repo yet. Phase 0 was built without it. Phases 1 and later are placeholders, to be redone once the document arrives.

The deliverable is a single self-contained HTML page (`index.html`), published as an artifact after each meaningful change. Bit-for-bit reproducibility is not a goal. A seeded initial condition is still cheap to keep, so the seed is exposed.

## Phase 0: scaffold (done)

What exists and why:

| Piece | Choice | Why |
|---|---|---|
| Grid | Icosahedral geodesic grid of frequency `f`: N = 10 f² + 2 cells (12 pentagons, the rest hexagons). Built by subdividing each face as a flat lattice, then normalizing to the sphere. | Near-uniform cells. Because each face's lattice is the gnomonic (central) projection of its cells, **point location is O(1)**: pick the face, project onto its plane, round lattice coordinates. |
| Plate representation | Each plate is a rigid body with a cumulative rotation quaternion. Its crust is a raster on the *same* lattice in the plate's own body frame (`layer[p*N + bodyCell]`). | Motion is exact (no repeated resampling, so no numerical diffusion and no "stuck" sub-cell motion). World ownership is re-derived every step by looking up each world cell in nearby plates' body frames. |
| Kinematics | Fixed random Euler poles; speed 2–9 cm/yr, slower for continent-heavy plates. Automatic substepping keeps motion under ~0.6 cell per substep, so the user's time step controls speed, not accuracy. | Placeholder. |
| Overlap rule | Where ≥2 plates claim a cell, the higher-standing crust (by elevation) stays. Continental losers stack their full thickness onto the winner (collision). Oceanic losers are consumed and return 10 % of their volume to the winner (arc). | Placeholder. Conserves continental volume, not area. |
| Gap rule | No plate claims a cell: new 7 km oceanic crust on the departing plate. A pinhole with ≥4 of 6 filled neighbours in body frame is healed instead (lattice aliasing). | Placeholder. |
| Elevation | Seafloor: −2.6 − 0.33·√age km (half-space cooling). Continent: Airy isostasy, 0.5 + 0.15·(T − 35) km. Smooth blend between. | Standard textbook approximations. |
| Relaxation | Volume-conserving diffusion of thickness among continental cells (κ = 3000 km²/Myr). | Stands in for erosion and orogenic collapse. |
| Rendering | WebGL2. Each cell is drawn as a hexagon or pentagon fan. Per-cell colour and relief normal live in textures updated each step. | |
| UI | Time step, resolution, continental fraction, plate count, radius, seed; pause/step/reset; views (elevation, plates, seafloor age, thickness); drag/scroll/pinch orbit; collapsible panel. | |

Known phase-0 artefacts:

- Grid cells are not exactly equal-area. The gnomonic construction makes cells near icosahedron corners smaller, by up to about 2× in area. Fine for now. If it matters, a spring-relaxed grid would need a different point-location scheme.
- About 1 % of arbitrary points locate to a neighbour of the true nearest cell. The lattice nearest neighbour differs slightly from the spherical one. Harmless.
- No plate creation (rifting) or plate merging. Poles never change. Plate count only goes down.
- Continental area slowly shrinks through collisions (stacking). Arcs slowly make new crust.

## Phase 1+: to be planned from the algorithm document

Placeholders, ordered visual-first:

1. Replace placeholder rules with the document's rules, one mechanism per commit, each visible as a view mode.
2. Plate dynamics (forces → poles), rifting, suturing.
3. Move the per-cell step to the GPU (WebGPU compute, or WebGL2 transform feedback) once the rules settle.
4. Game-facing parameters (sea level, planet radius effects) and export.
