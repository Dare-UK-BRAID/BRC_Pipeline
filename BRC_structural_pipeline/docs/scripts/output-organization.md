# output_organization.sh — Final Output Organisation

**Location:** `BRC_structural_pipeline/scripts/output_organization.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

---

## Purpose

Moves all intermediate files from temporary working directories into the canonical, human-readable BRC output folder hierarchy. This is the final "tidy-up" step that runs at the end of every successful pipeline execution.

Additionally, when T2 data is present and tissue segmentation is requested, this script runs **multi-channel FAST** (jointly on T1 and T2 brains) and automatically assigns CSF/GM/WM labels to the output PVE maps.

---

## Operations by Section

### T1 Data Organisation

Moves core T1 volumes from `temp/` to `processed/data/`:

| Source | Destination |
|--------|-------------|
| `temp/T1.nii.gz` | `processed/data/T1.nii.gz` |
| `temp/T1_brain.nii.gz` | `processed/data/T1_brain.nii.gz` |
| `temp/T1_brain_mask.nii.gz` | `processed/data/T1_brain_mask.nii.gz` |
| `temp/T1_unbiased.nii.gz` | `processed/data/T1_unbiased.nii.gz` |
| `temp/T1_unbiased_brain.nii.gz` | `processed/data/T1_unbiased_brain.nii.gz` |
| `temp/T1_orig_defaced.nii.gz` | `raw/T1/T1_orig_defaced.nii.gz` *(if defacing enabled)* |

### Segmentation Organisation

Moves FAST outputs to `processed/seg/tissue/sing_chan/` with renamed labels:

| Source | Destination |
|--------|-------------|
| `FAST/T1_brain_pve_0` | `sing_chan/T1_pve_CSF` |
| `FAST/T1_brain_pve_1` | `sing_chan/T1_pve_GM` |
| `FAST/T1_brain_pve_2` | `sing_chan/T1_pve_WM` |
| `FAST/T1_brain_CSF_mask` | `sing_chan/T1_CSF_mask` |
| `FAST/T1_brain_GM_mask` | `sing_chan/T1_GM_mask` |
| `FAST/T1_brain_WM_mask` | `sing_chan/T1_WM_mask` |
| `FAST/T1_brain_pveseg` | `sing_chan/T1_pveseg` |
| `FAST/T1_brain_seg` | `sing_chan/T1_seg` |
| `FAST/T1_brain_bias` | `preproc/bias/T1_brain_bias` |

### Registration Organisation

Linear registration files are moved to `processed/data2std/` and `preproc/reg/`:

| Source | Destination |
|--------|-------------|
| `reg/T1_to_MNI_linear` | `data2std/T1_2_std` |
| `reg/T1_brain_to_MNI_linear` | `data2std/T1_2_std_brain_lin` |
| `reg/T1_to_MNI_linear.mat` | `preproc/reg/T1_2_std.mat` |
| Computed inverse | `preproc/reg/std_2_T1.mat` |

For RegType 2 or 3, non-linear outputs are also organised:

| Source | Destination |
|--------|-------------|
| `reg/T1_to_MNI_nonlin` | `data2std/T1_2_std_warped` |
| `reg/T1_brain_to_MNI_nonlin` | `data2std/T1_2_std_brain_warped` |
| `reg/T1_to_MNI_nonlin_field` | `preproc/reg/T1_2_std_warp_field` |
| `reg/T1_to_MNI_nonlin_coeff_inv` | `preproc/reg/std_2_T1_warp_field` |
| `SIENAX/*` | `preproc/SIENAX/` |

### Multi-Channel Tissue Segmentation *(T2 + tissue seg enabled)*

When both T2 data and `--dotissueseg=yes` are active, FAST is run in multi-channel mode:

```bash
fast -o FAST -g -N -S 2 T1_brain T2_brain
```

Because FAST does not guarantee a fixed label ordering for multi-channel inputs, the script automatically assigns labels by ranking mean PVE intensities:

```bash
# Sort PVE means in ascending order
# Lowest  → CSF
# Middle  → GM
# Highest → WM
```

This ensures correct labelling regardless of FAST's internal ordering.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--t1folder` | Path | Root T1 folder |
| `--t2folder` | Path | Root T2 folder |
| `--datat1folder` | Path | Destination for processed T1 data |
| `--data2stdt1folder` | Path | Destination for T1 in standard space |
| `--segt1folder` | Path | Destination for segmentation outputs |
| `--regt1folder` | Path | Destination for T1 registration files |
| `--tempt1folder` | Path | T1 temp folder (source of most files) |
| `--t2exist` | yes/no | Whether T2 data is present |
| `--dotissueseg` | yes/no | Whether to run multi-channel FAST |
| `--dodefacing` | yes/no | Whether de-facing was applied |
| `--regtype` | 1/2/3 | Determines which non-linear files to move |
| `--sienaxt1folder` | Path | SIENAX permanent output folder |
| `--sienaxtempfolder` | Path | SIENAX temp folder (source) |
| `--biancat2folder` | Path | BIANCA permanent output folder |
| `--biancatempfolder` | Path | BIANCA temp folder (source) |
| `--logfile` | Path | Log file path |
