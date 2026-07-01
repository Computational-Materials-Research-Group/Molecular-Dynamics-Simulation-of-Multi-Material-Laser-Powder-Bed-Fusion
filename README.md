# Molecular Dynamics Simulation of Multi-Material Laser Powder Bed Fusion (Al/Cu Powder on Al Substrate) Using EAM Potential

<p align="center">
  <img src="https://img.shields.io/badge/LAMMPS-MD%20Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/LPBF-Multi--Material%20Al%2FCu-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/EAM-Al%2FCu%20Alloy%20Potential-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Powder%20Bed-Base%20%2B%20Stacked%20Spheres-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Laser%20Scan-Moving%20Heat%20Source-yellow?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/OVITO-Visualization-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Ubuntu-WSL-red?style=for-the-badge&logo=ubuntu&logoColor=white"/>
  <img src="https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A fully atomistic <b>molecular dynamics simulation of multi-material Laser Powder Bed
  Fusion (LPBF)</b> using LAMMPS. A heterogeneous Al/Cu powder bed — built from a base
  layer of packed spheres plus additional stacked spheres forming a multi-layer bed, with
  deliberately forced Al–Cu contact junctions — sits atop a pure Al substrate. A moving
  cylindrical laser heat source scans across the bed, capturing substrate equilibration,
  laser-induced heating, and dissimilar-material thermal response, all resolved at the
  angstrom scale.
  The simulation uses the <i>Al/Cu EAM/alloy potential</i>, a layered Langevin/NVE
  thermostat scheme for substrate heat dissipation, per-atom von Mises stress and
  hydrostatic pressure fields, and multi-phase trajectory output for OVITO visualization.
</p>

<img width="1600" height="1200" alt="hetero lpbf" src="https://github.com/user-attachments/assets/e89cab60-0377-4912-be1f-36ec334afd3e" />


---

## Why This is a Multi-Material LPBF Simulation

This simulation captures dissimilar-material powder bed fusion in three physically distinct respects:

- **Heterogeneous powder composition** — the powder bed is built from randomly placed
  spheres independently assigned to Al or Cu, rather than a single homogeneous material,
  reproducing the mixed-powder feedstock used in multi-material LPBF processes.

- **Forced dissimilar-material contact** — spheres are not only base-packed but also
  stacked on top of one another, with each stacked sphere given a high probability of
  being assigned the *opposite* material to the sphere it touches. This guarantees
  Al–Cu contact junctions exist in the bed rather than relying on chance placement,
  enabling direct study of dissimilar-material interface behavior under laser heating.

- **Layered thermal management** — a Langevin-thermostatted layer beneath a pure-NVE
  conduction layer models realistic heat dissipation from the laser-heated powder down
  into the bulk substrate, with a rigid frozen anchor layer preventing artificial
  substrate drift ("blowing") during the run.

This contrasts with single-material LPBF MD simulations where the powder bed is a
uniform composition and sphere placement is a simple non-overlapping grid.

---

## Simulation Overview

| Property | Value |
|----------|-------|
| Substrate | Aluminum — FCC, a₀ = 4.05 Å |
| Powder | Aluminum (type 1) + Copper (type 2) — randomly assigned per sphere |
| Potential | Al/Cu EAM/alloy (`CuAlW.txt`, Al and Cu elements referenced) |
| Substrate footprint | 62 × 111 unit cells (≈251.1 × 449.55 Å in X/Y) |
| Box height | Substrate + stacked-powder headroom (`stack_layers` multiplier) |
| Boundary conditions | Periodic in X/Y; shrink-wrap in Z |
| Substrate atoms | ~234,000 |
| Powder atoms (post-cleanup) | Variable — coincident atoms at Al/Cu contact points removed via `delete_atoms overlap` |
| Timestep | 1 fs (0.001 ps) |
| Substrate ensemble | Frozen anchor (bottom) + Langevin thermostat (mid layer) + NVE (conduction + powder) |
| Impact/scan mechanism | Moving cylindrical laser zone, `fix heat`, swept across the bed at constant scan speed |
| Total simulation length | Equilibration (`equil_steps`) + full-length laser scan (no cooling stage in this version) |

---

## System Geometry

