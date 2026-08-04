# Pipeline Overview

!!! info "Target Audience"
    Neuroimaging researchers and pipeline administrators.

This page describes the high-level architecture and processing flow of the BRC Structural Pipeline.

---

## Architecture

The pipeline is orchestrated by a single top-level entry script (`struc_preproc.sh`) that dispatches work to three sequential processing parts. Each part can be run locally (sequentially) or submitted as an independent cluster job when `CLUSTER_MODE=YES`.

```
struc_preproc.sh
  ├── struc_preproc_part_1.sh      # T1 + T2 preprocessing
  │     ├── run_T1_preprocessing.sh
  │     │     └── run_T1_sienax.sh
  │     └── run_T2_preprocessing.sh
  │           └── run_T2_bianca.sh
  ├── struc_preproc_part_2.sh      # FreeSurfer (optional)
  │     └── recon-all
  └── struc_preproc_part_3.sh      # FastSurfer + output organisation
        ├── run_fastsurfer.sh
        └── output_organization.sh
```

---

## Processing Flow

### Part 1 — Preprocessing

All core image processing happens in Part 1. For the T1 image this includes:

1. Reorientation to standard orientation (`fslreorient2std`)
2. Field-of-view cropping (`robustfov`)
3. Initial brain extraction (FSL BET, recursive)
4. FOV reduction (`standard_space_roi`)
5. Linear registration to MNI152 1mm (FLIRT, 12 DOF)
6. Non-linear registration to MNI152 (FNIRT or ANTs SyN, depending on `--regtype`)
7. Atlas-based brain mask extraction
8. De-facing (MNI facemask warped to native space)
9. Bias field estimation and correction (FSL FAST)
10. Tissue-type segmentation (CSF / GM / WM partial volume maps)
11. Subcortical segmentation (FSL FIRST, optional)
12. Brain volume estimation (SIENAX)

For the T2 image (if provided), Part 1 also:

1. Co-registers T2 to T1 space
2. Applies the T1 brain mask and bias field
3. Computes T2-to-MNI registration
4. Applies de-facing
5. Runs BIANCA white matter lesion detection

### Part 2 — FreeSurfer (optional)

Runs `recon-all -all` for full cortical parcellation and surface reconstruction. When a T2-FLAIR image is provided, it is passed to `recon-all` via the `-FLAIR` flag for improved cortical surface accuracy.

### Part 3 — FastSurfer + Output Organisation

Optionally runs FastSurfer for rapid DNN-based cortical reconstruction. Regardless of the surface option chosen, Part 3 always calls `output_organization.sh` to move all temporary outputs into the canonical BRC folder hierarchy and optionally runs multi-channel tissue segmentation (FSL FAST joint T1+T2).

---

## Registration Strategies

The pipeline supports three registration modes selected via `--regtype`:

| `--regtype` | Method | Notes |
|-------------|--------|-------|
| `1` | Linear only (FLIRT 12 DOF) | Fastest; brain mask from BET propagated via linear transform |
| `2` *(default)* | Linear + FNIRT non-linear | Recommended; brain mask from warped MNI atlas; requires ~12 GB RAM |
| `3` | Linear + ANTs SyNQuick | Highest accuracy; requires ANTs and c3d; slow without multi-core |

---

## Cluster Mode

When `CLUSTER_MODE=YES`, the three parts are submitted as dependent cluster jobs via `jobsub`:

- **Job 1** (Part 1): T1+T2 preprocessing. Time limit: 12 h (RegType 2) or 5 h (RegType 1 or 3).
- **Job 2** (Part 2, FreeSurfer): runs in parallel with Job 1. Time limit: 24 h.
- **Job 3** (Part 3): waits on Job 1 (and Job 2 if FreeSurfer enabled). Time limit: 7.5 h (FastSurfer) or 15 min (output organisation only).

ANTs (RegType 3) uses 24 CPU cores in cluster mode; all other modes use 1 core.
