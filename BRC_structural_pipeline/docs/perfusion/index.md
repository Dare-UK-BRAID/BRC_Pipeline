# Perfusion (ASL) Pipeline

The **BRC Perfusion Pipeline** (`BRC_perfusion_pipeline`) processes single-TI pseudo-continuous arterial spin labelling (pCASL) data to produce quantified cerebral blood flow (CBF) maps in both native and standard space, with optional partial volume correction.

---

## Overview

The pipeline takes a difference (perfusion-weighted) image and a calibration (M0) image as inputs. It performs CBF quantification using a standard kinetic model, registers the ASL data to the subject's T1 anatomical image (using boundary-based registration leveraging the white matter segmentation), propagates the result to MNI152 standard space, organises outputs, and optionally applies multi-tissue-linear-series (MLTS) partial volume correction.

---

## Quick Start

```bash
export BRC_GLOBAL_SCR=/opt/brc/global/scripts
export BRC_PMRI_SCR=/opt/brc/BRC_perfusion_pipeline/scripts
export FSLDIR=/usr/local/fsl
source ${FSLDIR}/etc/fslconf/fsl.sh

ASL_preproc.sh \
    --input_pwi  /data/sub-001/perf/sub-001_pwi.nii.gz \
    --input_m0   /data/sub-001/perf/sub-001_m0.nii.gz \
    --path       /data/study \
    --subject    sub-001 \
    --tr 3.2 \
    --ti 3.0 \
    --bolus 1.8 \
    --pvcmethod MLTS
```

!!! note "Structural prerequisite"
    The perfusion pipeline requires a completed structural pipeline run for the same subject. It uses the T1 brain, white matter mask, and T1→MNI registration produced by `BRC_structural_pipeline`.

---

## Arguments

### Compulsory

| Argument | Description |
|----------|-------------|
| `--input_pwi` | Full path to the difference (perfusion-weighted) image |
| `--input_m0` | Full path to the calibration (M0) image |
| `--path` | Output directory (absolute path) |
| `--subject` | Subject identifier |

### Optional

| Argument | Default | Description |
|----------|---------|-------------|
| `--tr` | `3.2` | Repetition time of calibration data (seconds) |
| `--ti` | `3.0` | Inversion time (seconds) |
| `--bolus` | `1` | Labelling (bolus) duration (seconds) |
| `--cgain` | `1` | Relative gain between calibration and ASL image |
| `--pvcmethod` | `NONE` | Partial volume correction method: `MLTS` or `NONE` |
| `--name` | `aslMRI` | Output folder name |

---

## Processing Steps

### 1. CBF Quantification (`Image_Quantification.sh`)

Applies the standard single-compartment kinetic model to the difference image, using TR, TI, bolus duration, and calibration gain to compute absolute CBF in ml/100g/min. The M0 image is used to scale the signal to physiological units.

### 2. ASL-to-T1 Registration (`ASL_2_T1_Registration.sh`)

Registers the mean ASL image to the T1 brain using boundary-based registration (FSL `epi_reg` style), leveraging the white matter partial volume estimate and WM mask from the structural pipeline for optimal cortical boundary alignment. The registration uses a configurable number of degrees of freedom (DOF) and super-sampling level.

### 3. Apply Registration (`Apply_Registration.sh`)

Propagates the CBF map and ASL data to T1 native space and MNI152 standard space by applying the ASL→T1 affine and the pre-computed T1→MNI warp field in a single `applywarp` call.

### 4. Partial Volume Correction (`Partial_Volume_Correction.sh`)

When `--pvcmethod MLTS` is specified, applies the multi-tissue linear series (MLTS) method (`mlts_partial_volume_correction.py`) using grey matter (GM) and white matter (WM) partial volume estimates from FAST. This corrects for the overestimation of CBF in voxels with mixed tissue types, improving spatial specificity.

### 5. Data Organisation (`Data_Organization.sh`)

Moves processed files into the canonical BRC output hierarchy under `<subject>/analysis/aslMRI/`.

---

## Kinetic Model Parameters

The quantification model assumes single-TI pCASL and applies:

| Parameter | Symbol | Default |
|-----------|--------|---------|
| Repetition time | TR | 3.2 s |
| Inversion time | TI | 3.0 s |
| Bolus duration | τ | 1.0 s |
| Calibration gain | M | 1 |

Modify these via the corresponding command-line arguments to match your acquisition protocol.

---

## Dependencies

| Tool | Used For |
|------|----------|
| FSL (`flirt`, `applywarp`, `fslmaths`) | Registration, warp application |
| Python 3 | MLTS partial volume correction |
| Structural pipeline outputs | T1 brain, WM mask, T1→MNI registration |
| `BRC_GLOBAL_SCR` | Logging |
| `BRC_PMRI_SCR` | All pipeline sub-scripts |

---

## Key Output Files

All paths are relative to `<path>/<subject>/analysis/aslMRI/`.

| File | Description |
|------|-------------|
| `processed/CBF.nii.gz` | Quantified CBF map in ASL native space |
| `processed/CBF_2_T1.nii.gz` | CBF map registered to T1 native space |
| `processed/CBF_2_std.nii.gz` | CBF map in MNI152 standard space |
| `processed/CBF_PVC.nii.gz` | Partial volume corrected CBF (if MLTS) |
| `reg/asl2str.mat` | ASL → T1 affine transform |
| `reg/asl2std_warp.nii.gz` | ASL → MNI warp field |
| `log/log.txt` | Full processing log |

---

## Script Reference

| Script | Purpose |
|--------|---------|
| `ASL_preproc.sh` | Entry point — argument parsing and orchestration |
| `aslMRI_preproc_part_1.sh` | Processing dispatcher |
| `Image_Quantification.sh` | Kinetic model CBF quantification |
| `ASL_2_T1_Registration.sh` | ASL-to-T1 boundary-based registration |
| `Apply_Registration.sh` | Apply affine + warp to propagate CBF maps |
| `Partial_Volume_Correction.sh` | PVC wrapper (MLTS method) |
| `mlts_partial_volume_correction.py` | MLTS partial volume correction implementation |
| `Data_Organization.sh` | Move outputs into BRC folder hierarchy |
