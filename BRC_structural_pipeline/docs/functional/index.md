# Functional MRI Pipeline

The **BRC Functional MRI Pipeline** (`BRC_functional_pipeline`) preprocesses resting-state and task-based fMRI data from raw NIFTI through distortion correction, motion correction, registration to T1 and standard space, ICA-AROMA noise removal, temporal filtering, and spatial smoothing.

---

## Overview

The pipeline supports multiple susceptibility distortion correction strategies (TOPUP spin-echo, Siemens/GE field maps, or none), flexible slice-timing correction via FSL or SPM, and modular physiological noise removal. The default output is a cleaned, registered 4D fMRI volume in standard space, ready for group-level analysis.

---

## Pipeline Stages

| Stage | Script | Description |
|-------|--------|-------------|
| **Part 1** | `fMRI_preproc_part_1.sh` | Gradient distortion correction, EPI distortion correction, spin-echo bias field |
| **Part 2** | `fMRI_preproc_part_2.sh` | Motion correction, slice timing correction, EPI-to-T1 registration, one-step resampling |
| **Part 3** | `fMRI_preproc_part_3.sh` | Intensity normalisation, ICA-AROMA, physiological noise removal, temporal filtering, spatial smoothing, QC |

---

## Quick Start

```bash
export BRC_GLOBAL_SCR=/opt/brc/global/scripts
export BRC_FMRI_SCR=/opt/brc/BRC_functional_pipeline/scripts
export FSLDIR=/usr/local/fsl
source ${FSLDIR}/etc/fslconf/fsl.sh

fMRI_preproc.sh \
    --input   /data/sub-001/func/sub-001_task-rest_bold.nii.gz \
    --path    /data/study \
    --subject sub-001 \
    --dcmethod TOPUP \
    --SEPhaseNeg /data/sub-001/fmap/se_AP.nii.gz \
    --SEPhasePos /data/sub-001/fmap/se_PA.nii.gz \
    --echospacing 0.00058 \
    --unwarpdir y- \
    --fmrires 2
```

---

## Compulsory Arguments

| Argument | Description |
|----------|-------------|
| `--input` | Full path to the 4D fMRI NIFTI file |
| `--path` | Output directory (absolute path) |
| `--subject` | Subject identifier |

---

## Optional Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--mctype` | `MCFLIRT6` | Motion correction: `MCFLIRT6` (6 DOF), `MCFLIRT12` (12 DOF), or `EDDY` |
| `--dcmethod` | `NONE` | Distortion correction: `TOPUP`, `SiemensFieldMap`, `GeneralElectricFieldMap`, `NONE` |
| `--fmriscout` | `NONE` | Single-band reference (SBRef) image |
| `--slice2vol` | off | Slice-to-volume motion correction |
| `--slspec` | — | JSON slice specification file |
| `--SEPhaseNeg` | `NONE` | Spin-echo field map — negative PE direction |
| `--SEPhasePos` | `NONE` | Spin-echo field map — positive PE direction |
| `--fmapmag` | `NONE` | 4D magnitude image for Siemens-style field map |
| `--fmapphase` | `NONE` | 3D phase difference for Siemens-style field map |
| `--echodiff` | `NONE` | Echo time difference for Siemens field map |
| `--echospacing` | — | Effective echo spacing of SE field map (seconds) |
| `--echospacing_fMRI` | `0.0` | Echo spacing of fMRI data (seconds), if different from SE field map |
| `--unwarpdir` | — | Unwarp direction: `y` (PA), `y-` (AP), `x` (RL), `x-` (LR) |
| `--biascorrection` | `NONE` | Bias correction: `NONE` or `SEBASED` (spin-echo based; requires TOPUP) |
| `--intensitynorm` | off | Grand-mean intensity normalisation |
| `--stcmethod` | `0` | Slice timing correction method (0=none, 1–8; see below) |
| `--slstiming` | — | Custom slice order/timing file |
| `--fwhm` | `0` | Spatial smoothing sigma (mm); 0 = no smoothing |
| `--noaroma` | off | Disable ICA-AROMA noise removal |
| `--fmrires` | `2` | Target output resolution (mm) |
| `--tempfilter` | `0` | High-pass temporal filter cutoff (seconds); 0 = no filtering |
| `--name` | `rfMRI` | Output folder name |
| `--noqc` | off | Disable QC analysis |
| `--clean` | off | Delete all intermediate files after completion |
| `--delvolumes` | `0` | Number of volumes to remove from the start of the timeseries |
| `--physin` | — | Physiological data file (text format) |
| `--samplingrate` | `100` | Physiological data sampling rate (Hz) |