```
z = box_zhi ┌─────────────────────────────┐  ← box top (shrink-wrap)
             │                             │
             │        ●●●   ●●●           │  ← stacked powder spheres
             │      ●●●●●●●●●●●●●         │     (Al/Cu, forced dissimilar
             │        ●●●   ●●●           │      contact at junctions)
             │   ●●●●●●●●●●●●●●●●●●●●     │
z ≈ pow_z0   │   ●●●●●●●●●●●●●●●●●●●●     │  ← base powder layer (Al/Cu)
             ├═════════════════════════════┤  ← substrate top
             │                             │
z_conduct_hi │   conduction layer (NVE)    │  ← heat sink toward thermostat
             │                             │
z_therm_hi   │   thermostat layer          │  ← Langevin, T_bed
             │   (Langevin, 10*dt damping) │
             │                             │
z_fix_hi     │   frozen anchor layer       │  ← setforce 0 0 0, rigid base
             │                             │
z = 0        └─────────────────────────────┘  ← box bottom (shrink-wrap)
```

| Component | Atom type(s) | Region | Z range | Ensemble |
|-----------|-------------|--------|---------|----------|
| Base + stacked powder spheres | 1 (Al) / 2 (Cu), random | Sphere unions, generated externally | Above substrate top | NVE (`g_nve_only`) |
| Conduction layer | 1 (Al) | Block | `z_therm_hi` to `z_conduct_hi` | NVE (`g_nve_only`) |
| Thermostat layer | 1 (Al) | Block | `z_fix_hi` to `z_therm_hi` | Langevin + NVE (`g_therm`) |
| Frozen anchor | 1 (Al) | Block | `0` to `z_fix_hi` | Fixed (`setforce 0 0 0`) |

---

## Simulation Phases

```
Three-Stage Minimization (substrate → powder → full system)
      ↓
Pre-Relaxation  (fire minimizer on powder group)
          Resolves near-touching/slightly-fused spheres from stacking
      ↓
Velocity Initialization
          Mobile atoms seeded at T_bed; fixed atoms explicitly zeroed
      ↓
NVE/Limit Warm-Up  (200 steps, displacement-capped)
          Prevents initial-kick instability; thermostat rescaled to T_bed after
      ↓
Stage 1 — Equilibration  (equil_steps)
          Bulk system equilibrates at T_bed; thermostat/powder temps monitored
      ↓
Stage 2 — Laser Scan  (scan_steps, dense output every step)
          Moving cylindrical heat source sweeps across the full powder bed
      ↓
Final Frame + Output
          dump_ALL.lammpstrj, write_data, write_restart
          (No cooling/solidification stage in this version — see Extending section)
```

---

## Atom Types and Color Coding

| Type | Element | OVITO Color (typical) | Role |
|------|---------|------------------------|------|
| 1 | Al | 🔴 Red | Substrate (frozen + thermostat + conduction) + Al powder spheres |
| 2 | Cu | 🔵 Blue | Powder spheres only (base layer + stacked) |

Use **Color Coding → by Particle Type** in OVITO to distinguish Al (red) from Cu (blue)
directly — no expression filter needed, since the two materials are genuinely different
atom types (unlike a homogeneous Al–Al simulation).

---

## Repository Structure

```
LPBF_MultiMaterial_AlCu/
│
├── in.het                        # Main LAMMPS input script
├── generate_powder_spheres.py    # Powder bed geometry generator (run first)
├── powder_spheres.lmp            # Auto-generated sphere regions (output of the above)
├── CuAlW.txt                     # EAM/alloy potential file (required; Al, Cu elements used)
├── README.md                     # This file
│
└── output/                       # Generated on run
    ├── dump_00_check.lammpstrj        # Post-minimization geometry check frame
    ├── dump_ALL.lammpstrj              # Full trajectory: equilibration + laser scan + final frame
    ├── al_cu_pbf_final.data            # Final LAMMPS data file
    └── al_cu_pbf_final.restart         # Binary restart file
```

---

## Requirements

- LAMMPS (7 Feb 2024 or newer, with `MANYBODY`/EAM support): https://www.lammps.org
- EAM potential file `CuAlW.txt` — place in the same directory as the input script
- Python 3 (standard library only — no external dependencies) for sphere generation
- OVITO for visualization: https://www.ovito.org

---

## Installation

```bash
# Ubuntu / WSL
sudo apt update && sudo apt install -y lammps
```

---

## Running the Simulation

```bash
# 1. Generate the multi-material powder bed geometry
python3 generate_powder_spheres.py

# 2. Verify the geometry file was (re)generated
grep -c "^region  s" powder_spheres.lmp

# 3. Run the simulation — single core
lmp -in in.het

# Parallel run (recommended for large beds)
mpirun -np 4 lmp -in in.het
```

**Important:** step 1 must be re-run any time a geometry parameter changes in the Python
script. LAMMPS does not regenerate `powder_spheres.lmp` on its own.

---

## Simulation Parameters

### Group and Fix Assignment

