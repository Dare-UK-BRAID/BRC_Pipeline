# run_T1_preprocessing.sh — Core T1 Preprocessing

!!! info "Target Audience"
    Neuroimaging researchers and pipeline developers.

**Location:** `BRC_structural_pipeline/scripts/run_T1_preprocessing.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

---

## Purpose

Performs all core T1 image preprocessing. This is the most computationally intensive script in the pipeline. It combines brain extraction, registration to standard space, bias correction, tissue segmentation, subcortical segmentation, and brain volume estimation into a single coordinated workflow.

---

## Processing Steps

### 1. Copy and Reorient

```bash
fslreorient2std T1_orig_ud T1_orig_ud
```

The raw input is copied to the working directory as `T1_orig_ud` and reoriented to standard (RAS) orientation.

### 2. Field-of-View Cropping *(optional)*

```bash
robustfov -i T1_orig_ud
fslmaths T1_orig_ud -roi 0 -1 0 -1 $head_top 170 0 1 T1_tmp
```

Automatically crops the superior FOV to remove neck. Skipped when `--docrop=no`.

### 3. Initial Brain Extraction

```bash
bet T1_tmp T1_tmp_brain -R -m -f <fBET>
```

Recursive BET with fractional intensity threshold `fBET` (default 0.5). Produces an initial brain mask used in subsequent steps.

### 4. FOV Reduction to Head Region

```bash
standard_space_roi T1_tmp_brain T1_tmp2 -maskNONE -ssref MNI152_T1_1mm_brain -altinput T1_orig_ud -d
```

Reduces the FOV to the head region using a standard space reference. Output is renamed to `T1`.

### 5. Linear Registration to MNI152

```bash
flirt -interp spline -dof 12 -in T1 -ref MNI152_T1_1mm -omat T1_to_MNI_linear.mat
```

12-DOF affine registration to MNI152 1mm. The resulting `.mat` file is used to initialise non-linear registration.

### 6. Non-Linear Registration *(RegType 2 or 3)*

=== "RegType 2 — FNIRT"

    ```bash
    fnirt --in=T1 \
          --ref=MNI152_T1_1mm \
          --aff=T1_to_MNI_linear.mat \
          --config=bb_fnirt.cnf \
          --refmask=MNI152_T1_1mm_brain_mask_dil_GD7 \
          --cout=T1_to_MNI_nonlin_coeff \
          --fout=T1_to_MNI_nonlin_field
    ```

=== "RegType 3 — ANTs SyN"

    ```bash
    antsRegistrationSyN.sh -d 3 \
        -f MNI152_T1_1mm.nii.gz \
        -m T1.nii.gz \
        -o SynOut \
        -n 48 -j 1
    ```

    ANTs output is converted to FSL format using `c3d_affine_tool` and `convertwarp`.

### 7. Brain Extraction Using Inverse Warp

The MNI brain mask is warped back to native T1 space using the inverse of the non-linear warp (RegType 2/3) or the linear transform inverse (RegType 1):

```bash
applywarp --rel --interp=trilinear \
    --in=MNI152_T1_1mm_brain_mask \
    --ref=T1 \
    -w T1_to_MNI_nonlin_coeff_inv \
    -o T1_brain_mask
fslmaths T1 -mul T1_brain_mask T1_brain
```

### 8. De-Facing *(optional)*

```bash
# Warp MNI facemask to native T1 space
convert_xfm -omat grot.mat -concat MNI_to_MNI_BigFoV_facemask.mat T1_to_MNI_linear.mat
convert_xfm -omat grot.mat -inverse grot.mat
flirt -in MNI152_T1_1mm_BigFoV_facemask -ref T1 -out grot -applyxfm -init grot.mat
# Zero-out face voxels
fslmaths grot -binv -mul T1 T1
```

A QC metric is computed: number of voxels where the facemask overlaps the brain mask.

### 9. Bias Field Estimation & Correction

```bash
fast -b -o FAST/T1_brain T1_brain
fslmaths T1 -div FAST/T1_brain_bias T1_unbiased
fslmaths T1_brain -div FAST/T1_brain_bias T1_unbiased_brain
```

FSL FAST estimates the bias field (`-b` flag). The field is divided out to produce bias-corrected volumes.

### 10. Tissue Segmentation *(optional)*

```bash
fslmaths FAST/T1_brain_pve_0 -thr 0.5 -bin FAST/T1_brain_CSF_mask
fslmaths FAST/T1_brain_pve_1 -thr 0.5 -bin FAST/T1_brain_GM_mask
fslmaths FAST/T1_brain_pve_2 -thr 0.5 -bin FAST/T1_brain_WM_mask
```

Thresholds the FAST partial volume estimates at 0.5 to produce binary tissue masks. Skipped when `--dotissueseg=no`.

### 11. Apply Registration to Bias-Corrected Volumes

Linear and (if applicable) non-linear warps are applied to `T1_unbiased` and `T1_unbiased_brain` to produce images in MNI space.

### 12. Subcortical Segmentation *(optional)*

```bash
run_first_all -i T1_unbiased_brain -b -o FIRST/T1_first
```

FSL FIRST segments 15 subcortical structures. Skipped when `--dosubseg=no`.

### 13. SIENAX Brain Volume Estimation *(RegType 2 or 3 only)*

Calls [`run_T1_sienax.sh`](t1-sienax.md) to produce normalised brain volume estimates.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--workingdir` | Path | T1 temporary working directory |
| `--t1input` | File path | Path to `T1_orig.nii.gz` |
| `--dosubseg` | yes/no | Enable FSL FIRST subcortical segmentation |
| `--dotissueseg` | yes/no | Enable FAST binary tissue masks |
| `--docrop` | yes/no | Enable FOV cropping |
| `--dodefacing` | yes/no | Enable de-facing |
| `--fastfolder` | Path | FSL FAST output directory |
| `--firstfolder` | Path | FSL FIRST output directory |
| `--sienaxtempfolder` | Path | SIENAX temporary directory |
| `--regtempt1folder` | Path | Registration outputs directory |
| `--regtype` | 1/2/3 | Registration strategy |
| `--fbet` | Float 0–1 | BET fractional intensity threshold (default 0.5) |
| `--logfile` | Path | Log file path |

---

## Key Outputs

| File | Description |
|------|-------------|
| `T1.nii.gz` | Cropped, reoriented T1 |
| `T1_brain.nii.gz` | Skull-stripped T1 |
| `T1_brain_mask.nii.gz` | Binary brain mask |
| `T1_unbiased.nii.gz` | Bias-corrected whole-head T1 |
| `T1_unbiased_brain.nii.gz` | Bias-corrected skull-stripped T1 |
| `T1_orig_defaced.nii.gz` | De-faced original T1 *(if defacing enabled)* |
| `reg/T1_to_MNI_linear.mat` | T1 native → MNI affine matrix |
| `reg/T1_to_MNI_nonlin_field.nii.gz` | Non-linear warp field T1 → MNI |
| `reg/T1_to_MNI_nonlin_coeff_inv.nii.gz` | Inverse warp: MNI → T1 |
| `FAST/T1_brain_bias.nii.gz` | Estimated bias field |
| `FAST/T1_brain_pve_{0,1,2}.nii.gz` | CSF / GM / WM partial volume estimates |
| `FAST/T1_brain_{CSF,GM,WM}_mask.nii.gz` | Binary tissue masks |
