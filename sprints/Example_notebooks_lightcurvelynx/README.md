# Example notebooks: simulating SNe Ia with `lightcurvelynx`

This folder contains two example notebooks showing how to simulate **Type Ia supernova (SN Ia) light curves** with [`lightcurvelynx`](https://lightcurvelynx.readthedocs.io/en/latest/) on LSST-like cadences, and how to check the simulations with a **Bayesian fit of the SALT3 parameters** (`emcee`).

| Notebook | Survey / cadence | Purpose |
|---|---|---|
| `01_SimulateSNLightCurvelynx.ipynb` | DP2 + pre-LSST visits (wide survey, 2025/04 - 2026/07) | Minimal end-to-end example: cadence -> SN Ia population -> light curves -> posteriors -> diagnostics |
| `02_SimulateSNLightCurvelynx_DDF.ipynb` | The six LSST Deep Drilling Fields (per-CCD visit table) | Realistic DDF simulation: expected rates, dust, nightly co-addition, search efficiency, detection and SALT quality cuts, fits, counts per field |

Both notebooks follow the same implementation pattern:

```
observation table  ->  SN Ia population graph  ->  simulate_lightcurves  ->  emcee posteriors  ->  validation (pulls, ...)
```

Notebook 02 is a rewrite with `lightcurvelynx` of the `skysurvey`-based notebook `Simulate_DDF.ipynb` by Madeleine Ginolin (`../Example_notebooks_madeleine2026/`).

- Author: Sylvie Dagoret-Campagne
- Created: 2026-10-29 (notebook 01), 2026-10-30 (notebook 02)

---

## 01 - `01_SimulateSNLightCurvelynx.ipynb`

Simulates 200 SNe Ia over the DP2 + pre-LSST cadence and checks that the SALT3 parameters can be recovered.

**Steps**

1. **Cadence**: reads the visits database `dp2/prelsst_20260607_visits.db` from `$RUBIN_SIM_DATA_DIR`, lists its tables and columns (the schema is not assumed).
2. **Slim export**: writes a reduced copy of the visits table (`visits_slim.db`, only the needed columns) for offline checks.
3. **ObsTable loading**: the loader is selected from the column names among three schemas (`ccdvisit`, `sv_visits`, `opsim`). The notebook documents the differences between them and why only a subset of the ~270 columns is kept. Here the `sv_visits` schema is used (`LSSTObsTable.from_sv_visits_table`), after cleaning (science exposures only, finite values, physical zero point / sky / seeing / exposure time).
4. **Cadence checks**: time distribution of the visits and sky footprint (MOC).
5. **SN Ia population** (SALT3 + Tripp, flat LCDM):
   - redshift uniform in [0.02, 0.30], positions drawn from the survey footprint, `t0` uniform over the cadence;
   - `x1 ~ N(0, 1)`, `c ~ N(0, 0.1)`, `M_abs ~ N(-19.3, 0.1)`, `alpha = 0.14`, `beta = 3.1`, `H0 = 70`, `Omega_m = 0.3`;
   - rest-frame phase window [-20, +50] days (domain of SALT3).
6. **Simulation**: `simulate_lightcurves` with the survey noise model (Poisson noise from the measured zero point, sky background and seeing). SNe outside the footprint or the time range are dropped.
7. **Selection and plots**: keeps well-sampled SNe (>= 10 detections at S/N > 5 in >= 2 bands) and plots a few light curves with the noise-free model.
8. **Bayesian inference**: for each selected SN (10 by default), `emcee` samples the posterior of `(t0, mB, x1, c)` at the known redshift (16 walkers, 700 steps, 250 burn-in). The model is the same as in the simulation, so this is a **closed-loop test**: pulls should be ~ N(0, 1). Results are saved after each SN so the loop can be restarted.
9. **Diagnostics**: pulls and empirical coverage of the 68% / 95% intervals, chain convergence (acceptance, autocorrelation time), Hubble diagram and residuals, and a corner plot of one posterior.

**Outputs**

- `data_01_SimSNIa/01_SimSNIa_posteriors.pkl` (posterior chains), `data_01_SimSNIa/passbands/` (cached LSST passbands)
- `figs_01_SimSNIa/` (PDF + PNG): `cadence_time`, `footprint`, `pulls`, `hubble`, `corner_sn0`

---

## 02 - `02_SimulateSNLightCurvelynx_DDF.ipynb`

Simulates SNe Ia in the six LSST DDFs (COSMOS, ECDFS, EDFS_a, EDFS_b, ELAISS1, XMM_LSS), using the same inputs as the `skysurvey` notebook but with `lightcurvelynx` for all the simulation machinery. All fluxes are in nJy.

**Steps**

1. **Inputs**: per-CCD visit table `edp2_detectorvisit_table.parquet` (one row per visit and detector), SNANA search-efficiency file, and Rubin bandpasses (`filter_lsst_<band>.npy`), all read from `../Example_notebooks_madeleine2026/`. The noise scheme is detected from the columns:
   - `skysurvey` scheme (`zp`, `skynoise`, `gain`): converted to the `lightcurvelynx` Poisson model (zero point in nJy per electron, sky variance in e-^2, no read noise or dark current), so that the original flux errors are reproduced exactly;
   - DP2 `VisitDetector` scheme (`zeroPoint`, `seeing`, `skyBg`): standard `from_ccdvisit_table` loader.
2. **Bandpasses**: converts the Rubin transmission curves to `lightcurvelynx` passbands (falls back to the LSST baseline throughputs if the files are missing).
3. **Milky Way dust**: E(B-V) from the SFD maps (`dustmaps`, directory given by `$DUST_DATA_DIR`), applied with the F99 law in the observer frame. Falls back to a constant E(B-V) = 0.02 if the maps are not found.
4. **Population and expected counts**: Frohmaier+2019 volumetric rate `r(z) = r0 (1+z)^alpha`, SNANA asymmetric-Gaussian `x1` and `c`, Tripp relation (`alpha = 0.15`, `beta = 3.15`), cosmology `H0 = 70`, `Omega_m = 0.315`, 0.05 < z < 1.2. For each field the expected number of SNe is computed from the bounding box of the pointings, and the SNe are drawn with a Poisson fluctuation around it.
5. **Simulation, field by field**: one `LSSTObsTable` per DDF (with the CCD footprint), SALT3 source with F99 extinction, `simulate_lightcurves`.
6. **Processing**: nightly inverse-variance co-addition of the points, then application of the SNANA search efficiency band by band (random thinning according to S/N).
7. **Detection cuts**: S/N >= 5 detections within phases [-10, +40] days, in at least 3 bands.
8. **Bayesian fits**: `emcee` posteriors of `(t0, mB, x1, c)` (and optionally `z` in the `free_z` mode) from the co-added points, using the same SALT3 + F99 model as the simulation (E(B-V) fixed to its true value). Flat priors by default (`PRIOR_MODE = "population"` is also available). Limited to `N_FIT = 20` detected SNe by default.
9. **SALT quality cuts**: `-0.3 < c < 0.3`, `sigma_c < 0.2`, `-3 < x1 < 3`, `sigma_x1 < 1`, `sigma_t0 < 2` d, fit probability > 1e-4, E(B-V) < 0.3.
10. **Counting and validation**: number of SNe per field at each stage (expected, drawn, observed, detected, fitted, selected), pull distributions (fit - truth) / error, and an example light curve showing data, true model and maximum-posterior fit.

**Main configuration switches** (cell "Imports and configuration"): `SIM_FRACTION` (fraction of the expected SNe to simulate, reduce for quick tests), `FIT_MODES` (`"fixed_z"`, add `"free_z"`), `N_FIT`, `PRIOR_MODE`, `USE_MW_DUST`.

**Outputs**

- `data_02_SimSNIa_DDF/`: `02_SimSNIa_DDF_sim_params.parquet` (true parameters of the simulated SNe), `02_SimSNIa_DDF_counts_per_field.csv`, `02_SimSNIa_DDF_posteriors.pkl`
- `figs_02_SimSNIa_DDF/` (PDF + PNG): `bandpasses`, `counts_per_field`, `pulls_fixed_z`, `example_lc_<SN>`

---

## Other files in this folder

| File | Description |
|---|---|
| `prelsst_20260607_visits.db` | DP2 + pre-LSST visits database (SQLite, ~313 MB) used as cadence by notebook 01 |
| `visits_slim.db` | Slim version of the visits table (~20 MB), written by notebook 01 |
| `data_*/`, `figs_*/` | Outputs of each notebook (see above) |

## Requirements

- Kernel: `lightcurvelynx_py313` (Python 3.13)
- Python packages: `lightcurvelynx`, `numpy`, `pandas`, `scipy`, `astropy`, `matplotlib`, `emcee`, `corner`, `tqdm`, `pyarrow`, `sncosmo` (via `lightcurvelynx`)
- Notebook 02 additionally needs `dust_extinction` (`conda install conda-forge::dust_extinction`) and `dustmaps` with the SFD maps
- Environment variables:
  - `RUBIN_SIM_DATA_DIR`: directory containing `dp2/prelsst_20260607_visits.db` (notebook 01 and 02)
  - `DUST_DATA_DIR`: directory containing `sfd/SFD_dust_4096_{ngp,sgp}.fits` (notebook 02)
- Notebook 02 reads its input files from `../Example_notebooks_madeleine2026/` (visit table, search efficiency, filter curves)

## Usage

Run the notebooks in order from this directory (the output paths are relative to the working directory):

```bash
cd sprints/Example_notebooks_lightcurvelynx
jupyter lab   # select the kernel lightcurvelynx_py313
```

The `emcee` fits are the slow part (~0.5-1 min per SN). Results are cached in the `*_posteriors.pkl` files, and already-fitted SNe are skipped when a notebook is re-run.

## Documentation

- `lightcurvelynx`: https://lightcurvelynx.readthedocs.io/en/latest/
- SN Ia example: https://lightcurvelynx.readthedocs.io/en/latest/notebooks/simple_snia.html
- OpSim example: https://lightcurvelynx.readthedocs.io/en/latest/notebooks/opsim_notebook.html
