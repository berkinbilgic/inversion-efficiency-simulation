# inversion-efficiency-simulation

Bloch simulation of the inversion efficiency (IE) of an adiabatic inversion pulse over a grid of T1 and T2 values. The resulting table maps (T1, T2) to IE and is used as a lookup when simulating 3D-QALAS signal evolution, to account for relaxation during the 12.8 ms inversion pulse.

## inversion_efficiency_lookup.m

1. **Pulse**: reads the HS4 adiabatic inversion pulse (`IR180_HS4`, from MP2RAGE; 12.8 ms, 1280 samples at 10 µs, magnitude and phase) from `data/mp2rage_invpulse.txt`. The complex waveform is scaled to a nominal flip of 1500° (`sc_pulse = 1500/180`), defined from its integral.
2. **Simulation**: for each (T1, T2) pair, starts from M = [0 0 1] and integrates the Bloch equations with T1 and T2 relaxation through the pulse using `ode45`. Single isochromat, on resonance (B0 = 0 Hz), nominal B1 (scale 1).
3. **IE** = −Mz at the end of the pulse. 1 is perfect inversion; values below 1 come from relaxation during the pulse.
4. Saves the table to `data/ielookup_toolbox_v3.mat`.

Grid (620 × 730 pairs):

- T1 = 5:5:3000 ms, 3100:100:5000 ms (620 values)
- T2 = 1:1:700 ms, 710:20:1000 ms, 1100:100:2500 ms (730 values)

Output struct `ielookup`:

| Field | Size | Content |
|---|---|---|
| `ies_mtx` | 730 × 620 | IE; rows follow T2, columns follow T1 |
| `T1` | 1 × 620 | T1 [s] |
| `T2` | 730 × 1 | T2 [s] |

IE at arbitrary T1, T2 [s]:

```matlab
load('data/ielookup_toolbox_v3.mat');
ie = interp2(ielookup.T1, ielookup.T2, ielookup.ies_mtx, t1, t2);
```

Each (T1, T2) pair takes about 0.05 s, so the full grid takes roughly 6 hours.

## Data

| File | Content |
|---|---|
| `data/mp2rage_invpulse.txt` | HS4 adiabatic inversion pulse `IR180_HS4` (12.8 ms, beta = 0.04, FS = 830 Hz); magnitude and phase [rad] |
| `data/ielookup_toolbox_v3.mat` | IE table on the grid above |
| `data/ielookup_4qalas.mat` | IE table on an earlier, coarser grid (227 × 339): T1 = 5:10:3000, 3100:50:5000 ms; T2 = 1:2:350, 370:20:1000, 1100:100:3000 ms. Same pulse and settings; these are the commented "ori qalas blochsim" ranges in the script |
| `data/HyperbolicSecantRF_T2prep.txt` | hyperbolic secant refocusing pulse for the T2-prep helpers; magnitude and phase [rad] |

Re-simulating sample grid points with the script reproduces the stored values in both tables exactly.

## simulation_functions/

| File | Content |
|---|---|
| `preparing_inversion.m` | scales the pulse and builds the inversion event block |
| `simulate_event_blocks.m` | runs a list of event blocks: RF blocks with `pulse_ode45`, free relaxation with `recovery` |
| `pulse_ode45.m` | Bloch equations during RF, integrated with `ode45` |
| `recovery.m` | analytic T1/T2 relaxation |
| `preparing_adiabatic_pulse_t2prep.m` | T2-prep event blocks: hard 90°, four adiabatic refocusing pulses, hard 90° with 180° phase; timings are hard-coded |
| `t2prep.m` | same as `simulate_event_blocks.m` under another name |
| `plot_magnetization.m`, `rmse.m` | small helpers |

The T2-prep functions, `plot_magnetization.m` and `rmse.m` are not used by `inversion_efficiency_lookup.m`.

## Notes

- Run the script from the repository root (it uses relative paths).
- Core MATLAB only (`ode45`, `interp1`, `interp2`, `readtable`). The script opens a `waitbar`.
- With `save_flag = 1` the script overwrites `data/ielookup_toolbox_v3.mat`. Set `save_flag = 0`, or change the file name, to keep the distributed table.
- `ielookup_toolbox_v3.mat` is loaded by the QALAS dictionary fit in [xd-qalas](https://github.com/berkinbilgic/xd-qalas) (`fn_qalas_blochsim_map_v4.m`), which reshapes `ies_mtx` into its own T1/T2 dictionary order. If you change the grid here, change it there too.
