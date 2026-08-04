# run_preprocessing.sh — FreeSurfer-Based Intensity Normalisation

**Location:** `BRC_structural_pipeline/scripts/run_preprocessing.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

---

## Purpose

An alternative T1 intensity normalisation and brain extraction path that leverages **FreeSurfer's `autorecon1`** step. Rather than relying solely on FSL FAST for bias correction, this script uses FreeSurfer's more robust pipeline (NU-correction and watershed-based skull stripping), converts the results back to NIfTI, registers them to native T1 space, and exports the output as `T1_brain_norm`.

This is used in the legacy `--anatbasedFS` path within `FSL_anat.sh`.

---

## Processing Steps

### 1. Run FreeSurfer autorecon1

```bash
recon-all -i T1input -s FreeSurfer -autorecon1
```

Performs NU-intensity correction and talairach registration. Produces:
- `mri/T1.mgz` — intensity-normalised T1
- `mri/brainmask.mgz` — brain mask

### 2. Convert to NIfTI

```bash
mri_convert -it mgz -ot nii mri/T1.mgz mri/T1_FS.nii.gz
mri_convert -it mgz -ot nii mri/brainmask.mgz mri/brainmask_FS.nii.gz
```

### 3. Register Back to Native T1 Space

```bash
flirt -ref T1input -in mri/T1_FS.nii.gz \
      -omat mri/rigid_manToFs.mat \
      -dof 12 -cost normmi -searchcost normmi

flirt -ref T1input -in mri/brainmask_FS.nii.gz \
      -out mri/brainmask.nii.gz \
      -init mri/rigid_manToFs.mat -applyxfm
```

Uses 12-DOF normalised mutual information registration to align FreeSurfer output back to native T1 coordinates.

### 4. Reorient to Standard

```bash
fslreorient2std mri/brainmask > mri/brainmask_orig2std.mat
fslreorient2std mri/brainmask mri/brainmask
```

### 5. Export Brain Norm

```bash
fslmaths mri/brainmask OutNormFolder/T1_brain_norm
```

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--workingdir` | Path | `processedT1Folder` — also used as FreeSurfer `SUBJECTS_DIR` |
| `--t1input` | File path | Raw T1 input image passed to `recon-all -i` |
| `--fsfoldername` | String | FreeSurfer subject name (`-s` flag) |
| `--outnorm` | Path | Output directory for `T1_brain_norm` |
| `--logfile` | Path | Log file path |

---

## Key Outputs

| File | Description |
|------|-------------|
| `T1_brain_norm.nii.gz` | FreeSurfer-normalised brain mask, registered to native T1 space and reoriented |
| `<SUBJECTS_DIR>/FreeSurfer/mri/` | Full FreeSurfer `autorecon1` output (`T1.mgz`, `brainmask.mgz`, etc.) |
