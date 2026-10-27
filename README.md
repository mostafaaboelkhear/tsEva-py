# tsEva-py #

This is a python implementation of transformed-stationary EVA (tsEva), a methodology for non-stationary extreme value analysis. The original MATLAB implementation can be found at https://github.com/menta78/tsEva.


## References ##

* [Mentaschi, L., Vousdoukas, M., Voukouvalas, E., Sartini, L., Feyen, L., Besio, G., and Alfieri, L.: The transformed-stationary approach: a generic and simplified methodology for non-stationary extreme value analysis, Hydrol. Earth Syst. Sci., 20, 3527–3547, doi:10.5194/hess-20-3527-2016, 2016](http://www.hydrol-earth-syst-sci.net/20/3527/2016/)

* [Bahmanpour, M. H., Tilloy, A., Vousdoukas, M., Federico, I., Coppini, G., Feyen, L., and Mentaschi, L.: Transformed-stationary EVA 2.0: a generalized framework for non-stationary multivariate extremes analysis, Hydrol. Earth Syst. Sci., 30, 2301–2314](https://hess.copernicus.org/articles/30/2301/2026/)




## Native exPostCorrection in the two main Python modules

The correction **no longer requires an adapter**. The updated modules have
separate, well-defined roles:

- `tsEva.py`: public `tsEvaExPostCorrection(time, stationary_series, thresholds,
  windows, theta_original, min_peak_distance_days=..., max_lag_days=...)`
  computes the bivariate event counts (`N11`, `N10`, `N01`), symmetric ratio,
  original/corrected Gumbel tail dependence, and per-window corrected theta.
  The bivariate calculation is directly available from the monovariate module,
  but is only meaningful when supplied with two stationary marginals.
- `tsEvaMultivariate.py`: the public `tsCopulaExtremes(...)` function calls the
  calculation during bivariate Gumbel/GPD fitting, attaches
  `copulaAnalysis['exPostCorrection']`, and can apply the corrected theta to
  `copulaAnalysis['copulaParam']['rho']` for downstream Monte Carlo and plots.
  It also natively supplies the sampling thresholds, full joint-event
  indices, and MATLAB-style 1-based per-margin peak indices.

### Usage

```python
import tsEvaMultivariate as tsm

copulaAnalysis = tsm.tsCopulaExtremes(
    time, series2D,
    copulaFamily='gumbel',
    marginalDistributions='gpd',
    exPostCorrection=True,           # calculate corrections (default on supported analyses)
    applyExPostCorrection=True,      # use corrected theta downstream (default False)
    # plus the existing sampling/fit arguments
)

correction = copulaAnalysis['exPostCorrection']
print(correction['thetaCorrected'])
```

`caseStudy01.py` imports `tsEvaMultivariate` directly and enables both flags.
It prints correction audit results in the terminal and saves only `CaseStudy01_output.png` (no exPostCorrection CSV).
The original fitted matrices remain in
`copulaAnalysis['copulaParam']['rhoFittedBeforeExPostCorrection']`, while the
simulation uses corrected `rho`. To inspect the correction *without applying*
it, use `applyExPostCorrection=False`.

`caseStudy02.py` (trivariate) and `caseStudy03.py` (GEV marginals) also import
`tsEvaMultivariate` directly. Applying exPostCorrection in unsupported modes
raises an explicit error; no correction parameters are fabricated.

Compatibility files `tseva_matlab_parity_adapter.py` and
`tseva_expost_reconstruction_corrected.py` are retained only for existing
callers and optional MATLAB comparison CLI. Their implementations delegate to
the core modules; they are **not required by the case studies**.

### Verification and scientific boundaries

- Case Study 01 completed: 108 compound events and 31 corrected windows.
  All nine exPostCorrection CSV columns are identical to the prior adapter
  implementation (maximum absolute difference of zero).
- Case Study 03 completed using the native imports and its original GEV mode.
- `python3 -m unittest discover -s test_multivariate -p 'test_expost_correction.py'`
  tests event counts, correction formulas, NaN handling, and API guards.

**Important:** The ex-post correction equation was reconstructed numerically
from MATLAB reference outputs. The original MATLAB correction function was not
provided. Exact MATLAB parity for NaN-containing records has not yet been
independently validated. Current implementation scope: **bivariate Gumbel
copula with GPD marginals only**. No GEV or trivariate correction is claimed.

### WSL execution

```bash
cd /mnt/g/4-train_the_model_linear_mostafa/ML10/FFF_LORENZO/tsEva-py-dev
python3 -m unittest discover -s test_multivariate -p 'test_expost_correction.py'
cd test_multivariate
python3 caseStudy01.py
```

The untouched uploaded core files remain under `backup_original/`. Previous
project state immediately before integrating the native correction is backed
up in `backup_before_native_expost/` in the full-project archive.


### exPostCorrection across the example studies

The core also provides `tsEvaExPostCorrectionPairs` for multiple Gumbel pairs.
CaseStudy01 continues to use the reference-tested bivariate GPD pathway.
CaseStudy02 computes pairwise threshold-event corrections for all 3 pairs of
its trivariate GPD analysis. CaseStudy03 computes a bivariate **annual-maxima
matching** correction for GEV. **Cases 02/03 are experimental, not validated
against MATLAB**; the trivariate vine may not reproduce the complete corrected
pairwise target. Their scripts explicitly set `allowExperimentalExPostCorrection=True`
when applying corrected copula parameters. The scripts print correction audits
in the terminal; no exPostCorrection CSV files are written.

To compute but *not* apply an experimental correction, set
`applyExPostCorrection=False` in the case study. For analyses lacking a
supported correction, pass `exPostCorrection=False`.


### exPostCorrection terminal audit (no CSV files)

All three `test_multivariate/caseStudy0X.py` scripts now print:

- `CALCULATED: YES/NO`, plus number of time windows and variable pairs.
- `NUMERICAL CHECKS: PASS`, after verifying event counts, symmetric ratio,
  original/corrected lambda and theta, and the values in the actual active
  `copulaParam['rho']` matrices. Invalid values **raise a ValueError**.
- `APPLIED TO COPULA: YES/NO`, showing the largest change in theta.
- `METHOD: ...`, clearly labeling experimental trivariate/GEV extensions.
- `MONTE CARLO: CORRECTED parameters USED`, printed **after** probability
  simulation, checking the parameters actually submitted. When
  `nonStationarity='margins'`, Monte Carlo uses the **mean** of corrected
  theta values, so the terminal notes this aggregation explicitly.

These are implementation-consistency checks, **not an independent scientific
validation** of exPostCorrection or MATLAB equivalence. NaN peak detection
parity is unverified; Case 02/03 are experimental.

The case studies do **not write** `CaseStudy0X_exPostCorrection.csv` files.
They continue to save the PNG figures. Existing CSV files from earlier runs
are not deleted by the script; manually delete them if no longer needed.

Run tests:

```bash
python3 -m unittest discover -s test_multivariate -p 'test_expost_correction.py'
```
