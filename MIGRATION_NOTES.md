# Migration notes — native exPostCorrection

## What changed

- `tsEva.py`: contains the bivariate Gumbel exPostCorrection statistical core,
  including NaN-aware peak detection, chronological threshold-event matching,
  N11/N10/N01 counts, rSymmetric and corrected theta/lambda calculations.
  Public entry point: `tsEva.tsEvaExPostCorrection(...)`.
- `tsEvaMultivariate.py`: integrates the correction **directly** into
  `tsCopulaExtremes`. It also exposes sampler outputs without a temporary
  monkey-patch, reconstructs each margin's timestamp index and preserves
  MATLAB-style 1-based sampling indices. Applying correction updates `rho` for
  downstream Monte Carlo, while storing uncorrected rho separately.
- `test_multivariate/caseStudy01.py`, `caseStudy02.py`, `caseStudy03.py`:
  import `tsEvaMultivariate as tsm` directly, not an adapter.
- `tseva_matlab_parity_adapter.py`: now a backwards-compatible import alias.
- `tseva_expost_reconstruction_corrected.py`: optional CLI and backwards-
  compatible functions that delegate to the core `tsEva.py` implementation.
- The original uploaded code is in `backup_original/`; the state immediately
  before this native integration is in `backup_before_native_expost/`.

## exPostCorrection interface

In `tsCopulaExtremes`:

```python
exPostCorrection=True,       # compute fields (default True)
applyExPostCorrection=True,  # apply corrected theta (default False)
```

Supported for correction calculation: **2 or 3 variables, Gumbel copula, GPD or GEV marginals**. The existing bivariate GPD path is reference-tested (except MATLAB NaN parity). Trivariate GPD and bivariate GEV are explicitly experimental and require `allowExperimentalExPostCorrection=True` to *apply* their corrected parameters. Unsupported configurations raise if `exPostCorrection=True` is explicitly requested.

Outputs:

- `copulaAnalysis['exPostCorrection']` with `N11`, `N10`, `N01`,
  `rSymmetric`, `thetaOriginal`, `thetaCorrected`, `lambdaOriginal`,
  `lambdaCorrected`, and diagnostic metadata.
- `copulaAnalysis['exPostCorrectionDiagnostics']` with information about the
  peak distances, matching lag, and missing-data parity limits.
- If `applyExPostCorrection=True`:
  `copulaAnalysis['copulaParam']['rhoFittedBeforeExPostCorrection']` preserves
  original fitted matrices, while `rho` receives corrected theta values.
  Monte Carlo and return-period calculations consume `rho` downstream.

### Case Study 01 settings

- Marginal peak spacing: **[30, 30] days**.
- Matching window: **45 days**.
- 31 corrected windows; 108 compound peaks in a full run.
- Earlier Case Study 01 numerical CSV results matched the previous adapter;
  the updated script now prints diagnostics and does not export CSV.

### Scientific limitation

The original MATLAB exPostCorrection source was not supplied. The formula and
counts were inferred/validated from saved MATLAB results on finite records.
For input with NaNs (Case 01 has them), the equivalent Python peak finder is
used, but exact agreement with MATLAB for those NaN-containing records has not
been independently established. Do not present this as a direct translation of
an unavailable MATLAB function. The trivariate GPD and bivariate GEV extensions are **exploratory**, not a verified MATLAB translation. Do not report them as MATLAB exPostCorrection parity or a validated trivariate copula calibration.

## Installation in existing WSL checkout

Extract the drop-in replacement ZIP **from the project root**:

```bash
cd /mnt/g/4-train_the_model_linear_mostafa/ML10/FFF_LORENZO/tsEva-py-dev
cp tsEva.py tsEva.py.bak
cp tsEvaMultivariate.py tsEvaMultivariate.py.bak
unzip -o /path/to/tsEva_native_exPostCorrection_drop_in.zip -d .
python3 -m unittest discover -s test_multivariate -p 'test_expost_correction.py'
cd test_multivariate
python3 caseStudy01.py
```

The ZIP does not delete your existing data or other project files.


## Additional experimental extensions (all case studies)

- `tsEva.tsEvaExPostCorrectionPairs(...)` handles pairwise threshold-peak events
  for 2-3 variables, and optionally timestamp-matched annual maxima for GEV.
  It exports `pairList` (one-based), corrected/original theta/lambda, N11/N10/N01
  and `rSymmetric` per pair and sliding window.
- `tsEvaMultivariate.tsCopulaExtremes(...)` now computes these results for the
  trivariate Gumbel/GPD Case 02 and bivariate Gumbel/GEV Case 03, as well as
  the existing bivariate Gumbel/GPD Case 01. Both extension types are marked
  experimental in `exPostCorrectionDiagnostics`.
- `caseStudy02.py` and `caseStudy03.py` explicitly choose:

```python
exPostCorrection=True,
applyExPostCorrection=True,
allowExperimentalExPostCorrection=True,
```

- The corrected pairwise Gumbel theta values are placed in `copulaParam['rho']`
  while originals are kept in `rhoFittedBeforeExPostCorrection`.
- For the trivariate C-vine, these pairwise inputs do not guarantee that the
  conditional dependence or the full simulated multivariate copula reproduces
  every targeted corrected pairwise tail coefficient. Additional scientific
  validation is required before treating the output as a calibrated method.
- The GEV annual-maxima correction counts matched **annual maxima event times**
  (not POT exceedances), using the Case 03 lag of 360 days. It is an
  exploratory adaptation, not the MATLAB POT exPostCorrection derivation.
  With long matching lags the overlap can be near 1, yielding small changes.
- Ex-post results are available in-memory in `copulaAnalysis['exPostCorrection']`
  and the code prints terminal audit summaries. CSV export has been removed.

Command to run tests:

```bash
python3 -m unittest discover -s test_multivariate -p 'test_expost_correction.py'
```


## Update: terminal audit instead of CSV export

- Each case study prints calculated/applied status, counts of windows and pairs,
  max theta change, method validation status and actual Monte Carlo use.
- `tsEvaMultivariate.tsEvaAuditExPostCorrection()` cross-checks counts,
  `rSymmetric`, corrected lambda/theta, and the active copula rho values.
  It raises on mismatch instead of falsely reporting success.
- Monte Carlo reports after probability samples are generated;
  `nonStationarity='margins'` is explicitly reported as averaging theta.
- The case studies do not save correction CSV files. PNG outputs remain.
- The audit verifies numerical consistency, not scientific correctness or
  equivalence to original MATLAB exPostCorrection. Cases 02/03 are experimental.
