# Tiny Tectonics: implementation plan

Spec: [`docs/tectonics_model.tex`](docs/tectonics_model.tex), "A Kinematic Plate Tectonics Model for Procedural World Generation". Section numbers below refer to it.

The deliverable is one self-contained page, `index.html`, published as an artifact after each meaningful change. Bit-for-bit reproducibility is not a goal. The initial condition is seeded, and the simulation is CPU-deterministic, so a seed currently reproduces a run.

## Phases

| Phase | Content (spec section) | Status |
|---|---|---|
| 0 | Geodesic grid with O(1) point location, WebGL2 viewer, controls | done |
| 1 | Plate-frame crust (§2), exact rotation (§4), compositing: gap flood fill, subduction polarity per pair, accretion and collision, h_max spreading (§5), volcanic arcs (§5.3), isostatic elevation and erosion (§6) | done |
| 2 | Spherical-harmonic mantle with exact relaxation and subduction/insulation sources, gradient flow, hotspots (§7); torque balance with basal drag, slab pull, implicit contact resistance, 3P×3P direct solve (§8) | done, untuned |
| 3 | Rifting: strength field, damage growth and healing, candidate lines by Dijkstra, projected line tension, split (§10) | done, see findings 6–10 |
| 4 | Welding (§9) | done |
| 4b | Escape from the late two-plate state: subduction initiation, or rift seeding for plates with little perimeter | next, see finding 11 |
| 5 | Topographic body force, ridge push (§8.3, optional); history fields: fold direction, event kind/time, V (§2); export to next stage (§12) | later |
| 6 | Performance: sparse plate grids, GPU compositing | when the rules settle |

## Implementation choices that differ from or sharpen the spec

1. **Grid.** The spec suggests a cube-sphere or HEALPix grid. This build uses an icosahedral geodesic grid, as requested. O(1) lookup still holds: each face's lattice is the gnomonic image of its cells, so lookup is face pick → plane projection → lattice rounding. Cells vary in area by up to about 2×, so every sum is weighted by the cell area a_i.
2. **Candidate plates.** The spec says to test every plate whose bounding cap contains the cell. The code tests only the plates that owned the cell or its 1-ring last substep. This is equivalent while motion stays under one cell per substep, which automatic substepping guarantees (≤0.6 cell). The user's Δt therefore sets speed, not accuracy, and ω is re-solved every substep.
3. **Plate grid storage.** The spec wants sparse plate grids. The code uses a shared record pool (about N records) plus a dense per-plate index (4 bytes × N per plate). Fully sparse storage is deferred to phase 6.
4. **Mantle evaluation.** The harmonics (L = 6) are evaluated on a coarse geodesic grid (f = 16) and interpolated to the world grid with Gaussian weights. The surface gradient is a least-squares fit on the coarse grid.
5. **Overlap with neither cell subductable** is unspecified in §5.2. The code sends the lower-standing (lower elevation) cell down.
6. **Contact cells.** A boundary edge between p and q counts as contact while the pair has converged within t_slab. κ is K_coll·ℓ when neither side is subductable, otherwise K_sub·ℓ.

## Findings from running the spec

These are reproducible with the default seed (1), f = 64, Δt = 2 Myr.

1. **World-to-plate lookup does not prevent pinholes.** §5 claims it does. It holds only for crust that existed at t = 0. Crust created at a ridge occupies only the plate cells the world grid sampled at that moment. Once the plate has rotated by a fraction of a cell, later samples land between them. Measured in phase 0: pinholes were about 40% of all empty lookups (86k of 229k between 438 and 538 Myr). Fix: a pinhole is an empty plate cell with ≥4 of 6 occupied neighbours that was **never occupied**. It is filled by copying a neighbour. Consumed cells are tombstoned (`idx = -2`) and never healed. Without tombstones the healer resurrected consumed continent along every collision front, and continental area doubled over 1.5 Gyr.
2. **Erosion as a sink makes continental area decay.** The spec deletes eroded volume. Collisions stack crust, conserving volume but not area, and arcs are the only area source, so continents fell from 30% to 13% over 1.6 Gyr. **Deviation:** eroded thickness is shed to lower same-plate neighbours (sediment). Volume is conserved, and long-run continental area tends to (continental volume) / h₀. The "continental area" control is therefore really an initial volume, and arcs add to it slowly (about 29% → 40% over 1 Gyr at default settings).
3. **Thickened oceanic crust was immortal.** Seamounts and arcs between h_crit (15 km) and h₀ (35 km) could neither subduct nor erode, so oceans filled with permanent islands. **Deviation:** thickened oceanic crust relaxes toward h_oc by the same sediment-shedding rule.
4. **Sea level should be derived, not a parameter.** If continents relax to h₀ and E(h₀) = 0, every old continent sits exactly at sea level, a knife edge. Sea level is now solved each update from a fixed ocean volume (mean depth over the sphere; Earth ≈ 2.6 km). Land fraction emerges. As a bonus, young, shallow ocean basins raise sea level, the mechanism usually invoked for high Cretaceous sea levels.
5. **Without rifting the system jams.** Stiff collision contacts lock colliding plates into one rigid assembly. After about 800 Myr all surviving plates share one ω, at 2–10 km/Myr. The spec's supercontinent cycle depends on rifting (phase 3), so this is expected, not a defect of phase 2.

