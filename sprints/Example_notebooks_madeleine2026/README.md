# README: SN Ia simulation notebooks from Madeleine Ginolin (SN Sprint, 2026)

- Author of this README / adaptations: Sylvie Dagoret-Campagne
- Creation date: 2026-10-26
- Goal: work with Madeleine Ginolin on the SN Sprint

This folder contains four notebooks written with [`skysurvey`](https://github.com/MickaelRigault/skysurvey) and `sncosmo`. They simulate **Type Ia supernovae (SNe Ia) in the six LSST Deep Drilling Fields (DDFs)** using the real per-CCD observing log of the DP2/EDP2 visits, fit the simulated light curves with SALT3, and compare the simulations with the real DDF SN Ia candidates.

Two of the notebooks are the **original versions** received from Madeleine Ginolin (unchanged). The other two are the **versions adapted by SDC**, which are the ones to run.

## Original vs adapted notebooks

| Notebook | Status | What it does |
|---|---|---|
| `Simulate_DDF_original.ipynb` | **Original** (Madeleine Ginolin) | Simulation of SNe Ia in the 6 DDFs + SALT3 fits |
| `Simulate_DDF.ipynb` | **Adapted by SDC** from the original above | Same science, plus dust-map setup/download (laptop and NERSC), environment checks and explanatory comments |
| `Data_vs_sims_comparison_orignal.ipynb` | **Original** (Madeleine Ginolin; the file name is spelled `orignal`) | Simulates the real DDF candidates and compares data and simulated light curves |
| `Data_vs_sims_comparison.ipynb` | **Adapted by SDC** from the original above | Same comparison, made runnable stand-alone (own inputs for the SALT fits, local output path, more robust fit) |

Both adapted notebooks are headed "from Madeleine Ginolin, adapted by SDC", dated 2026-10-26, and use the kernel `conda_py313` on the laptop and `desc-td-env` (not `-dev`) at NERSC.

---

## `Simulate_DDF_original.ipynb` / `Simulate_DDF.ipynb`

### What the notebook does

It simulates SNe Ia in the six DDFs (COSMOS, ECDFS, EDFS_a, EDFS_b, ELAISS1, XMM_LSS) and tests how well their light curves can be recovered with SALT3.

1. **Population model**: SALT3 template, asymmetric-Gaussian distributions (SNANA parametrisation) of stretch `x1` and colour `c`, Tripp standardisation (`M = -19.3`, intrinsic scatter 0.1 mag, `alpha = -0.15`, `beta = 3.15` in the `skysurvey` sign convention), Frohmaier+2019 volumetric rate `r(z) = 2.3e4 (1+z)^1.70` Gpc^-3 yr^-1, flat LCDM with `H0 = 70`, `Omega_m = 0.315` (cosmology agreed at the DESC July 2026 meeting), and Milky Way extinction (F99 law, E(B-V) from the SFD maps).
2. **Survey**: per-CCD visit table `edp2_detectorvisit_table.parquet` (one row per visit and detector, adapted by Madeleine from the DP2 `VisitDetector` table so that it is compatible with `skysurvey`), a square CCD footprint (area 9.6 deg^2 / 189 CCDs), and the Rubin bandpasses from the butler (`filter_lsst_<band>.npy`) registered in `sncosmo` as `rubinu` ... `rubiny`.
3. **Simulation**: field by field, SNe are drawn with 0.05 < z < 1.2 and peak dates within the visits +/- 30 days, then "observed" over rest-frame phases [-20, +40] days (noisy fluxes, errors, zero points).
4. **Processing**: fluxes put on a common zero point (ZP = 30), nightly inverse-variance co-addition, SNANA search efficiency (`SEARCHEFF_PIPELINE_LSST_DC2_Run2.2i_WFD.DAT`, estimated from Sanchez+22) applied band by band with a binomial draw.
5. **Light-curve cuts**: 5-sigma detections in at least 3 bands (phase range [-10, +40] days) and counting of the SNe per DDF. XMM_LSS has no co-add templates, so it would have no SNe in the real sample.
6. **SALT3 fits** (`sncosmo.fit_lc`, with Milky Way dust): the section titled "SALT fits **without** z info" runs with `fit_z = True` (redshift free, starting from the true value), and the section "SALT fit **with** z info" runs with `fit_z = False` (redshift fixed to the true value). Start values: true `t0` + N(0, 10 d). Quality cuts: `-0.3 < c < 0.3` and `sigma_c < 0.2`, `-3 < x1 < 3` and `sigma_x1 < 1`, `sigma_t0 < 2` d, fit probability > 1e-4, E(B-V) < 0.3.
7. **Validation**: pull distributions (fit - truth) / error for `x1`, `c`, `t0` and `m_B`, and an example light curve with the true and the fitted SALT3 models.

The selection and counting cells after each of the two fits are identical and overwrite the same variables, so the final pull plots and the example light curve describe the **second** fit (redshift fixed to its true value).

### What was adapted by SDC (`Simulate_DDF.ipynb`)

The science is unchanged. The adaptation concerns the setup, portability and documentation:

- **Dust maps**: new cells to set up `dustmaps` and download the SFD maps (`SFD_dust_4096_ngp.fits` and `SFD_dust_4096_sgp.fits`, from the NERSC portal). The download streams to a temporary `.part` file and is skipped if the maps are already present. A check cell prints the shape and size of the FITS files, and an `SFDQuery` object is created to test that the maps can be read.
- **Machine detection**: the host name selects the data directory, `./ASTROMAPS` at NERSC and `~/DATA/ASTROMAPS` on the laptop, and registers it in the `dustmaps` configuration.
- **Environment check**: a cell prints the Python executable and version, and where `skysurvey`, `sncosmo`, `astropy`, `numpy` and `dustmaps` are imported from (to make sure the right kernel is used).
- **Bandpass registration**: `force=True` in `sncosmo.register`, so the cell can be re-run without a "already registered" error.
- **Final light-curve figure**: annotations placed in axes coordinates (they stay inside the frame; in the original they used hard-coded data coordinates), headroom above the data, legend moved to the upper right, and `tight_layout`.
- **Documentation**: an "Overview" section and detailed comments on almost every cell (added on 2026-10-29), plus notes in the SALT-fit section (fits with or without redshift, no redshift from the host galaxy).

---

## `Data_vs_sims_comparison_orignal.ipynb` / `Data_vs_sims_comparison.ipynb`

### What the notebook does

It checks how realistic the simulations are by simulating **the real DDF SN Ia candidates** and comparing their simulated light curves with the real ones.

1. **Real sample**: `final_sample_ddf.parquet` (the file of "Mi", to be updated once a better candidate list exists, since some candidates may be variable stars or AGNs). It contains for each candidate the SALT parameters (`z`, `x1`, `c`, `t0`, `x0`, `mwebv`), the position of the DIA object and its difference-image forced-photometry light curve (`diaObjectForcedSource`).
2. **Simulation of the same SNe**: the SALT parameters of each real candidate are used as inputs of `skysurvey` (instead of drawing a random population), with the same per-CCD visit table, footprint and Rubin bandpasses as in `Simulate_DDF`. Cosmology is Planck18, the simulated phase range is [-260, +260] days, and the simulated fluxes are converted to ZP = 31.4 (nJy) to match the data. Co-addition, search efficiency and light-curve cuts are **skipped** (the data are not co-added, the search efficiency is probabilistic, and the real sample already went through quality cuts).
3. **Comparison for one SN**: data and simulated light curves, band by band and side by side.
4. **Comparison on the full sample, per band**: pulls (F_sim - F_data) / sqrt(err_sim^2 + err_data^2) after point-to-point matching in time (tolerance 0.01 d), ratio of the simulated and real flux errors, and histograms of the flux errors (output figure `EDP2_fluxerr_hist_per_band.pdf`).
5. **Comparison per DDF** (ELAISS1, EDFS, ECDFS, COSMOS, selected with RA/Dec boxes): histograms of flux errors, flux-error ratios and pulls.
6. **SALT fits** of the simulated light curves, with the same fit function, cuts and pull plots as in `Simulate_DDF`.
7. **Example light curves**: the true SALT3 model with the simulated points, and the real light curve from the data.

### What was adapted by SDC (`Data_vs_sims_comparison.ipynb`)

- **Header**: author, date (2026-10-26), kernels, and the `scp` command used to copy `final_sample_ddf.parquet` to NERSC.
- **Inputs of the SALT fits**: in the original, the fit cells use `lcs_coadd`, `snias_new`, `snias_detected` and `targets_to_consider`, which are only defined in `Simulate_DDF`. A new section rebuilds them directly from the simulated light curves (one point per visit, no co-addition): removal of non-finite points and of points with a non-positive error, selection of the SNe detected at 5 sigma in at least 3 bands within [-10, +40] days, and restriction of the light curves to the phase range [-20, +50] days where SALT3 is defined. Far-from-peak points made the fit covariance singular.
- **Robust fits**: `fit_salt_integrated` also catches `np.linalg.LinAlgError`, and a failed fit gives NaN instead of stopping the loop.
- **Output path**: the figure `EDP2_fluxerr_hist_per_band.pdf` is saved in the working directory. The original wrote to an absolute path on Madeleine's laptop (`/Users/mginolin/...`), which fails on any other machine.
- **Leftover cell**: the cell that displayed the undefined `saltpars` is commented out.

The fit function of both `Data_vs_sims` notebooks uses the `salt3-nir` source (SALT3 extended to the near infrared), whereas the `Simulate_DDF` notebooks use `salt3`.

### Known leftovers (in both versions)

The last cells of both `Data_vs_sims` notebooks are scratch cells inherited from the original. As far as I can tell from the code, they refer to variables that are not defined (`sn_fit`, `data_lc_g`, `mask_mjd_data`, `sim_lc_g`, `mask_mjd_sim`), so they will fail if run as is. The "Missing plots here" cell is also a placeholder from the original.

---

## Other files in this folder

| File | Description |
|---|---|
| `edp2_detectorvisit_table.parquet` (~237 MB) | Per-CCD visit table of the DDFs, made by Madeleine from the DP2 `VisitDetector` table and compatible with `skysurvey` |
| `final_sample_ddf.parquet` (~1.6 MB) | Real DDF SN Ia candidates (SALT parameters, DIA object position, forced-photometry light curve) |
| `filter_lsst_{u,g,r,i,z,y}.npy` | Rubin transmission curves from the butler (wavelength in nm, transmission) |
| `SEARCHEFF_PIPELINE_LSST_DC2_Run2.2i_WFD.DAT` | SNANA search efficiency as a function of S/N, one block per band (estimated from Sanchez+22) |
| `EDP2_fluxerr_hist_per_band.pdf` | Output of `Data_vs_sims_comparison.ipynb`: histograms of simulated and real flux errors per band |

## Requirements

- Kernels: `conda_py313` (laptop), `desc-td-env` (NERSC, not `-dev`). The original notebooks were saved with a Python 3.12 kernel.
- Python packages: `skysurvey`, `sncosmo`, `astropy`, `numpy`, `scipy`, `pandas`, `matplotlib`, `shapely`, `nested_pandas` (`Data_vs_sims` notebooks), and `dustmaps` with `requests` (adapted notebooks, to download and read the SFD maps).
- The input files of the previous table must be in the working directory when running the notebooks.

## At NERSC

The notebook can be found here: `/global/cfs/cdirs/lsst/groups/TD/SN/EDP2/skysurvey_sims/Simulate DDFs.ipynb`

And perhaps the data that are not saved in this directory.

To copy a file to NERSC:

```bash
scp final_sample_ddf.parquet dagoret@dtn01.nersc.gov:/pscratch/sd/d/dagoret/SimTtranscientLSSTSurvey/sprints/Example_notebooks_madeleine2026/.
```

## Remarks on Slack

2026-10-26 : Madeleine Ginolin  
Pour ce notebook j'utilise la table des visites par CCD (que j'ai mis dans le .zip) qui est sur RSP (à /rubin/lsdb_data/public-files/visit_detector.parquet, et le schema est là https://sdm-schemas.lsst.io/dp2.html#VisitDetector) et que j'ai modifié pour qu'elle soit compatible avec skysurvey. Tu peux aussi utiliser directement la database dp2_visits.db avec skysurvey.LSST.from_opsim(opsim_path) (que tu as l'air d'utiliser dans tes notebooks) avec mais elle est moins précise parce que tout est calculé au niveau d'un pointé (la limite de détection à 5sigma varie beaucoup sur toute la footprint donc les logs par CCD sont plus précis, et ça prend aussi en compte si un détecteur n'as pas été utilisé)
