# Key Output Files Reference

All paths below are relative to `<path>/<subject_id>/analysis/anatMRI/`.

---

## T1 Native Space

| File | Description |
|------|-------------|
| `T1/processed/data/T1.nii.gz` | Cropped, reoriented T1 (not bias-corrected) |
| `T1/processed/data/T1_brain.nii.gz` | Skull-stripped T1 |
| `T1/processed/data/T1_brain_mask.nii.gz` | Binary brain mask |
| `T1/processed/data/T1_unbiased.nii.gz` | Bias-corrected whole-head T1 |
| `T1/processed/data/T1_unbiased_brain.nii.gz` | Bias-corrected skull-stripped T1 |

---

## T1 Standard Space

| File | Description |
|------|-------------|
| `T1/processed/data2std/T1_2_std.nii.gz` | T1 in MNI152 1mm (linear registration) |
| `T1/processed/data2std/T1_2_std_brain_lin.nii.gz` | Brain-only T1 in MNI (linear) |
| `T1/processed/data2std/T1_2_std_warped.nii.gz` | T1 in MNI152 1mm (non-linear, RegType 2/3) |
| `T1/processed/data2std/T1_2_std_brain_warped.nii.gz` | Brain-only T1 in MNI (non-linear) |

---

## Registration Transforms

| File | Description |
|------|-------------|
| `T1/preproc/reg/T1_2_std.mat` | T1 → MNI linear affine |
| `T1/preproc/reg/std_2_T1.mat` | MNI → T1 linear affine (inverse) |
| `T1/preproc/reg/T1_2_std_warp_field.nii.gz` | T1 → MNI non-linear warp field |
| `T1/preproc/reg/T1_2_std_warp_coeff.nii.gz` | FNIRT warp coefficients (RegType 2 only) |
| `T1/preproc/reg/T1_2_std_warp_jac.nii.gz` | Jacobian of non-linear warp (RegType 2 only) |
| `T1/preproc/reg/std_2_T1_warp_field.nii.gz` | MNI → T1 inverse warp field |

---

## Bias Field

| File | Description |
|------|-------------|
| `T1/preproc/bias/T1_brain_bias.nii.gz` | FAST-estimated bias field |

---

## Tissue Segmentation

### Single-Channel (T1 only)

| File | Description |
|------|-------------|
| `T1/processed/seg/tissue/sing_chan/T1_pve_CSF.nii.gz` | CSF partial volume estimate |
| `T1/processed/seg/tissue/sing_chan/T1_pve_GM.nii.gz` | Grey matter partial volume estimate |
| `T1/processed/seg/tissue/sing_chan/T1_pve_WM.nii.gz` | White matter partial volume estimate |
| `T1/processed/seg/tissue/sing_chan/T1_CSF_mask.nii.gz` | Binary CSF mask (PVE > 0.5) |
| `T1/processed/seg/tissue/sing_chan/T1_GM_mask.nii.gz` | Binary grey matter mask |
| `T1/processed/seg/tissue/sing_chan/T1_WM_mask.nii.gz` | Binary white matter mask |
| `T1/processed/seg/tissue/sing_chan/T1_pveseg.nii.gz` | FAST hard segmentation (integer labels) |
| `T1/processed/seg/tissue/sing_chan/T1_seg.nii.gz` | FAST segmentation map |

### Multi-Channel (T1 + T2, if T2 provided)

| File | Description |
|------|-------------|
| `T1/processed/seg/tissue/multi_chan/T1_pve_CSF.nii.gz` | Joint FAST CSF PVE |
| `T1/processed/seg/tissue/multi_chan/T1_pve_GM.nii.gz` | Joint FAST GM PVE |
| `T1/processed/seg/tissue/multi_chan/T1_pve_WM.nii.gz` | Joint FAST WM PVE |

---

## Brain Volume (SIENAX)

| File | Description |
|------|-------------|
| `T1/preproc/SIENAX/report.sienax` | Brain volume report with VSCALING, pgrey, vcsf, GREY, WHITE, BRAIN in mm³ |

---

## Cortical Reconstruction

| File | Description |
|------|-------------|
| `T1/processed/FreeSurfer/` | FreeSurfer `recon-all` subject directory (if `--freesurfer`) |
| `T1/processed/FastSurfer/` | FastSurfer subject directory (if `--fastsurfer`) |

---

## Raw / Defaced

| File | Description |
|------|-------------|
| `../../raw/anatMRI/T1/T1_orig.nii.gz` | Original T1 input copy |
| `../../raw/anatMRI/T1/T1_orig_defaced.nii.gz` | De-faced original T1 (if defacing enabled) |

---

## T2 Outputs (if T2 provided)

| File | Description |
|------|-------------|
| `T2/processed/data/T2.nii.gz` | T2 aligned to T1 native space |
| `T2/processed/data/T2_brain.nii.gz` | Skull-stripped T2 |
| `T2/processed/data/T2_unbiased.nii.gz` | Bias-corrected T2 |
| `T2/processed/data2std/T2_2_std.nii.gz` | T2 in MNI space (linear) |
| `T2/processed/data2std/T2_2_std_warped.nii.gz` | T2 in MNI space (non-linear) |
| `T2/preproc/reg/T2_2_std.mat` | T2 → MNI affine |
| `T2/preproc/lesions/final_mask.nii.gz` | Binary white matter lesion mask (BIANCA) |
| `T2/preproc/lesions/volume.txt` | Total lesion volume in voxels |
| `../../raw/anatMRI/T2/T2_orig_defaced.nii.gz` | De-faced original T2 |

---

## Log

| File | Description |
|------|-------------|
| `T1/log/log.txt` | Full processing log with timestamps, parameters, and software versions |
