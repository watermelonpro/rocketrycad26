# Camera Pod Drag Increment (dCd) — SolidWorks Flow Simulation

## Method

Two SolidWorks Flow Simulation External Flow projects were compared within the assembly
`Full Rocket Assembly.SLDASM`: **Rocket Only** (bare airframe) and **Camera Mount Test**
(airframe with two camera pods added). Both projects were audited to confirm identical
computational domain, global mesh settings, and inlet conditions except for the camera pod
geometry and an additional local mesh refinement box around the pods in Camera Mount Test.
The computational domain was found to be undersized relative to the recommended minimum
clearances (0.5 m upstream / 5.0 m downstream / 0.5 m lateral) and was enlarged identically
in both projects before solving. A drag-coefficient equation goal was rebuilt in both
projects, referencing the live `GG Force (Y) 1` force goal:
`Cd = 2*abs({GG Force (Y) 1})/(rho*V^2*A_ref)`, evaluated in SI units regardless of the
model's MMGS display units, with A_ref equal to the rocket's own cross-sectional area only
(camera frontal area was never used as a reference area). Both projects were solved to
solver-reported convergence (goal delta below the 0.5 N convergence criterion, and current
vs. averaged force agreeing to well under 1.0 N). Drag coefficients for the clean and
camera-equipped configurations were computed from the **averaged** (not instantaneous) force
values, and the incremental drag coefficient dCd was taken as the difference between the two.

## Step 1 — Audit (pre-enlargement)

| Parameter | Rocket Only | Camera Mount Test |
|---|---|---|
| Domain X | −0.489081938 to 0.448641201 m | same |
| Domain Y | −0.82083068 to 2.88010912 m | same |
| Domain Z | −0.743365462 to 0.624013432 m | same |
| Global Mesh | Automatic, Level 4, min gap 0.2480082 m, ratio 0.7 | same |
| Local Mesh | none | Local Mesh 1: box X ±0.0257–0.0275 m, Y 1.129–1.343 m, Z −0.0740–0.0760 m, refinement 4/9 |
| Inlet velocity | Vx=0, Vy=−240 m/s, Vz=0 | same |
| Pressure / Temperature | 101325 Pa / 293.2 K | same |
| GG Force (Y) 1 convergence (as found) | `[auto]` | `[auto]` |
| Fluid | Air (Gases), Laminar and Turbulent, High Mach off | same |

Domain, mesh level, and inlet were identical between the two projects apart from the local
mesh refinement around the camera pods (expected, since the pods require local refinement).
No stop condition was triggered.

Per user instruction, the GG Force (Y) 1 convergence criterion was changed from `[auto]` to
a manual absolute value of **0.5 N** (Use in convergence = Yes) in both projects.

## Step 3 — Domain enlargement

Original domain was found tighter than the required minimum clearances (0.5 m upstream of
nose, 5.0 m downstream of tail, 0.5 m lateral, for a 2.208 m long, 0.108 m diameter rocket).
Per user confirmation, domain was enlarged identically in both projects to:
- X: −0.6 to 0.6 m
- Y: −5.1 to 2.9 m
- Z: unchanged (−0.743365462 to 0.624013432 m)

## Step 5 — Solve results

| | Rocket Only | Camera Mount Test |
|---|---|---|
| GG Force (Y) 1 — current | −152.735 N | −153.714 N |
| GG Force (Y) 1 — **averaged** | **−152.924 N** | **−153.821 N** |
| Min / Max | −153.203 / −152.709 N | −154.109 / −153.614 N |
| Delta / Criteria | 0.494 / 0.500 | 0.495 / 0.500 |
| Equation Goal 1 (Cd) — averaged | 0.4764856 | 0.4792781 |
| Iterations | 148 | 158 |
| Solver status | Solver is finished, no warnings | Solver is finished, no warnings |
| Convergence | Current vs. averaged differ by 0.189 N (<1.0 N) — converged | Current vs. averaged differ by 0.107 N (<1.0 N) — converged |

Mesh refinement was not required — both runs converged cleanly on the first solve at the
existing global mesh level (Level 4).

## Step 6 — Computed drag coefficients

Using **rho = 1.225 kg/m³, V = 240 m/s, A_ref = 0.009097 m²** (rocket cross-section only;
q·A_ref = 320.9 N):

```
Cd_clean  = 2*abs(F_rocket_only) / (rho*V^2*A_ref) = F_rocket_only / (q*A_ref)
          = 152.924 / 320.9 = 0.4765

Cd_camera = 153.821 / 320.9 = 0.4794

dCd       = Cd_camera - Cd_clean = 0.0029
```

Inlet velocity: 240 m/s. At T = 293.2 K, speed of sound a = sqrt(1.4·287·293.2) ≈ 343.2 m/s,
giving **Mach ≈ 0.70** (high subsonic / low transonic).

### Sanity check (flagged, not silently passed)

| Check | Result | Bound | Status |
|---|---|---|---|
| Cd_clean in 0.45–0.60 | 0.4765 | pass | ✅ |
| Cd_camera > Cd_clean | 0.4794 > 0.4765 | pass | ✅ |
| dCd in 0.005–0.040 | 0.0029 | **below minimum** | ⚠️ FLAGGED |
| \|F_rocket_only\| in 165–185 N | 152.9 N | **below range** | ⚠️ FLAGGED |

Two of the four sanity bounds specified for this analysis are violated: the clean-rocket
force magnitude and the drag increment dCd both came in below the expected ranges. Both
Cd values individually are physically reasonable and self-consistent (Cd_clean matches the
live SolidWorks equation-goal output exactly, and Cd_camera > Cd_clean as expected), and both
runs converged cleanly with no solver warnings on an unchanged mesh/domain between the two
configurations. This suggests the expected-range priors (165–185 N, dCd 0.005–0.040) may not
match this specific geometry/reference-area/velocity combination, or that the true camera-pod
drag increment at Mach 0.70 is genuinely smaller than anticipated — rather than a
computational error. A "High Mach number flow" compressibility-correction re-run was
attempted at the user's request but was reverted: SolidWorks Flow Simulation's own solver
flagged it as "not recommended" at this Mach number (that option targets hypersonic-range
flows, not M≈0.7), so the standard solver settings (already compressible-capable in this
Mach range) were retained as the correct configuration. This flag is being surfaced per
instruction rather than silently reported; the numbers above should be treated with this
caveat until the user decides whether to investigate further (e.g. re-check A_ref/velocity
assumptions, or accept the result as-is).

## Equation goal expression used (both projects)

```
2*abs({GG Force (Y) 1})/(1.225*240^2*0.009097)
```
Dimensionality: No unit (dimensionless). "Use for convergence control" enabled.