6. **Per-cell torques do balance.** The torque on each cell (drag at the solved ω, slab pull, its share of contact resistance) sums to about zero over each plate. The residual is 1–6% for plates of a few hundred cells or more, and larger only for plates under about 100 cells, where splitting contact forces between the two sides is crude. Plates smaller than 2% of the sphere are not tested for rifting. An earlier residual of 30–99% was a bookkeeping bug: a plate slot reused within the same check was tested before its velocity was re-solved.
7. **Rift lines along the plate edge cause runaway shattering.** Dijkstra prefers to run along a damaged rifted margin and then cut inward. Such a line is nearly free, so every rift made the next one cheaper: 1,700 rifts and 128 plates within about 1 Gyr. **Deviation:** perimeter cells may only be a line's endpoints. The spec's damage rate k_r(σ − S̄)/S̄ also diverges as S̄ → 0, so the fractional overload is capped at 5.
8. **Sediment relabelling was a continental-area leak.** Once rifting created many margins, oceanic cells pushed past h_crit by sediment became thin continental crust that never shed again: continental crust reached 76% while land fell. Sediment no longer relabels crust. Only merges do, as in the spec.
9. **Insulation of 0.02 never breaks a supercontinent.** The mantle under continents ran only about 0.1 above the oceanic mean, line loads stayed at 0.4–0.8, and the world stalled at 1 km/Myr. At 5× (β = 0.1, now the default), the sub-continental anomaly is about +0.45. Applied to a stalled supercontinent at 1.3 Gyr, that broke it within 100 Myr, and plate count then cycled 2 → 10 → 3 → 12 → 24. This supports the spec's §14 conjecture, but only from that starting point. From t = 0 the same setting gives the late stall described in finding 11.
10. **Arcs alone over-grow continents.** With slab pull ×3, continental crust reached 58% and the world locked harder. **Addition:** subduction erosion removes continental crust at the front cell at A_se·v per unit trench length (A_se = A_arc), so Andean-type margins roughly break even. The literature balance between arc addition and subduction erosion is cited from memory and should be checked. In practice it made little difference (about 42% → 44% at 3 Gyr), so the excess growth is mainly elsewhere.
11. **Open: the late two-plate state.** Seeds 1 and 2 both have a lively first ~800 Myr (30–60 plates, 20–45 km/Myr, continents reassembling), then end at about 2 Gyr as two hemisphere-sized plates moving 1–2 km/Myr with line loads below 1. Mechanism: welding continues until the 50% cap blocks it (a deviation that exists because a near-global plate has no perimeter to seed rifts from). Huge, continent-laden plates have drag proportional to area but slab pull only proportional to trench length (Forsyth & Uyeda), so they are slow, and slow plates carry little tension. The missing process is probably subduction initiation at old passive margins, which is how Earth breaks up large plates without rifting them.

## Untuned constants

These are ratios only (spec §13), set so early plate speeds fall in 15–100 km/Myr: F_sp = 6e5, K_coll = 2e5 km, K_sub = 2e3 km, U₀ = 40 km/Myr, α = 1, β = 0.02, A_arc = 0.35 km, hotspot birth 0.6 and thickening 0.6 km/Myr. Rifting and welding: S_cont = 7.5e5 (×3 for cratons after 500 Myr), S_oc = 4e6·(1 + √(age/10 Myr)), T_w = 0.5, k_r = 0.005 /Myr, ε = 0.15, K = 12 seeds, f_min = 0.1, v_weld = 5 km/Myr, t_weld = 20 Myr. The live "Dynamics" sliders scale slab pull, mantle flow, insulation (β), collision resistance (K_coll), and lithosphere strength (S_cont and S_oc).
