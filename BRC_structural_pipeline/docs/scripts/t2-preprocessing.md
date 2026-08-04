# run_T2_preprocessing.sh — Core T2 Preprocessing

!!! info "Target Audience"
    Neuroimaging researchers using T2-FLAIR images.

**Location:** `BRC_structural_pipeline/scripts/run_T2_preprocessing.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

---

## Purpose

Preprocesses the T2-weighted (FLAIR) image by co-registering it to the T1, applying the T1 brain mask, computing the T2→MNI registration, applying de-facing, and performing bias field correction using the T1-derived bias field. For RegType 2 and 3, it also invokes BIANCA for white matter lesion detection.

---

## Processing Steps

### 1. Co-registration to T1

The T2 is registered to the T1 space. The approach differs by `--regtype`:

=== "RegType 1"

    ```bash
    # Crop + BET the T2
    bet T2_tmp T2_tmp_brain -R -m -f 0.50
    # Register T2 brain to T1 brain (6 DOF, corratio cost)
    flirt -in T2_tmp_brain -ref T1_brain -dof 6 -cost corratio \
          -omat T2_orig_ud_to_T2.mat
    ```

=== "RegType 2 / 3"

    ```bash
    # Initial T2→T1_orig_ud alignment
    flirt -in T2_orig_ud -ref T1_orig_ud -dof 6 -omat T2_tmp.mat
    # Refine using T1 brain mask as weighting
    flirt -in T2_orig_ud -ref T1_brain -refweight T1_brain_mask \
          -nosearch -init T2_tmp2.mat -dof 6 -omat T2_orig_ud_to_T2.mat
    ```

    The T1 brain mask is used directly as the T2 brain mask.

### 2. T2→MNI Linear Transform

```bash
convert_xfm -omat T2_orig_ud_to_MNI_linear.mat \
    -concat T1_to_MNI_linear.mat T2_orig_ud_to_T2.mat
```

Combines the T2→T1 affine with the T1→MNI affine to get the T2→MNI transform without re-running registration.

### 3. Apply Linear Warp to MNI

```bash
applywarp --rel -i T2_orig -r MNI152_T1_1mm \
    -o T2_to_MNI_linear \
    --premat=T2_orig_ud_to_MNI_linear.mat --interp=spline
```

### 4. Non-Linear Warp to MNI *(RegType 2 or 3)*

```bash
applywarp --rel --interp=spline \
    -i T2_unbiased -r MNI152_T1_1mm \
    -w T1_to_MNI_nonlin_field \
    -o T2_to_MNI
```

The T1 non-linear warp field is reused — no separate T2 registration is run.

### 5. De-Facing *(optional)*

The MNI facemask is warped to T2 native space using the concatenated T2→MNI affine and then inverted. Facial voxels are zeroed in both the T2_orig and T2.

### 6. Bias Field Correction

```bash
fslmaths T2.nii.gz -div T1_brain_bias.nii.gz T2_unbiased.nii.gz
fslmaths T2_brain.nii.gz -div T1_brain_bias.nii.gz T2_unbiased_brain.nii.gz
```

The T1-derived bias field (from FSL FAST) is applied to the T2.

### 7. BIANCA Lesion Detection *(RegType 2 or 3)*

Calls [`run_T2_bianca.sh`](t2-bianca.md) for white matter lesion detection.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--workingdir` | Path | T2 temporary working directory |
| `--t2input` | File path | Path to `T2_orig.nii.gz` |
| `--tempt1folder` | Path | T1 temp folder (co-registration reference) |
| `--fastfolder` | Path | T1 FAST folder (provides bias field) |
| `--regtempt1folder` | Path | T1 registration temp folder (T1→MNI transforms) |
| `--regtempt2folder` | Path | T2 registration temp folder (output) |
| `--dodefacing` | yes/no | Apply de-facing to T2 |
| `--regtype` | 1/2/3 | Registration strategy (must match T1) |
| `--docrop` | yes/no | Apply FOV cropping (RegType 1 only) |
| `--biancatempfolder` | Path | BIANCA temporary directory |
| `--t2lesionpath` | Path | Pre-existing lesion mask to propagate to T1 space |
| `--logfile` | Path | Log file path |

---

## Key Outputs

| File | Description |
|------|-------------|
| `T2.nii.gz` | T2 aligned to T1 native space |
| `T2_brain.nii.gz` | Skull-stripped T2 (T1 brain mask applied) |
| `T2_brain_mask.nii.gz` | Brain mask (copied from T1) |
| `T2_unbiased.nii.gz` | Bias-corrected T2 |
| `T2_unbiased_brain.nii.gz` | Bias-corrected skull-stripped T2 |
| `T2_orig_defaced.nii.gz` | De-faced T2 original *(if defacing enabled)* |
| `reg/T2_to_MNI_linear.mat` | T2 native → MNI affine |
| `reg/T2_to_MNI_linear.nii.gz` | Bias-corrected T2 in MNI (linear) |
| `reg/T2_to_MNI.nii.gz` | Bias-corrected T2 in MNI (non-linear, RegType 2/3) |
