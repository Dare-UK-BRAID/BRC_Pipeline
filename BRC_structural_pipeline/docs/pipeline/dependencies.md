# External Dependencies

!!! info "Target Audience"
    System administrators and TRE operators installing the pipeline.

The pipeline requires several external tools to be installed and accessible on the system `PATH`. This page lists each dependency, what it is used for, and which pipeline flags require it.

---

## Required for All Runs

| Tool | Environment Variable | Used For |
|------|---------------------|----------|
| **FSL** | `FSLDIR` | BET, FAST, FLIRT, FNIRT, FIRST, BIANCA, SIENAX, image maths |
| **BRC Global Scripts** | `BRC_GLOBAL_SCR` | Logging (`log.shlib`), version tracking (`Show_version.sh`) |
| **BRC Pipeline Scripts** | `BRC_SCTRUC_SCR` | All structural sub-scripts |
| **BRC Global Data** | `BRC_GLOBAL_DIR` | Templates (MNI masks, facemask, FNIRT config), BIANCA classifier |

---

## Conditionally Required

| Tool | Required When | Environment Variable |
|------|--------------|----------------------|
| **FreeSurfer** (`recon-all`, `mri_convert`) | `--freesurfer` flag is set | Must be on `PATH` |
| **FastSurfer** (`run_fastsurfer.sh`) | `--fastsurfer` flag is set | Must be on `PATH` |
| **ANTs** (`antsRegistrationSyN.sh`, `antsApplyTransforms`) | `--regtype 3` | `ANTSPATH` |
| **c3d** (`c3d_affine_tool`) | `--regtype 3` (ITK→FSL transform conversion) | `C3DPATH` |
| **jobsub** | `CLUSTER_MODE=YES` | `JOBSUBpath` |

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `FSLDIR` | Always | FSL installation directory (e.g. `/usr/local/fsl`) |
| `BRC_GLOBAL_SCR` | Always | Path to BRC global utility scripts |
| `BRC_SCTRUC_SCR` | Always | Path to structural pipeline `scripts/` directory |
| `BRC_GLOBAL_DIR` | Always | Path to BRC global data (templates, configs, BIANCA classifier) |
| `BRCDIR` | Always | Root BRC directory (contains `Show_version.sh`) |
| `CLUSTER_MODE` | Optional | Set to `YES` to submit jobs to a cluster scheduler |
| `JOBSUBpath` | When `CLUSTER_MODE=YES` | Path to `jobsub` binary |
| `FAST_t` | Optional | FSL FAST image type flag — `1` (T1w), `2` (T2w), `3` (PD). Default: `1` |
| `ANTSPATH` | When `--regtype 3` | Path to ANTs binaries directory |
| `C3DPATH` | When `--regtype 3` | Path to c3d binaries directory |
| `RUN` | Optional | Command prefix for execution (leave blank for local runs) |
| `Subject` | Cluster mode | Subject identifier used to name cluster job IDs |

---

## BRC Templates

The `BRC_GLOBAL_DIR` must contain the following files used during processing:

| Template File | Used In | Purpose |
|---------------|---------|---------|
| `templates/MNI152_T1_1mm_brain_mask_dil_GD7` | FNIRT | Dilated MNI brain mask for FNIRT reference masking |
| `templates/MNI152_T1_1mm_BigFoV_facemask` | Defacing | Face region mask in MNI space |
| `templates/MNI_to_MNI_BigFoV_facemask.mat` | Defacing | Affine from MNI to BigFOV facemask space |
| `templates/bianca_class_data` | BIANCA | Pre-trained k-NN classifier for lesion detection |
| `config/bb_fnirt.cnf` | FNIRT | FNIRT configuration file |
