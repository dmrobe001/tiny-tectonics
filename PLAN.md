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
| 4a | Lithosphere: mantle-lithosphere thickness m, cratons as seeded roots, crust foundering, strength from m and h | done, see findings 12–14 |
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

## Lithosphere model (phase 4a)

This replaces the time-since-event "cratonisation" ramp, the age clock in drag coupling, the separate √age seafloor strength, and the subduction erosion I added earlier.

- **State.** Crust thickness h (as before), plus mantle-lithosphere thickness m: carried per record for continents, derived for seafloor as m = min(100, 10 + 9√age) km.
- **Cratons.** At t = 0 a Poisson number of patches (mean 2, at least 1) covers a chosen share of the initial continents (slider, default 25%) with m up to 250 km, tapering over 300 km to the ordinary continental value m_eq = 150 km. Nothing later creates m above m_eq.
- **Strength.** S = S₀(type) · (m/100) · exp(−max(h − 35, 0)/15) · (1 − d) · exp(−max(T, 0)/T_w). Thick lids are strong; thick, hot, quartz-rich crust is weak, so orogens are weak belts.
- **Drag.** Coupling c = C_oc · max(0.3, (m/100)²).
- **Destruction, never reset.**
  - A rift line and its margin multiply m by 0.5.
  - m above m_eq decays toward m_eq with τ = 1000 Myr while an active slab lies beneath (refertilisation, modelled on the North China craton; cited from memory).
  - Thinned continental lithosphere regrows toward m_eq (τ = 200 Myr) but never beyond it.
- **Foundering.** Crust below 60 km relaxes away with τ = 30 Myr: a volume sink tied to collisions. The spec's lateral spreading above h_max = 70 km is kept for plateaus.
- **Collisions.** Mantle lithosphere does not stack; the lower plate's mantle part sinks with its slab.

12. **The spec's pair polarity destroyed cratons within 200 Myr.** In an overlap the lower cell is deleted, with its root, and the pair's polarity is set once by elevation, which cratons share with every continent. A craton that arrived at a trench as the passenger of a subducting plate was deleted cell by cell. Its crust was plastered onto the overriding seafloor, which became continent with oceanic-thickness lithosphere. Measured: 226k of 261k km·cells of root excess lost to overlaps by 160 Myr. **Deviation:** per-cell overrides of pair polarity. If exactly one cell is subductable, it goes down (buoyant crust clogs the trench, as in subduction polarity reversal). If neither is, the thinner lithosphere is consumed and its crust stacks onto the thicker. Pair polarity still governs ocean–ocean overlaps, where flicker would matter.
13. **Craton loss channels after that fix** (seed 1, to 1600 Myr, refertilisation τ = 300 Myr): refertilisation 117k, rifting 94k, consumption by thicker roots 58k; half-life about 400 Myr. Refertilisation was then slowed to τ = 1000 Myr.
14. **Rift lines favour thin lithosphere.** 75% of 456 splits (seed 1) ran through lithosphere thinner than the plate mean, at 0.87× the mean on average.

