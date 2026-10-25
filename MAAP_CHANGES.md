# Changes from upstream

Upstream: https://github.com/UWGlaciology/CommunityFirnModel, tag `v3.2.0` (branch `main`) at commit ab871c857534025b41725ff6434ef958136baa97 (2026-08-28), by C. Max Stevens and the CFM contributors listed in `CITATION.cff` (University of Washington Glaciology), MIT License (`LICENSE.txt`, Copyright (c) 2018 UW Glaciology). Please cite Stevens et al. (2020), GMD, https://doi.org/10.5194/gmd-13-4355-2020.

This copy starts from upstream's file tree at ab871c8 without its git history. Nothing was left out (upstream has no `.github/` folder). It is unchanged except for the items below. The physics, the example inputs and the committed reference outputs in `CFM_main/CFMoutput_example/` are untouched.

## 2026-10-25: unused psutil import removed

`CFM_main/firn_density_nospin.py` imported `psutil` but never used it, and `psutil` is not in `requirements.txt` or `setup.py`. A fresh install with the listed packages stopped at import with `ModuleNotFoundError: No module named 'psutil'`. The import is gone; nothing else changed. Results do not change.

## 2026-10-25: rolling-mean climate works with pandas 3

The 10-year rolling means of surface temperature and accumulation (`T_mean`, `bdot_av`, `firn_density_nospin.py` around line 537) come from `pd.Series(...).rolling(...).mean().values`, and the NaNs at the start are then filled in place. With pandas 3 (copy-on-write), `.values` is read-only, so the fill raised `ValueError: assignment destination is read-only`; the `except` fell back to the mean over the whole record and printed only `Error calculating T_mean, using mean surface over all time`. Since `T_mean` and `bdot_av` drive the densification rate in most physics options, a pandas 3 install quietly gave different answers from a pandas 2 install.

Both arrays are now copied before the fill (`.values.copy()`), the same change as on upstream's `dev` branch (commit 423514da645c6f6fd17c2ef861ddad6e298744df, 2026-09-21).

Results: with pandas 2 nothing changes (both release examples identical before and after). With pandas 3 the model now uses the rolling mean as intended and reproduces the pandas 2 results. For the release examples that is a change of up to 68 kg/m3 at single depth points, the time-mean depth of 550 kg/m3 moves from 16.0 to 16.2 m and the mean firn air content from 26.07 to 26.14 m (csv) and 26.22 to 26.29 m (dataframe). The reference outputs in `CFM_main/CFMoutput_example/` match the old pandas 3 fallback, so they now differ from a pandas 3 or pandas 2 run by those amounts; they were not regenerated here.

## 2026-10-25: setup.py version change taken out again

Tag `v3.2.0.post1` also changed the version in `setup.py` from `1.1.9` to `3.2.0` and added `setup.py` to the release checklist in `changelog.md`. That does not affect whether the model runs or what it computes, so both are back as the authors have them.

## Notes for running it

- Install: a conda (conda-forge) Python 3 with `numpy scipy h5py pandas xarray netcdf4 matplotlib`, then clone this repository. `setup.py` is not needed for the examples; the modules import each other from `CFM_main/`.
- Run from `CFM_main/`: input and output folders in the JSON configs are relative to the working directory.
- `python main.py example_csv.json -n` and `python main.py example_df.json -n` write into `CFMoutput_example/csv` and `/df` and overwrite the committed reference files there. The results folder is cleared at the start of each run.

## Check

On 2026-10-25, on Linux (x86_64), in a fresh conda env from conda-forge (Python 3.12.15, numpy 2.5.4, scipy 1.18.1, h5py 3.16.0, pandas 3.1.0, xarray 2026.9.0, netCDF4 1.7.4, matplotlib 3.11.2, no psutil), from a fresh clone, in `CFM_main/`:

```sh
python main.py example_csv.json -n
python main.py example_df.json -n
python run_CFM_example.py "72.5,-38.75"
```

All three ran to the end with exit code 0 and without the `Error calculating T_mean` message. The csv run took 9 s and the dataframe run 6 s (0.19 GB peak each); the daily Summit example with the surface energy balance took 6.4 minutes on 2 cores and 1.24 GB peak, and wrote a 771 MB results file.

Compared with the unchanged code run with pandas 2.3.3 (where the rolling mean worked), the csv example is identical in every dataset, and the dataframe example agrees to float32 rounding (largest density difference 0.0003 kg/m3). Compared with the committed reference outputs, both examples differ by the amounts given above for the rolling-mean change (depth, liquid water, melt, refreeze and runoff are identical). The unchanged code with pandas 3 reproduces the reference outputs (csv to within 3e-11 m in DIP, dataframe to float32 rounding).
