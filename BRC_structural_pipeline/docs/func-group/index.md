# Functional Group Analysis Pipeline

The **BRC Functional Group Analysis Pipeline** (`BRC_func_group_analysis`) performs group-level resting-state fMRI analysis: parcellation-based timeseries extraction, functional connectivity matrix estimation, group ICA (MELODIC), dual regression, and cross-subject statistical inference.

---

!!! note "Prerequisites"
    This pipeline requires completed [Functional Pipeline](../functional/index.md) runs for all subjects, which in turn requires a completed [Structural Pipeline](../pipeline/overview.md) run.


The pipeline takes individually preprocessed and registered fMRI data (output of the `BRC_functional_pipeline`) and performs:

- ROI timeseries extraction using standard or user-defined atlases
- Pairwise functional connectivity (FC) matrix estimation with multiple association measures
- Group ICA decomposition via FSL MELODIC
- Dual regression to derive subject-specific spatial maps and timeseries
- FSLNets-based network analysis and group-difference GLMs (via MATLAB)

---

## Quick Start

```bash
fmri_group_analysis.sh \
    --in   /data/study/subject_list.txt \
    --indir /data/study \
    --outdir /data/study/group \
    --parcellation SHEN \
    --fmrires 3 \
    --tr 1.45 \
    --groupdiffs
```

Where `subject_list.txt` is a space-separated file: subject ID, group label (1 or 2).

---

## Compulsory Arguments

| Argument | Description |
|----------|-------------|
| `--in` | Text file listing subject IDs (and optionally group labels) |
| `--indir` | Input directory containing one preprocessed folder per subject |
| `--outdir` | Output directory for group results |
| `--parcellation` | Atlas to use for ROI extraction (see below) |
| `--fmrires` | Data resolution of fMRI and atlas (mm) |
| `--tr` | Repetition time in seconds |

---

## Parcellation Options

| Atlas | Description |
|-------|-------------|
| `FS_DKA` | FreeSurfer Desikan-Killiany cortical atlas (native fMRI space) |
| `FS_DA` | FreeSurfer Destrieux cortical atlas (native fMRI space) |
| `AAL` | Automated Anatomical Labeling — 116 ROIs in MNI152 space |
| `SHEN` | Shen functional atlas — 268 ROIs (cortical + subcortical) in MNI152 space |
| `MELODIC` | Group ICA decomposition (requires `--approach` and `--dim`) |
| `NONE` | User-defined atlas — specify with `--inatlas` and `--labels` |

---

## Optional Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--labels` | all | List of label values to extract from atlas |
| `--corrtype` | `CORR` | FC association measure (see below) |
| `--regval` | 0 / 0.1 | Regularisation parameter for PCORR / RPCORR |
| `--varnorm` | `0` | Temporal variance normalisation: `0`=none, `1`=whole-subject stddev, `2`=per-timeseries |
| `--groupdiffs` | off | Run cross-subject GLM for group differences |
| `--approach` | `concat` | Group ICA approach: `concat` (temporally concatenated) or `tica` (tensor-ICA) |
| `--dim` | auto | ICA dimensionality (number of components) |
| `--inatlas` | — | User-defined atlas NIFTI |
| `--gmroi` | off | Add MNI152 GM mask as an additional ROI |
| `--nofr2z` | off | Disable Fisher r-to-Z transformation of FC matrices |

---

## Functional Connectivity Measures

| Code | Method |
|------|--------|
| `CORR` | Full (Pearson) correlation — diagonal zeroed (default) |
| `COV` | Covariance (non-normalised correlation) |
| `AMP` | Node amplitude only |
| `RCORR` | Full correlation after regressing out global mean |
| `PCORR` | Partial correlation; with `--regval λ`, uses L1-norm regularisation |
| `RPCORR` | Ridge regression partial correlation (L2-norm Tikhonov); `--regval ρ` |

---

## Processing Steps

### Melodic Processing (`Melodic_Processing.sh`)

Runs FSL `melodic` on the concatenated or tensor group dataset to produce group-ICA spatial maps. The approach (`--approach`) and number of components (`--dim`) control decomposition strategy.

### Dual Regression (`Dual_Regression_Processing.sh`)

Performs dual regression against the group ICA maps to generate:

1. Subject-specific spatial maps (regression of group maps against each subject's 4D data)
2. Subject-specific timeseries (regression of spatial maps back into 4D data)

Outputs can be used for group-level comparisons of spatial map intensities.

### Reference Network Generation (`Generate_ref_Networks.sh`)

Creates reference network visualisations from the group ICA decomposition.

### Functional Connectivity Analysis (`Functional_Connectivity_Analysis.sh`)

Extracts ROI mean timeseries from each subject using the selected parcellation, then estimates pairwise FC matrices with the chosen association measure.

### Single-Subject FC Analysis (`SS_FC_Analysis.sh`)

Applies the same FC pipeline to individual subjects independently (without group ICA).

### Group Design Generation (`Generate_design.sh`)

Constructs FSL randomise-compatible design matrices and contrast files for cross-subject inference.

### Group Map Generation (`Generate_maps.sh`)

Generates group-average maps for visualisation and thresholded statistical maps.

### FSLNets Network Analysis

MATLAB toolbox (`FSLNets/`) provides network matrix analysis, including:

- `nets_load.m` — load and preprocess network matrices
- `nets_netmats.m` — estimate network matrices
- `nets_glm.m` — cross-subject GLM
- `nets_stats.m` — permutation inference
- `nets_netweb.m` — interactive network visualisation (via `netjs/`)
- `nets_hierarchy.m` — hierarchical clustering
- `nets_spectra.m` — spectral analysis

The L1precision toolbox (`L1precision/`) implements L1-regularised precision matrix estimation for sparse partial correlation networks.

---

## Dependencies

| Tool | Used For |
|------|----------|
| FSL (`melodic`, `dual_regression`, `randomise`, `fslmaths`) | ICA, statistical inference, group GLM |
| MATLAB + FSLNets | Network matrix analysis, visualisation |
| `BRC_GLOBAL_SCR` | Logging |

---

## Key Output Files

All paths are relative to `<outdir>/Group_Analysis/`.

| File | Description |
|------|-------------|
| `melodic_IC.nii.gz` | Group ICA spatial maps |
| `dr_stage1_*.nii.gz` | Dual regression stage 1 spatial maps (per subject) |
| `dr_stage2_*.txt` | Dual regression stage 2 timeseries (per subject) |
| `FC_matrices/` | Per-subject functional connectivity matrices |
| `group_FC_mean.txt` | Group-average FC matrix |
| `GLM/` | Design matrix, randomise results |
| `log/log.txt` | Processing log |

---

## Script Reference

| Script | Purpose |
|--------|---------|
| `fmri_group_analysis.sh` | Entry point — argument parsing, orchestration |
| `Melodic_Processing.sh` | Group ICA via FSL MELODIC |
| `Dual_Regression_Processing.sh` | Dual regression for subject-specific maps |
| `Functional_Connectivity_Analysis.sh` | ROI timeseries extraction + FC matrix estimation |
| `SS_FC_Analysis.sh` | Single-subject FC analysis |
| `Generate_design.sh` | FSL design matrix generation |
| `Generate_maps.sh` | Group-average map creation |
| `Generate_ref_Networks.sh` | Reference network visualisations |
| `FSLNets/` | MATLAB FSLNets network analysis toolbox |
| `L1precision/` | MATLAB L1-regularised precision matrix estimation |
