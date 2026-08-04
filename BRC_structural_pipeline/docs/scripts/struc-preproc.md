# struc_preproc.sh — Pipeline Entry Point

!!! info "Target Audience"
    Neuroimaging researchers running the pipeline.

**Location:** `BRC_structural_pipeline/struc_preproc.sh`  
**Last updated:** 19 May 2019

---

## Purpose

This is the top-level entry point for the entire structural pipeline. It:

1. Parses and validates all user-facing command-line arguments
2. Constructs the full output directory hierarchy
3. Initialises the log file and records software versions
4. Copies raw input images to the `raw/` folder
5. Dispatches the three processing parts — either as sequential local calls or as dependent cluster jobs (`CLUSTER_MODE=YES`)

---

## Usage

```bash
struc_preproc.sh \
  --input <T1.nii.gz> \
  --path <output_dir> \
  --subject <SubjectID> \
  [OPTIONS]
```

---

## Compulsory Arguments

| Argument | Description |
|----------|-------------|
| `--input <path>` | Full path to the T1-weighted input NIfTI image |
| `--path <path>` | Root output directory — a subject sub-folder is created inside |
| `--subject <name>` | Subject identifier used to name the output sub-folder |

---

## Optional Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--t2 <path>` | — | Full path to T2-weighted (FLAIR) image. Enables T2 processing. |
| `--freesurfer` | off | Enable FreeSurfer `recon-all` cortical reconstruction (Part 2) |
| `--fastsurfer` | off | Enable FastSurfer cortical reconstruction (Part 3) |
| `--subseg` | off | Enable subcortical segmentation via FSL FIRST |
| `--qc` | off | Enable quality control of T1 data |
| `--noreg` | off | Disable registration steps (FLIRT and FNIRT) |
| `--noseg` | off | Disable tissue-type segmentation (FSL FAST) |
| `--nocrop` | off | Disable automated field-of-view cropping |
| `--nodefacing` | off | Disable brain defacing |
| `--regtype <1\|2\|3>` | `2` | Registration strategy: `1`=linear only, `2`=linear+FNIRT, `3`=linear+ANTs |
| `--t2lesion <path>` | — | Pre-existing lesion mask in T2 native space. Requires `--t2`. |
| `--fbet <0–1>` | `0.5` | Fractional intensity threshold for FSL BET initialisation |

---

## Data Flow

```
[struc_preproc.sh]
  │
  ├─ Validate arguments
  ├─ Create directory tree under <path>/<subject>/
  ├─ Initialise log file
  ├─ Copy T1_orig.nii.gz (and T2_orig.nii.gz) to raw/
  │
  ├─ [LOCAL MODE]
  │     struc_preproc_part_1.sh → struc_preproc_part_2.sh → struc_preproc_part_3.sh
  │
  └─ [CLUSTER MODE]
        Job 1 (Part 1) → Job 2 (Part 2, parallel if FreeSurfer) → Job 3 (Part 3, waits on Job 1+2)
```

---

## Cluster Mode Resource Allocation

| Job | Time Limit | Memory | Cores |
|-----|-----------|--------|-------|
| Part 1 (RegType 2) | 12:00:00 | 100 GB | 1 |
| Part 1 (RegType 1 or 3) | 05:00:00 | 100 GB | 1 (or 24 for ANTs) |
| Part 2 (FreeSurfer) | 24:00:00 | 60 GB | 1 |
| Part 3 (FastSurfer) | 07:30:00 | 60 GB | 1 |
| Part 3 (output only) | 00:15:00 | 10 GB | 1 |

---

## Directory Naming Conventions

The script defines all directory names as variables before creating the structure. Key folder names:

| Variable | Value |
|----------|-------|
| `AnalysisFolderName` | `analysis` |
| `AnatMRIFolderName` | `anatMRI` |
| `T1FolderName` | `T1` |
| `preprocessFolderName` | `preproc` |
| `processedFolderName` | `processed` |
| `FSFolderName` | `FreeSurfer` |
| `FastSurferFolderName` | `FastSurfer` |
| `BiancaFolderName` | `lesions` |
| `SienaxFolderName` | `SIENAX` |
