# Diffusion MRI Pipeline

The **BRC Diffusion MRI Pipeline** (`BRC_diffusion_pipeline`) processes single- and multi-shell diffusion-weighted imaging (DWI) data from raw NIFTI through distortion correction, eddy current correction, diffusion model fitting, and optional registration to standard space and group-level analysis (TBSS).

---

!!! note "Structural prerequisite"
    Registration steps require a completed [Structural Pipeline](../pipeline/overview.md) run to provide the T1 brain and T1→MNI transforms.


The pipeline is designed for data acquired with opposing phase-encode directions (LR/RL or AP/PA), enabling field-map-free susceptibility distortion correction via FSL TOPUP. It supports standard Stejskal-Tanner DTI, as well as advanced multi-shell models including NODDI, DKI/WMTI, and the ALPS diffusion perivascular space metric.

---

## Pipeline Stages

| Stage | Script | Description |
|-------|--------|-------------|
| **Part 1** | `dMRI_preproc_part_1.sh` | Data copy, basic preprocessing, TOPUP |
| **Part 2** | `dMRI_preproc_part_2.sh` | EDDY current/motion correction, QC |
| **Part 3** | `dMRI_preproc_part_3.sh` | Eddy post-processing, model fitting (DTI/NODDI/DKI), data organisation |
| **Part 4** | `dMRI_preproc_part_4.sh` | Registration to T1/standard space, TBSS, ALPS |

---

## Quick Start

```bash
export BRC_GLOBAL_SCR=/opt/brc/global/scripts
export BRC_DMRI_SCR=/opt/brc/BRC_diffusion_pipeline/scripts
export FSLDIR=/usr/local/fsl
source ${FSLDIR}/etc/fslconf/fsl.sh

dMRI_preproc.sh \
    --input  AP_run1@AP_run2 \
    --input_2 PA_run1@PA_run2 \
    --path   /data/study \
    --subject sub-001 \
    --echospacing 0.00058 \
    --pe_dir 2 \
    --reg \
    --tbss
```

---

## Compulsory Arguments

| Argument | Description |
|----------|-------------|
| `--input` | `@`-separated list of input DWI volumes (primary PE direction) |
| `--path` | Output directory (absolute path) |
| `--subject` | Subject identifier |
| `--echospacing` | Effective echo spacing in seconds |
| `--pe_dir` | Phase encoding direction: `1` = LR/RL, `2` = AP/PA |

---

## Optional Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--input_2` | `NONE` | Reverse PE direction volumes |
| `--slice2vol` | off | Slice-to-volume motion correction (FSL eddy) |
| `--slspec` | — | JSON slice specification file for slice-to-vol |
| `--movebysusceptibility` | off | Estimate susceptibility change with motion (FSL 6+) |
| `--cm_flag` | `2` | CombineMatchedFlag: `2`=all uncombined, `1`=combine matched pairs, `0`=include singles |
| `--p_im` | `1` | In-plane parallel imaging (GRAPPA) factor |
| `--hires` | off | Increase time/memory limits for high-resolution data |
| `--dtimaxshell` | `1500` | Maximum b-value shell for DTI model fitting |
| `--skip_preproc` | off | Skip preprocessing; run only downstream modules |
| `--mppca` | off | Marcenko-Pastur PCA denoising (MP-PCA) |
| `--unring` | off | Gibbs ringing removal (MRTrix) |
| `--use_topup` | — | Path to pre-computed TOPUP folder (skip TOPUP step) |
| `--qc` | off | Quality control of dMRI data |
| `--reg` | off | Registration to T1 and standard space |
| `--tbss` | off | Tract-Based Spatial Statistics |
| `--noddi` | off | NODDI model fitting (multi-shell required) |
| `--dki` | off | Diffusion Kurtosis Imaging model (multi-shell required) |
| `--wmti` | off | White Matter Tract Integrity model (multi-shell required) |
| `--alps` | off | Diffusion along perivascular spaces (ALPS) metric |

---

## Processing Steps

### Part 1 — Preprocessing and TOPUP

1. **Data copy** (`data_copy.sh`): validates and copies raw DWI volumes, bvals, and bvecs into the working directory
2. **Basic preprocessing** (`basic_preproc.sh`):
   - Computes total readout time from echo spacing and matrix size
   - Identifies and extracts b0 volumes (bval < `b0maxbval`, default 50)
   - Applies optional MP-PCA denoising (`dwidenoise`, MRTrix)
   - Applies optional Gibbs unringing (`mrdegibbs`, MRTrix)
   - Generates acquisition parameters file for TOPUP/EDDY
   - Handles LR/RL and AP/PA conventions separately
3. **TOPUP** (`run_topup.sh`): estimates susceptibility-induced field map from b0 pairs using FSL `topup`; a `--hires` flag switches to a finer configuration for sub-millimetre data

### Part 2 — EDDY Correction

