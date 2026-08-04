# struc_preproc_part_1.sh — Stage 1 Dispatcher

**Location:** `BRC_structural_pipeline/scripts/struc_preproc_part_1.sh`

---

## Purpose

Thin dispatcher script for processing Stage 1. Receives the full set of folder paths and processing flags from `struc_preproc.sh` and invokes:

1. `run_T1_preprocessing.sh` — core T1 processing
2. `run_T2_preprocessing.sh` — core T2 processing (only if `--t2=yes`)

T2 preprocessing runs *after* T1 preprocessing completes because it depends on T1 outputs (brain mask, bias field, registration transforms).

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--tempt1folder` | Path | T1 temporary working directory |
| `--rawt1folder` | Path | Directory containing `T1_orig.nii.gz` |
| `--dosubseg` | yes/no | Whether to run FSL FIRST subcortical segmentation |
| `--dotissueseg` | yes/no | Whether to run FSL FAST tissue segmentation |
| `--docrop` | yes/no | Whether to crop the field of view |
| `--dodefacing` | yes/no | Whether to apply de-facing |
| `--fastt1folder` | Path | FAST output temporary directory |
| `--firstt1folder` | Path | FIRST output directory |
| `--sienaxt1folder` | Path | SIENAX permanent output folder |
| `--biancatempfolder` | Path | BIANCA temporary working directory |
| `--regtempt1folder` | Path | T1 registration temporary folder |
| `--t2` | yes/no | Whether a T2 image is present |
| `--tempt2folder` | Path | T2 temporary working directory |
| `--rawt2folder` | Path | Directory containing `T2_orig.nii.gz` |
| `--regtempt2folder` | Path | T2 registration temporary folder |
| `--sienaxtempfolder` | Path | SIENAX temporary working directory |
| `--t2lesionpath` | Path | Path to the T2 lesion mask (may be empty) |
| `--fbet` | Float | BET fractional intensity threshold |
| `--regtype` | 1/2/3 | Registration strategy |
| `--logt1folder` | Path | Log file path |

---

## Calls

```
struc_preproc_part_1.sh
  ├── run_T1_preprocessing.sh   (always)
  └── run_T2_preprocessing.sh   (only if --t2=yes)
```
