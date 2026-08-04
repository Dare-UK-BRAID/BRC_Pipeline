# IDP Extraction Pipeline

The **BRC IDP Extraction Pipeline** (`BRC_IDP_extraction`) extracts a standardised set of imaging-derived phenotypes (IDPs) from pre-processed structural and diffusion MRI data for a cohort of subjects, producing a group-level IDP table suitable for statistical analysis.

---

## Overview

IDPs are summary scalar measurements derived from neuroimaging data — for example, subcortical structure volumes, tissue fractions, white matter skeleton metrics, and cortical thickness estimates from FreeSurfer. The pipeline iterates over a list of subjects, computes IDPs for each, and assembles a group-level matrix.

This pipeline requires completed runs of `BRC_structural_pipeline` (and optionally `BRC_diffusion_pipeline`) for each subject in the cohort.

---

## Quick Start

```bash
export BRC_GLOBAL_SCR=/opt/brc/global/scripts
export BRC_IDPEXTRACT_SCR=/opt/brc/BRC_IDP_extraction/scripts
export FSLDIR=/usr/local/fsl
source ${FSLDIR}/etc/fslconf/fsl.sh

idp_extract.sh \
    --in    /data/study/subject_list.txt \
    --indir /data/study \
    --outdir /data/study/group_IDPs
```

Where `subject_list.txt` is a single-column text file of subject IDs, one per line.

---

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--in` | Yes | Path to subject ID list (one ID per line) |
| `--indir` | Yes | Root directory containing one pre-processed folder per subject |
| `--outdir` | Yes | Output directory for the group IDP files |

---

## Processing Steps

### 1. Setup and Logging

The pipeline creates the output folder structure:

```
<outdir>/
  Group_IDP/
    list/          # Subject ID list copies
    IDP_files/     # Per-subject IDP files
    IDPs.txt       # Group-level concatenated IDP matrix
    log/log.txt    # Processing log
```

### 2. Per-Subject IDP Extraction (`idp_extract_part_1.sh`)

For each subject in the list:

1. Reads the IDP definition file at `${BRC_GLOBAL_DIR}/config/IDP_list.txt`, which specifies IDP names and the extraction script responsible for each
2. For each IDP entry, calls the corresponding extraction script (e.g. `brc_IDP_T1_SIENAX.sh`) or reads a cached result if already present in the subject's IDP folder
3. Calls `brc_FS_get_IDPs.py` to extract FreeSurfer-derived IDPs (cortical thickness, surface area, subcortical volumes from `recon-all` output)
4. Concatenates all IDP values into a single row and appends it to `IDPs.txt`

### 3. Group Matrix Assembly

After all subjects are processed, `IDPs.txt` contains one row per subject with all IDP values, suitable for loading into R, Python, or MATLAB for statistical analysis.

---

## Individual IDP Extraction Scripts

Each script outputs a single scalar value or set of values to a text file within the subject's IDP folder.

| Script | IDP Category | Description |
|--------|-------------|-------------|
| `brc_IDP_T1_SIENAX.sh` | Brain volume | VSCALING, total GM, WM, and brain volumes from SIENAX |
| `brc_IDP_T1_FIRST_vols.sh` | Subcortical structures | FIRST-derived volumes for hippocampus, amygdala, caudate, putamen, thalamus, etc. |
| `brc_IDP_T1_GM_parcellation.sh` | Cortical/subcortical parcellation | GM volume per ROI from atlas-based parcellation |
| `brc_IDP_T1_align_to_std.sh` | Registration quality | Metrics derived from T1-to-standard space alignment |
| `brc_IDP_T1_noise_ratio.sh` | Image quality | Signal-to-noise ratio estimate from T1 data |
| `brc_IDP_T2_FLAIR_WMH.sh` | White matter hyperintensities | Total WMH volume from BIANCA lesion mask |
| `brc_IDP_all_align_to_T1.sh` | Cross-modal alignment | Alignment quality metrics for all modalities relative to T1 |
| `brc_IDP_diff_TBSS.sh` | Diffusion (TBSS) | Mean FA, MD, and other DTI metrics within the white matter skeleton |
| `brc_IDP_diff_eddy_outliers.sh` | Diffusion QC | Eddy-estimated outlier slice statistics |
| `brc_FS_get_IDPs.py` | FreeSurfer | Cortical thickness, surface area, and subcortical volumes from `recon-all` |

---

## IDP List Configuration

The IDP definitions are driven by `${BRC_GLOBAL_DIR}/config/IDP_list.txt`. Each row specifies:

- IDP name
- IDP category
- Extraction script name

This configuration file controls which IDPs are computed and in what order they appear in the output matrix. Modifying this file allows customisation of the IDP set without editing pipeline scripts.

---

## Dependencies

| Tool | Used For |
|------|----------|
| FSL (`fslstats`, `fslmaths`) | Volume extraction, masking |
| FreeSurfer | Cortical surface and subcortical volume IDPs |
| Python 3 | `brc_FS_get_IDPs.py` FreeSurfer IDP parsing |
| Structural pipeline outputs | T1 processed images, FAST segmentations, SIENAX, BIANCA, FIRST |
| Diffusion pipeline outputs | TBSS skeleton metrics, eddy QC |
| `BRC_GLOBAL_SCR` | Logging |
| `BRC_IDPEXTRACT_SCR` | Extraction scripts |

---

## Key Output Files

All paths are relative to `<outdir>/Group_IDP/`.

| File | Description |
|------|-------------|
| `IDPs.txt` | Group-level IDP matrix — one row per subject, one column per IDP |
| `list/Subject_list.txt` | Copy of the processed subject list |
| `IDP_files/<subject>/` | Per-subject IDP intermediate files |
| `log/log.txt` | Full processing log |

---

## Per-Subject IDP Files

Within each subject's folder at `<indir>/<subject>/analysis/IDP/`:

| File | Content |
|------|---------|
| `brc_IDP_T1_SIENAX.txt` | Brain volume estimates |
| `brc_IDP_T1_FIRST_vols.txt` | Subcortical structure volumes |
| `brc_IDP_T2_FLAIR_WMH.txt` | WMH total lesion volume |
| `brc_IDP_diff_TBSS.txt` | TBSS skeleton DTI metrics |
| `IDPs.txt` | All IDPs concatenated for this subject |

---

## Script Reference

| Script | Purpose |
|--------|---------|
| `idp_extract.sh` | Entry point — argument parsing, directory setup, orchestration |
| `idp_extract_part_1.sh` | Per-subject IDP extraction loop |
| `brc_IDP_T1_SIENAX.sh` | Brain volume IDPs from SIENAX |
| `brc_IDP_T1_FIRST_vols.sh` | Subcortical volume IDPs from FIRST |
| `brc_IDP_T1_GM_parcellation.sh` | GM parcellation volume IDPs |
| `brc_IDP_T1_align_to_std.sh` | T1 registration quality IDPs |
| `brc_IDP_T1_noise_ratio.sh` | T1 image quality (SNR) IDPs |
| `brc_IDP_T2_FLAIR_WMH.sh` | White matter hyperintensity volume IDPs |
| `brc_IDP_all_align_to_T1.sh` | Cross-modal alignment quality IDPs |
| `brc_IDP_diff_TBSS.sh` | TBSS-based diffusion IDPs |
| `brc_IDP_diff_eddy_outliers.sh` | Eddy QC outlier IDPs |
| `brc_FS_get_IDPs.py` | FreeSurfer cortical/subcortical IDPs |