| Group | Fix | Temperature | Notes |
|-------|-----|-------------|-------|
| `g_fixed` (frozen anchor) | `setforce 0.0 0.0 0.0` | — | Rigid base; 3 unit cells (6 monolayers) |
| `g_therm` (thermostat layer) | `langevin` + `nve` | `T_bed` | Damping = `10*dt`; 2 unit cells (4 monolayers) |
| `g_nve_only` (conduction + powder) | `nve` | None (adiabatic) | Heat flows through, no direct thermostat |
| `g_mobile` | — | — | All atoms except `g_fixed`; used for velocity initialization only |

Per-material groups `g_Al` (type 1) and `g_Cu` (type 2) are defined for diagnostic
temperature tracking (`c_T_Al`, `c_T_Cu`) independent of the z-based layer groups.

### Powder Bed Generation Parameters

| Parameter | Value / Meaning |
|-----------|------------------|
| `R_pow` | Sphere radius, 24.9 Å |
| `n_base` / `n_stack` | Number of base-layer / stacked spheres |
| `base_min_gap` | Allowed overlap tolerance in the base layer (enables natural touching) |
| `stack_overlap` | Depth a stacked sphere sinks into its support sphere |
| `global_min_gap` | Hard overlap floor — never violated, regardless of stacking |
| `MATERIALS` | `{1: "Al", 2: "Cu"}` — atom-type to element mapping |

### Laser Scan Parameters

| Parameter | Value |
|-----------|-------|
| Laser power | `laser_power` (heat flux via `fix heat`) |
| Laser radius | `laser_radius` (cylindrical zone radius, Å) |
| Scan speed | `scan_speed` (Å/ps, moves along Y) |
| Scan zone | Dynamic group, recomputed every step (`group ... dynamic ... every 1`) |
| Scan length | Full box Y-length (`ly`) |

### Overlap Cleanup

| Parameter | Value |
|-----------|-------|
| `delete_atoms overlap` cutoff | 2.0 Å |
| Purpose | Removes coincident Al/Cu atoms placed by independent `create_atoms` calls at deliberate sphere-overlap (contact) locations |

### Neighbor List

| Parameter | Value |
|-----------|-------|
| Skin distance | 3.0 Å |
| Update | every step, check yes |

---

## Key Script Features

| Feature | Implementation |
|---------|---------------|
| Multi-material powder bed | Al/Cu spheres generated by an external Python rejection-sampling script, included via `powder_spheres.lmp` |
| Forced dissimilar contact | Stacked spheres preferentially assigned the opposite material of their support sphere (default 90% probability) |
| Coincident-atom cleanup | `delete_atoms overlap 2.0 all all`, run only after `pair_style`/`mass` are defined (required for its internal neighbor list) |
| Layered heat dissipation | Frozen anchor → Langevin thermostat → pure-NVE conduction/powder, avoiding substrate "blowing" |
| Pre-relaxation | `fire` minimizer pass on the powder group before the main 3-stage minimization, to resolve stacking-induced contact stress |
| Staged minimization tolerances | Stage A/B loosened to `1e-3` (handles larger initial contact forces); Stage C kept tight at `1e-6`/`1e-8` |
| Per-material diagnostics | `c_T_Al`, `c_T_Cu`, `c_T_therm`, `c_T_pow` all tracked separately in `thermo_style` |
| Dense laser-scan output | `freq_laser = 1` — every-step trajectory capture during the scan phase |
| No cooling stage | Simulation ends immediately after the laser scan; final frame is hot by design (see Extending section) |

---

## Visualization in OVITO

### Access trajectory from Windows (WSL users)

```
\\wsl$\Ubuntu\home\<username>\output\dump_ALL.lammpstrj
```

### Recommended modifiers

1. **Color Coding → by Particle Type** — directly separates Al (type 1) from Cu (type 2); no expression filter needed
2. **Color Coding → Color by `v_S_vm`** — per-atom von Mises stress; highlights contact necks between Al and Cu spheres
3. **Color Coding → Color by `v_S_hyd`** — hydrostatic stress; reveals compressive/tensile zones at dissimilar-material junctions
4. **Common Neighbor Analysis (CNA)** — FCC (green) vs. disordered/surface (white/red) atoms distinguish powder-sphere surfaces from bulk
5. **Slice through a stacked-sphere junction** — isolate a single Al–Cu contact point to inspect interface structure directly
6. **Color Coding → by `v_T_atom`** — per-atom temperature field; tracks the laser heat front as it sweeps across the bed

---

## What to Expect

### After Minimization + Pre-Relaxation
Contact necks between touching spheres (same- or dissimilar-material) relax from their
initial construction geometry. `E_pair` should be finite (never `inf`/`-nan`) once
`delete_atoms overlap` has removed coincident atoms from sphere-overlap regions.