4. **EDDY** (`run_eddy.sh`): corrects eddy-current distortions and inter-volume head motion using FSL `eddy_cuda` (GPU) or `eddy_openmp` (CPU fallback):
   - Slice-to-volume correction activated by `--slice2vol`
   - Move-by-susceptibility correction activated by `--movebysusceptibility`
   - Optional `eddy_squad` QC if `--qc` enabled
5. **EDDY Combine** (`eddy_combine.sh`): merges LR+RL (or AP+PA) corrected volumes, bvals, and bvecs

### Part 3 — Post-processing and Model Fitting

6. **Eddy post-processing** (`eddy_postproc.sh`):
   - Combines matched PE pairs according to `CombineMatchedFlag`
   - Runs `dtifit` for DTI (FA, MD, L1, L2, L3, MO, S0)
   - Optionally fits NODDI (`amico`), DKI, and WMTI models
7. **Shell extraction** (`extract_shells.py`): identifies unique b-value shells and assembles per-shell data for multi-shell models
8. **DKI fitting** (`run_DKI.py`): fits diffusion kurtosis model using DIPY

### Part 4 — Registration and Group Analysis

9. **Registration** (`diff_reg.sh`): registers dMRI data to T1 native space (`flirt`, boundary-based) then propagates to MNI152 standard space
10. **TBSS** (`run_tbss.sh`): tract-based spatial statistics across subjects (calls `tbss_step_1_preproc.sh` → `tbss_step_4_prestats.sh`)
11. **ALPS** (`run_alps.sh`): computes the ALPS index for perivascular space assessment from diffusion tensor projections along association fibres

---

## Dependencies

| Tool | Used For |
|------|----------|
| FSL (`topup`, `eddy`, `dtifit`, `flirt`, `fnirt`) | Distortion correction, model fitting, registration |
| MRTrix (`dwidenoise`, `mrdegibbs`) | Optional denoising and unringing |
| AMICO / NODDI toolbox | NODDI model fitting |
| DIPY | DKI model fitting |
| Python 3 | Shell extraction, DKI scripts |
| `BRC_GLOBAL_SCR` | Logging (`log.shlib`) |
| `BRC_DMRI_SCR` | All pipeline sub-scripts |

---

## Key Output Files

All outputs are relative to `<path>/<subject>/analysis/dMRI/`.

| File | Description |
|------|-------------|
| `data/data.nii.gz` | Eddy-corrected 4D DWI |
| `data/bvals`, `data/bvecs` | Corrected gradient table |
| `data/dti_FA.nii.gz` | Fractional anisotropy |
| `data/dti_MD.nii.gz` | Mean diffusivity |
| `data/dti_L1/L2/L3.nii.gz` | Principal eigenvalues |
| `data/dti_MO.nii.gz` | Mode of anisotropy |
| `reg/dMRI_2_T1.mat` | dMRI → T1 affine |
| `reg/dMRI_2_std_warp.nii.gz` | dMRI → MNI warp field |
| `data2std/dMRI_2_std.nii.gz` | FA in MNI space |
| `QC/eddy_squad/` | EDDY QC report |
| `TBSS/` | TBSS skeleton and results |
| `log/log.txt` | Full processing log |

---

## Script Reference

| Script | Purpose |
|--------|---------|
| `dMRI_preproc.sh` | Entry point — argument parsing and job dispatch |
| `dMRI_preproc_part_1.sh` | Dispatcher: data copy → basic preproc → TOPUP |
| `dMRI_preproc_part_2.sh` | Dispatcher: EDDY correction + QC |
| `dMRI_preproc_part_3.sh` | Dispatcher: eddy post-proc + model fitting |
| `dMRI_preproc_part_4.sh` | Dispatcher: registration + TBSS + ALPS |
| `data_copy.sh` | Raw data ingestion and validation |
| `basic_preproc.sh` | b0 identification, denoising, acq params |
| `run_topup.sh` | FSL TOPUP fieldmap estimation |
| `run_eddy.sh` | FSL EDDY current and motion correction |
| `eddy_combine.sh` | PE pair merging after eddy |
| `eddy_postproc.sh` | Post-eddy combination + DTI/NODDI/DKI |
| `diff_reg.sh` | dMRI-to-T1 and dMRI-to-standard registration |
| `run_tbss.sh` | TBSS group-level analysis coordinator |
| `tbss_step_1_preproc.sh` | TBSS preprocessing |
| `tbss_step_2_reg.sh` | TBSS registration |
| `tbss_step_3_postreg.sh` | TBSS post-registration |
| `tbss_step_4_prestats.sh` | TBSS pre-statistics |
| `tbss_non_FA.sh` | TBSS for non-FA maps |
| `run_alps.sh` | ALPS perivascular space metric |
| `extract_shells.py` | Multi-shell b-value extraction |
| `run_DKI.py` | Diffusion kurtosis model fitting |