---

## Slice Timing Correction Methods

| Code | Method | Description |
|------|--------|-------------|
| `0` | None | No slice timing correction (default) |
| `1` | SPM | Interleaved acquisition (0, 2, 4 … 1, 3, 5) |
| `2` | SPM | Forward acquisition (0, 1, 2, …) |
| `3` | SPM | Backward acquisition (n, n-1, n-2, …) |
| `4` | FSL | Bottom-to-top |
| `5` | FSL | Top-to-bottom |
| `6` | FSL | Interleaved |
| `7` | FSL | Custom slice order file (`--slstiming`) |
| `8` | FSL | Custom slice timings file (`--slstiming`) |

---

## Processing Steps

### Part 1 — Distortion Correction

1. **Gradient distortion correction**: applies gradient nonlinearity correction if coefficients are supplied; otherwise creates identity warp fields for pipeline compatibility
2. **EPI distortion correction** (`EPI_Distortion_Correction.sh`): runs FSL `topup` on spin-echo pair or prepares Siemens/GE field maps
3. **Spin-echo bias field** (`Compute_SpinEcho_BiasField.sh`): estimates receive-coil bias from spin-echo images (only when `biascorrection=SEBASED`)
4. **Parameter file generation** (`Generate_Parameter_File.sh`): writes acquisition parameter files required by eddy and topup

### Part 2 — Motion Correction and Registration

5. **Volume deletion** (`EddyPreprocessing.sh`): removes leading volumes specified by `--delvolumes`
6. **Motion correction** (`MotionCorrection.sh`): `mcflirt` (6 or 12 DOF, calls `mcflirt.sh`) or `eddy`-based within/between-volume correction (`eddy_cuda.sh`)
7. **Slice timing correction** (`Slice_Timing_Correction.sh`): FSL `slicetimer` or SPM-based correction via `run_spm_slice_time_correction.m`
8. **EPI-to-T1 registration** (`EPI_2_T1_Registration.sh`): boundary-based registration using `epi_reg_dof.sh`; concatenates with T1→MNI warp for single-step resampling
9. **One-step resampling** (`One_Step_Resampling.sh`): applies the full distortion + motion + registration warp in a single `applywarp` call to avoid cascading interpolation artefacts

### Part 3 — Cleaning and QC

10. **Intensity normalisation** (`Intensity_Normalization.sh`): scales 4D data to a grand-mean intensity of 10,000 (when `--intensitynorm` is set)
11. **ICA-AROMA** (`ICA_AROMA/ICA_AROMA.py`): identifies and removes motion-related artefact components using a pre-trained classifier; produces a non-aggressively denoised output
12. **Physiological noise removal** (`Physiological_Noise_Removal.sh`): models and removes cardiac and respiratory noise using FSL `fabber` / PNM if physiological traces are provided
13. **Temporal filtering** (`Temporal_Filtering.sh`): applies high-pass filter using FSL `bptf`
14. **Spatial smoothing** (`Spatial_Smoothing_Noise_Removal.sh`): Gaussian smoothing with specified FWHM
15. **Data organisation** (`Data_Organization.sh`): moves processed files into the final BRC output hierarchy
16. **QC analysis** (`QC_analysis.sh`, `run_QC_analysis.m`): generates motion parameters, tSNR maps, and summary statistics

---

## ICA-AROMA