### During Equilibration
The bulk system settles to `T_bed`. `c_T_therm` and `c_T_pow` should both track close to
`T_bed`; large sustained deviation indicates insufficient thermostat coupling or an
undersized thermostat/conduction layer.

### During the Laser Scan
As the cylindrical heat zone sweeps across the bed, `c_T_pow` rises sharply in atoms
beneath the beam. Because Cu (melting point ≈1358 K) and Al (≈933 K) respond differently
to the same laser power, `c_T_Al` and `c_T_Cu` should be compared directly — a laser
power tuned for Al alone may not fully melt Cu regions, which is itself a useful
multi-material diagnostic rather than necessarily an error.

### At Dissimilar-Material Contact Points
Von Mises and hydrostatic stress fields should show distinct signatures at Al–Cu contact
necks compared to Al–Al or Cu–Cu contacts, reflecting the different elastic/thermal
properties captured by the EAM potential.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `E_pair = inf` at first minimization step | Coincident Al/Cu atoms at deliberate sphere-overlap contact points | Run `delete_atoms overlap 2.0 all all` after powder creation |
| `Not all per-type masses are set` on `delete_atoms` | `pair_style`/`mass` defined after `delete_atoms overlap` | Move the potential/mass section before `delete_atoms overlap` (it builds a neighbor list) |
| `Non-numeric atom coords - simulation unstable` | Same root cause as above, surfaces later in `cg` minimization | Same fix; verify atom count drops between "before cleanup" and "after cleanup" prints |
| Only a handful of spheres visible in OVITO | Stale `powder_spheres.lmp` from a previous script version | Re-run `python3 generate_powder_spheres.py` before every LAMMPS run |
| Stacked spheres clipped at box top | `box_zhi` sized for a single flat powder layer only | Add `stack_layers` headroom multiplier to the `box_zhi` formula |
| Substrate atoms "blowing" apart | Langevin damping too slow / too few frozen or thermostat layers | Damping `= 10*dt`; 3 unit cells frozen; 2 unit cells thermostatted; velocity explicitly zeroed on `g_fixed` |
| `WARNING: only placed X/n spheres` from Python script | Box footprint too small for requested sphere count/radius/gap | Reduce `n_base`/`n_stack`, reduce `R_pow`, or enlarge `Nx_sub`/`Ny_sub` in both scripts |
| Element order mismatch in `pair_coeff` | Symbol order doesn't match `CuAlW.txt` header | Check the setfl header line in `CuAlW.txt`; reorder `pair_coeff * * CuAlW.txt <order>` to match exactly |

---

## Extending the Simulation

| Extension | What to Change |
|-----------|---------------|
| Add a cooling/solidification stage | Re-introduce an `nvt` ramp-down stage after the laser scan (unfix laser-scan fixes, add `fix ... nvt temp <current> T_bed <damp>`) |
| Third material (e.g. W) | Add `MATERIALS = {1: "Al", 2: "Cu", 3: "W"}` in the Python script; update `create_box 3`, `mass`, and `pair_coeff * * CuAlW.txt Al Cu W` |
| Denser/sparser bed | Adjust `n_base` / `n_stack` in the Python generator |
| Larger scan area | Increase `Nx_sub` / `Ny_sub` in both the Python script and LAMMPS input (must match) |
| Faster/slower laser scan | Adjust `scan_speed`; `scan_steps` recomputes automatically from `ly / scan_speed` |
| Higher/lower laser power | Adjust `laser_power` in `fix f_laser ... heat 1 ${laser_power}` |
| Per-sphere radius variation | Randomize `R_pow` per sphere in the Python generator instead of using a single fixed value |
| Quantify interface bonding | `compute rdf` restricted to atoms near stacked-sphere contact points |
| Track melt pool geometry | Combine CNA with `v_T_atom` thresholding in OVITO to outline the melt-pool boundary over time |

---

## Citation

If you use this simulation in your research, please cite:

```bibtex
@software{mishra_2026_multimaterial_lpbf,
  author    = {Mishra, Akshansh},
  title     = {Molecular Dynamics Simulation of Multi-Material Laser Powder Bed Fusion (Al/Cu Powder on Al Substrate) Using EAM Potential},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.21106899},
  url       = {https://doi.org/10.5281/zenodo.21106899}
}
```

Plain text citation:

> Mishra, A. (2026). *Molecular Dynamics Simulation of Multi-Material Laser Powder Bed Fusion (Al/Cu Powder on Al Substrate) Using EAM Potential*. Zenodo. https://doi.org/10.5281/zenodo.21106899

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
</p>

You are free to:

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material for any purpose, including commercially

Under the following terms:

- **Attribution** — You must give appropriate credit to the original author(s) and provide a link to the source repository