15. **Open: continents erode away under the lithosphere model.** With all phase-4a changes on, continental crust falls from about 24% to 1–3% of the sphere by 1 Gyr (seeds 1 and 2). Plates speed up to 60–130 km/Myr and almost never weld. Diagnosis by bisection (seed 1, 400 Myr; every change has a switch in `K_`):
    - All changes switched to the previous behaviour: 28.7% continental crust, 24 welds.
    - Re-enabling a single change: strength law, drag law, arc fronts from slabs, foundering, or refertilisation and rift thinning each keep 25–29%. Plateau spreading within crust type drops it to 10% with 0 welds; the per-cell polarity override drops it to 12% with 4 welds.
    - Interpretation: the previous slowdown and welding relied on two artifacts that clogged trenches with buoyant crust (continental spill onto seafloor; continents plastered onto overriding seafloor). Removing them removed the only brake on convergence, and continuous continental collision stacks crust and destroys area.
    - A physical replacement, stiff contact resistance when the subducting plate presents buoyant crust ("trench clogging", `K_.CLOG`), only partly helps: 27% at 200 Myr, but 3% by 1 Gyr. Neither culprit reverted on top of clogging fixes it either (6% and 0.9%). Plate counts in failing runs climb to 90–120, against about 60 in the previous version.
    - **Resolved.** A record-level budget (continental records in plate frames, immune to aliasing; seed 1) showed that about 90% of the loss is continental crust consumed by continental crust: about 3,700 records per 200 Myr, roughly 9% of the sphere. Only about 20% of it is between rift siblings younger than 100 Myr. Two missing physics: collisions never stopped (fronts kept converging at about 40 km/Myr), and nothing restored the area that stacking converts to thickness. Fixes:
      - **Orogenic spreading** (`K_.SPILL_RELABEL`): continental crust above h_max spreads onto adjacent same-plate seafloor too, and a recipient pushed past h_crit becomes continental (the spec's merged-crust rule).
      - **Stiffer collision contacts**: K_coll back up from 5e4 to 1e5.
    - Results to 2 Gyr (seeds 1 and 2): continental crust 26% → 42–45%; about 50 plates at 800 Myr. Collision ×4 also works but jams earlier (4–9 plates by 1200 Myr, against 27–46 at ×2).
16. **The late jam is back (finding 11), now with continents intact.** By about 1.6 Gyr every run settles into 5–13 plates moving 2–3 km/Myr. Continental crust also keeps growing (to about 45%), and craton area declines from about 5% to 0.6–1.4% of the sphere by 2 Gyr. Next candidates: subduction initiation at old passive margins, propagating rifts, and a continental volume sink to balance arcs.

17. **Subduction initiation at old passive margins** (addition, phase 4b). Within a plate, the margin cells (seafloor next to continental crust of the same plate) separate each continental block from its seafloor. Each block initiates at rate λ·L·max(0, (age − 60)/60) per Myr, with λ = 2.5×10⁻⁶ /km/Myr, L the margin length, and age the margin seafloor's mean age. The block becomes its own plate with the same orientation; the seafloor side is recorded as the lower plate and its margin cells get slab flags. A live slider scales λ. The first version required the same sizes as rifting and almost never fired at the jam: there, the dominant plate (44% of the sphere) carried more than a dozen continental blocks of 100–770 cells, all below the 2%-of-sphere minimum. Now a block needs only 0.5% of the sphere; the remaining ocean plate must keep 10% of the plate. Result (seeds 1 and 2): 60–80 initiations per 2 Gyr, and lively plates (43–51) through 1.6 Gyr instead of 14–16. By 2 Gyr it still jams (8–15 plates, 3–5 km/Myr).
18. **Orogenic spreading relabelled at h_crit: another thin-continent leak.** Spreading relabelled seafloor continental at 15 km, so continental area grew toward volume/15 km and reached 38% even with arcs off. Relabel now happens only at h₀ = 35 km. Continental crust then stays at 20–29% to 2 Gyr. The late jam is not explained by continental growth alone.

19. **No supercontinent cycle by 3.5 Gyr.** Seeds 1 and 2 stay at 5–11 plates, 1–8 km/Myr from 2 to 3.5 Gyr. Initiation keeps firing (209 and 132 events in total), but welds outpace rifts plus initiations (seed 1: 686 welds against 560 rifts and 209 initiations). Continental crust grows again during the stall (seed 2: 34% → 52%, with land at about 38%), so a thin-continent source remains. Likely structural cause: the weld criterion (mean relative speed below 5 km/Myr for 20 Myr) measures absolute slowness, not the end of a collision. Once a world slows down, every continental contact qualifies, which is self-reinforcing: slow → weld → bigger plates → slower.

20. **Archipelagos were mostly rift and collision debris, not weakness from damage.** Census of connected continental pieces (seed 1, 400 Myr): 163 pieces and 125 specks (<10 cells) at baseline. With damage ignored by strength: 204 and 160. With no rift splits: 25 and 21. Arcs, hotspots, spreading and initiation made no difference. The specks were 75–95% inherited crust, and their most common last event was a rift. Per-split accounting found about 2.4 new specks per split, all bordering the rift line on the departing side. Mechanism: the cheapest rift line stays on land (continent is 20–30× weaker than seafloor) and hugs coastlines, cutting off land strips. Fixes:
    - continental fragments on side B smaller than 0.1% of the sphere that border the line stay with side A;
    - stray pockets go to the side they border across the line;
    - continent–continent consumption uses one polarity per pair, decided at first contact by lithosphere thickness, overridden per cell only by a root more than 50 km thicker. Per-cell decisions had left checkerboard remnants.

    Result at 400 Myr: 83 pieces and 56 specks. About half the remaining specks by 800 Myr are crust formed during the run (island arcs, plateaus).

21. **Welding only when a collision has ended removes the late stall.** Old rule (the spec's: mean relative speed below 5 km/Myr for 20 Myr at a continental contact): from about 2 Gyr, 6–15 slow plates, welds outnumbering rifts, continents growing to 45–50%. New rule (`WELD_RULE: 'ended'`): a pair welds only if it stacked continental crust within 100 Myr, its peak contact speed while stacking reached V_coll_min, and it has slowed below min(5 km/Myr, f·peak) for 20 Myr.
    - With V_coll_min = 10 km/Myr and f = 0.3 it overshot: about 95 plates, rifts:welds about 3:1, continents eroding from 25% to 11% by 3 Gyr, cratons almost gone.
    - With V_coll_min = 5 km/Myr and f = 0.5 (now the default), both seeds reach a steady regime through 3 Gyr. Seed 1: 64–79 plates, 10–19 km/Myr, continents flat at 23–25%. Seed 2: 67–86 plates, 15–23 km/Myr, continents 18–20%.
    - Doubling lithosphere strength barely changed the rift count.
    - Still open: craton area falls from about 4% to 0.5–0.7% of the sphere by 3 Gyr. Plate counts (about 70) are high against Earth's roughly 15 major plates, but most are small. No distinct supercontinent cycle is visible in plate counts sampled every 500 Myr.

22. **Assembly measures show no supercontinent cycle (to 4 Gyr).** Two measures are recorded every 10 Myr and drawn in the page:
    - clustering order parameter R = |area-weighted mean position of continental crust| / (1 − f), with f the continental fraction, so that R = 1 is one compact spherical cap;
    - largest landmass share L, the largest connected continental area over all continental area.

    Seeds 1 and 2: R falls from 0.50–0.62 at the start to a mean of 0.12–0.14 after 200 Myr (sd 0.07–0.08, max 0.41), and L stays at a mean of 0.13–0.15 (max 0.42). Plate count is steady at about 70. Clustering episodes (R above mean + 1 sd): 3 and 5, irregular, the last at 1.56 and 3.37 Gyr. The autocorrelation of R peaks at 320 and 420 Myr with r = 0.42 and 0.15: weak quasi-periodicity at best. Baseline: n equal pieces placed at random give an expected R of about 1.2/√n, so with roughly 100 effective pieces the observed R is about what chance alone produces. Craton area falls from about 6.5% to 0.3–0.4%, with losses weakly concentrated in dispersed phases (correlation of craton change with R: −0.31 and −0.18). Hypothesis to test next: the spec's assembly mechanism (continents dragged toward cold downwellings over subduction zones by mantle flow) is too weak against the randomising effect of about 70 independently moving plates.

## Minimal cycle model (`cycle.html`)

A deliberately non-physical toy, built to find the smallest set of ingredients that produces a supercontinent cycle.

- **Continents:** N rigid caps doing overdamped Brownian motion on the sphere, dx = μ(−∇T)dt + √(2D dt) ξ, with overlaps resolved by position projection.
- **Mantle:** degree 1..L harmonics, T = T_c + T_h, each relaxed exactly. The cold field (memory τ_c = 30 Myr) has source −γ Σ a_cap Y_lm(x_k), standing in for slabs at continental margins. The hot field (memory τ = 300 Myr) has source +β Σ w_k a_cap Y_lm(x_k), insulation weighted by w_k = (s_k/N)^p, with s_k the size of continent k's touching cluster.

What was needed (seeds 1–4, 5 Gyr):
1. **Insulation alone gives no cycle.** Self-heating only ever repels, so the evenly spread state is stable and the field decays to rms about 0.01.
2. **Fast cold attraction plus slow hot repulsion** (activator with delayed inhibitor) gives aggregation and oscillation, but with a linear blanket (p = 0) the cluster never fully disperses, and stronger insulation suppresses assembly instead.
3. **A blanket nonlinearity** (insulation grows with the blanket's width; p = 0.5 is the default) gives robust cycles on all four seeds: 11–14 assemblies in 5 Gyr, mean intervals 200–440 Myr. Assemblies are counted when the largest cluster reaches 90% of the continents after dropping below 70%. With p = 1, one seed locked into two antipodal clusters, a natural fixed point of a degree-2 field.
4. **Geometry:** successive assemblies form a mean 97° apart for L = 2 (69–119° across seeds) and 115° for L = 1 (80–139°). Degree 2 pulls toward the "orthoversion" 90°, but the contrast is weaker than the clean 90°-versus-180° I predicted.

## Untuned constants

These are ratios only (spec §13), set so early plate speeds fall in 15–100 km/Myr: F_sp = 6e5, K_coll = 2e5 km, K_sub = 2e3 km, U₀ = 40 km/Myr, α = 1, β = 0.02, A_arc = 0.35 km, hotspot birth 0.6 and thickening 0.6 km/Myr. Rifting and welding: S_cont = 7.5e5 and S_oc = 1.5e7 at m = 100 km (see the lithosphere model), T_w = 0.5, k_r = 0.005 /Myr, ε = 0.15, K = 12 seeds, f_min = 0.1, v_weld = 5 km/Myr, t_weld = 20 Myr. The live "Dynamics" sliders scale slab pull, mantle flow, insulation (β), collision resistance (K_coll), and lithosphere strength (S_cont and S_oc).