ICA-AROMA is a Python-based tool that uses four spatial and temporal features to classify ICA components as signal or noise (motion-related). It includes:

- Edge fraction (proximity to brain edge)
- High-frequency content (power spectrum)
- Maximum real data correlation with CSF mask
- Maximum real data correlation with out-of-brain mask

The classifier was trained on manual component labels and operates without subject-specific thresholds. Non-aggressive denoising regresses only the noise time courses, preserving shared variance between signal and noise.

---

## Dependencies

| Tool | Used For |
|------|----------|
| FSL (`mcflirt`, `flirt`, `fnirt`, `applywarp`, `topup`, `slicetimer`) | Motion correction, registration, distortion correction |
| Python + scikit-learn | ICA-AROMA |
| MATLAB / SPM | Slice timing correction (SPM methods), QC analysis |
| `BRC_GLOBAL_SCR` | Logging |
| `BRC_FMRI_SCR` | All pipeline sub-scripts |

---

## Key Output Files

All paths are relative to `<path>/<subject>/analysis/rfMRI/` (or the name set by `--name`).

| File | Description |
|------|-------------|
| `processed/func2std.nii.gz` | Final cleaned fMRI in MNI152 standard space |
| `processed/func2str.nii.gz` | Final cleaned fMRI in T1 native space |
| `reg/func2str.mat` | EPI → T1 affine |
| `reg/func2std_warp.nii.gz` | EPI → MNI warp field |
| `ICA_testout.ica/` | Melodic ICA output (AROMA components) |
| `ICA_testout_denoised.nii.gz` | ICA-AROMA denoised 4D timeseries |
| `mc/prefiltered_func_data_mcf.par` | Motion parameters (6 DOF) |
| `QC/` | tSNR map, motion plots, summary statistics |
| `log/log.txt` | Full processing log |

---

## Script Reference

| Script | Purpose |
|--------|---------|
| `fMRI_preproc.sh` | Entry point — argument parsing and job dispatch |
| `fMRI_preproc_part_1.sh` | Part 1 dispatcher: GDC, EPI distortion, bias |
| `fMRI_preproc_part_2.sh` | Part 2 dispatcher: motion, STC, registration, resampling |
| `fMRI_preproc_part_3.sh` | Part 3 dispatcher: normalisation, AROMA, filtering, smoothing, QC |
| `Generate_Parameter_File.sh` | Write TOPUP/EDDY acquisition parameter files |
| `EPI_Distortion_Correction.sh` | TOPUP or fieldmap-based distortion correction |
| `Compute_SpinEcho_BiasField.sh` | SE-based receive-coil bias estimation |
| `EddyPreprocessing.sh` | Volume deletion and eddy current pre-steps |
| `MotionCorrection.sh` | MCFLIRT or EDDY motion correction |
| `mcflirt.sh` | FSL mcflirt wrapper |
| `eddy_cuda.sh` | GPU-accelerated eddy wrapper |
| `Slice_Timing_Correction.sh` | FSL/SPM slice timing correction |
| `EPI_2_T1_Registration.sh` | Boundary-based EPI-to-T1 registration |
| `epi_reg_dof.sh` | `epi_reg` wrapper with configurable DOF |
| `One_Step_Resampling.sh` | Combined warp application (distortion+motion+reg) |
| `Apply_Registration.sh` | Apply pre-computed registration transforms |
| `Intensity_Normalization.sh` | Grand-mean intensity scaling |
| `Physiological_Noise_Removal.sh` | Cardiac/respiratory noise modelling |
| `Temporal_Filtering.sh` | High-pass temporal filtering |
| `Spatial_Smoothing_Noise_Removal.sh` | Gaussian spatial smoothing |
| `Data_Organization.sh` | Move files into BRC output hierarchy |
| `QC_analysis.sh` | QC metrics and report generation |
| `ICA_AROMA/ICA_AROMA.py` | ICA-AROMA main classifier |
| `ICA_AROMA/ICA_AROMA_functions.py` | AROMA feature extraction functions |
