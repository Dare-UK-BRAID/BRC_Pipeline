# Output Directory Structure

!!! info "Target Audience"
    Neuroimaging researchers interpreting pipeline outputs.

The pipeline creates a structured output hierarchy under `<path>/<subject_id>/`. This page describes the full directory layout and the key files produced at each location.

---

## Top-Level Layout

```
<path>/<subject_id>/
├── raw/
│   └── anatMRI/
│       ├── T1/
│       │   ├── T1_orig.nii.gz          # original input copy
│       │   └── T1_orig_defaced.nii.gz  # defaced original (if --nodefacing not set)
│       └── T2/
│           ├── T2_orig.nii.gz
│           └── T2_orig_defaced.nii.gz
├── analysis/
│   └── anatMRI/
│       ├── T1/
│       │   ├── preproc/
│       │   ├── processed/
│       │   ├── temp/                   # removed after pipeline completes
│       │   └── log/
│       └── T2/
│           ├── preproc/
│           ├── processed/
│           └── temp/
└── log/
```

---

## T1 Preprocessed Outputs (`analysis/anatMRI/T1/preproc/`)

| Sub-folder | Key Files | Description |
|------------|-----------|-------------|
| `reg/` | `T1_2_std.mat` | T1 → MNI linear affine matrix |
| `reg/` | `std_2_T1.mat` | MNI → T1 inverse affine |
| `reg/` | `T1_2_std_warp_field.nii.gz` | T1 → MNI non-linear warp (RegType 2/3) |
| `reg/` | `T1_2_std_warp_coeff.nii.gz` | FNIRT warp coefficients (RegType 2 only) |
| `reg/` | `std_2_T1_warp_field.nii.gz` | MNI → T1 inverse warp |
| `bias/` | `T1_brain_bias.nii.gz` | FAST-estimated bias field |
| `SIENAX/` | `report.sienax` | Brain volume report (mm³) |
| `qc/` | *(optional)* | Quality control outputs |

---

## T1 Processed Outputs (`analysis/anatMRI/T1/processed/`)

| Sub-folder | Key Files | Description |
|------------|-----------|-------------|
| `data/` | `T1.nii.gz` | Cropped, reoriented T1 (not bias-corrected) |
| `data/` | `T1_brain.nii.gz` | Skull-stripped T1 |
| `data/` | `T1_brain_mask.nii.gz` | Binary brain mask |
| `data/` | `T1_unbiased.nii.gz` | Bias-corrected whole-head T1 |
| `data/` | `T1_unbiased_brain.nii.gz` | Bias-corrected skull-stripped T1 |
| `data2std/` | `T1_2_std.nii.gz` | T1 in MNI152 space (linear) |
| `data2std/` | `T1_2_std_brain_lin.nii.gz` | Brain-only T1 in MNI (linear) |
| `data2std/` | `T1_2_std_warped.nii.gz` | T1 in MNI152 space (non-linear) |
| `data2std/` | `T1_2_std_brain_warped.nii.gz` | Brain-only T1 in MNI (non-linear) |
| `seg/tissue/sing_chan/` | `T1_pve_CSF.nii.gz` | CSF partial volume estimate |
| `seg/tissue/sing_chan/` | `T1_pve_GM.nii.gz` | Grey matter partial volume estimate |
| `seg/tissue/sing_chan/` | `T1_pve_WM.nii.gz` | White matter partial volume estimate |
| `seg/tissue/sing_chan/` | `T1_{CSF,GM,WM}_mask.nii.gz` | Thresholded binary tissue masks |
| `seg/tissue/sing_chan/` | `T1_pveseg.nii.gz` | FAST hard segmentation (integer labels) |
| `seg/tissue/multi_chan/` | `T1_pve_{CSF,GM,WM}.nii.gz` | Joint T1+T2 FAST segmentation (if T2 provided) |
| `seg/sub/` | `T1_subcort_seg.nii.gz` | FIRST subcortical segmentation (if `--subseg`) |
| `FreeSurfer/` | Full recon-all output | FreeSurfer subject directory (if `--freesurfer`) |
| `FastSurfer/` | Full FastSurfer output | FastSurfer subject directory (if `--fastsurfer`) |

---

## T2 Outputs (`analysis/anatMRI/T2/`)

| Path | Key Files | Description |
|------|-----------|-------------|
| `processed/data/` | `T2.nii.gz` | T2 aligned to T1 native space |
| `processed/data/` | `T2_brain.nii.gz` | Skull-stripped T2 |
| `processed/data/` | `T2_unbiased.nii.gz` | Bias-corrected T2 |
| `processed/data2std/` | `T2_2_std.nii.gz` | T2 in MNI space (linear) |
| `processed/data2std/` | `T2_2_std_warped.nii.gz` | T2 in MNI space (non-linear) |
| `preproc/reg/` | `T2_2_std.mat` | T2 → MNI affine |
| `preproc/lesions/` | `final_mask.nii.gz` | Binary WM lesion mask (BIANCA) |
| `preproc/lesions/` | `volume.txt` | Lesion volume in voxels |

---

## Log Files

A log file is created at `analysis/anatMRI/T1/log/log.txt` and captures all processing steps, timestamps, software versions, and parameter values.
