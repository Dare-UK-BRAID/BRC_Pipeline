# move_rename.sh — Legacy Output Reorganisation

**Location:** `BRC_structural_pipeline/scripts/move_rename.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

!!! warning "Legacy Script"
    This script is designed for the **legacy `fsl_anat`-based preprocessing path** (`FSL_anat.sh` → `move_rename.sh`). The current default path uses `run_T1_preprocessing.sh` → `output_organization.sh`. See [Difference from output_organization.sh](#difference-from-output_organizationsh) below.

---

## Purpose

Moves and renames files from the `<T1Folder>/<anatFolder>/` directory structure produced by `FSL_anat.sh` into the canonical BRC folder hierarchy. When T2 data is present, it also co-registers the T2 to the T1 using `epi_reg` (for better white-matter-boundary alignment), optionally runs multi-channel FAST, and computes the full T2→MNI warp.

---

## Operations

### T1 Data

| Source (`<anatFolder>/`) | Destination |
|--------------------------|-------------|
| `T1_biascorr` | `processed/data/T1` |
| `T1_biascorr_brain` | `processed/data/T1_brain` |
| `T1_biascorr_brain_mask` | `processed/data/T1_brain_mask` |
| `T1_fast_pve_0` | `seg/tissue/sing_chan/T1_pve_CSF` |
| `T1_fast_pve_1` | `seg/tissue/sing_chan/T1_pve_GM` |
| `T1_fast_pve_2` | `seg/tissue/sing_chan/T1_pve_WM` |
| `T1_fast_pveseg` | `seg/tissue/sing_chan/T1_pveseg` |
| `T1_fast_seg` | `seg/tissue/sing_chan/T1_seg` |
| `T1_to_MNI_linear` | `processed/data2std/T1_2_std` |
| `T1_to_MNI_linear_temp.mat` | `preproc/reg/T1_2_std.mat` |
| `T1_to_MNI_nonlin` | `processed/data2std/T1_2_std_warp` |
| `T1_to_MNI_nonlin_coeff` | `preproc/reg/T1_2_std_warp_coeff` |
| `T1_to_MNI_nonlin_field` | `preproc/reg/T1_2_std_warp_field` |
| `T1_to_MNI_nonlin_jac` | `preproc/reg/T1_2_std_warp_jac` |
| `MNI_to_T1_nonlin_field` | `preproc/reg/std_2_T1_warp_field` |

### Subcortical Structures *(if `--dosubseg=yes`)*

FIRST shape files (`.bvars` and `.vtk`) for the following structures are moved to `seg/sub/shape/`:

Accumbens (L/R), Amygdala (L/R), Caudate (L/R), Hippocampus (L/R), Pallidum (L/R), Putamen (L/R), Thalamus (L/R), Brainstem.

### T2 Co-registration and Organisation *(if T2 present)*

T2→T1 co-registration uses **epi_reg** (boundary-based registration) for improved accuracy at the white matter boundary:

```bash
epi_reg \
    --epi=T2_brain \
    --t1=T1 \
    --t1brain=T1_brain \
    --out=reg/T2_2_T1_init \
    --wmseg=sing_chan/T1_pve_thr_WM

flirt -in T2_brain -ref T1_brain \
      -init T2_2_T1_init.mat \
      -out reg/T2_2_T1 -omat reg/T2_2_T1.mat -dof 6
```

If multi-channel segmentation is run, `epi_reg` is re-run using the multi-channel WM mask for improved alignment, and the T2→MNI non-linear warp is produced via:

```bash
applywarp --rel \
    --in=T2 \
    --ref=MNI152_T1_1mm \
    --premat=T2_2_T1.mat \
    --warp=T1_2_std_warp_coeff \
    --out=data2std/T2_to_std_warp
```

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--t1folder` | Path | Root T1 folder |
| `--t2folder` | Path | Root T2 folder |
| `--t2exist` | yes/no | Whether T2 data is present |
| `--dosubseg` | yes/no | Whether subcortical segmentation was run |
| `--dotissueseg` | yes/no | Whether to run multi-channel FAST |
| `--anatname` | String | Name of the `fsl_anat` output directory within T1/T2 folder |
| `--datat1folder` | Path | Destination for processed T1 data |
| `--data2stdt1folder` | Path | Destination for T1 in standard space |
| `--segt1folder` | Path | Destination for T1 segmentation outputs |
| `--regt1folder` | Path | Destination for T1 registration files |
| `--tempt1folder` | Path | T1 temp folder |
| `--datat2folder` | Path | Destination for processed T2 data |
| `--data2stdt2folder` | Path | Destination for T2 in standard space |
| `--regt2folder` | Path | Destination for T2 registration files |
| `--tempt2folder` | Path | T2 temp folder |
| `--logfile` | Path | Log file path |

---

## Difference from output_organization.sh

| Aspect | `move_rename.sh` | `output_organization.sh` |
|--------|-----------------|--------------------------|
| **Preprocessing path** | Legacy: `FSL_anat.sh` | Current: `run_T1_preprocessing.sh` |
| **Input naming** | `T1_biascorr` | `T1_unbiased` |
| **T2→T1 co-reg** | `epi_reg` (boundary-based) | T1 brain mask applied directly |
| **Subcortical** | Moves FIRST `.bvars`/`.vtk` to `shape/` | Subcortical support removed (commented) |
| **FSLanat temp** | Copies residual `fsl_anat` files to `FSLanat/` | No such copy |
