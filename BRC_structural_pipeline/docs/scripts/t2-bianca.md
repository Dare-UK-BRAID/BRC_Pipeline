# run_T2_bianca.sh — White Matter Lesion Detection

**Location:** `BRC_structural_pipeline/scripts/run_T2_bianca.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 16 Mar 2021

!!! note
    This script is called automatically by `run_T2_preprocessing.sh` when `--regtype` is 2 or 3. It does not need to be run manually.

---

## Purpose

Applies **FSL BIANCA** (Brain Intensity AbNormality Classification Algorithm) to detect white matter lesions on T2-FLAIR images. BIANCA is a supervised k-nearest-neighbour classifier. The BRC pipeline uses a pre-trained classifier stored in `BRC_GLOBAL_DIR/templates/bianca_class_data`.

---

## Pre-flight Validation

Before running BIANCA, the script verifies that all six required files are present. If any file is missing, it exits with a descriptive error:

```
Problem running Bianca. File <path> is missing
```

**Required files:**

| File | Location |
|------|----------|
| `T1_unbiased_brain.nii.gz` | T1 temp folder |
| `T1_unbiased.nii.gz` | T1 temp folder |
| `T2_unbiased.nii.gz` | T2 working directory |
| `T1_to_MNI_nonlin_coeff_inv.nii.gz` | T1 registration temp folder |
| `T1_to_MNI_linear.mat` | T1 registration temp folder |
| `T1_brain_pve_0.nii.gz` | T1 FAST folder |

---

## Processing Steps

### 1. Create Inclusion Mask

```bash
make_bianca_mask \
    T1_unbiased.nii.gz \
    T1_brain_pve_0.nii.gz \
    T1_to_MNI_nonlin_coeff_inv.nii.gz
```

Generates a white matter / periventricular inclusion mask (`T1_unbiased_bianca_mask`) to remove grey matter from BIANCA results.

### 2. Generate Configuration File

```bash
echo "T1_brain.nii.gz T2_unbiased.nii.gz T1_to_MNI_linear.mat" \
    > conf_file.txt
```

Single-line config listing the T1, T2-FLAIR, and T1→MNI affine for BIANCA's feature extraction.

### 3. Run BIANCA

```bash
bianca \
    --singlefile=conf_file.txt \
    --querysubjectnum=1 \
    --brainmaskfeaturenum=1 \
    --loadclassifierdata=bianca_class_data \
    --matfeaturenum=3 \
    --featuresubset=1,2 \
    -o bianca_mask
```

Features used: T2-FLAIR intensity (`featuresubset=1`) and T1 intensity (`featuresubset=2`).

### 4. Apply Inclusion Mask and Threshold

```bash
fslmaths bianca_mask \
    -mul T1_unbiased_bianca_mask.nii.gz \
    -thr 0.8 \
    -bin \
    final_mask
```

Probability values ≥ 0.8 within the WM inclusion mask are kept and binarised.

### 5. Extract Lesion Volume

```bash
fslstats final_mask -V | awk '{print $1}' > volume.txt
```

Total voxel count written to `volume.txt`.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--workingdir` | Path | T2 temp folder (provides `T2_unbiased.nii.gz`) |
| `--tempt1folder` | Path | T1 temp folder (provides `T1_unbiased` files) |
| `--fastfolder` | Path | FAST folder (provides `T1_brain_pve_0` for inclusion mask) |
| `--biancatempfolder` | Path | BIANCA working directory for all outputs |
| `--regtempt1folder` | Path | T1 registration temp folder (MNI warp and affine) |
| `--logfile` | Path | Log file path |

---

## Key Outputs

| File | Description |
|------|-------------|
| `BIANCA/bianca_mask.nii.gz` | Raw BIANCA probability map (0–1 per voxel) |
| `BIANCA/final_mask.nii.gz` | Binary lesion mask (threshold 0.8, inclusion mask applied) |
| `BIANCA/volume.txt` | Total lesion volume in voxels |
| `BIANCA/T1_unbiased_bianca_mask.nii.gz` | WM inclusion mask |
| `BIANCA/T1_unbiased_ventmask.nii.gz` | Ventricular mask (from `make_bianca_mask`) |
| `BIANCA/conf_file.txt` | BIANCA input configuration file |
