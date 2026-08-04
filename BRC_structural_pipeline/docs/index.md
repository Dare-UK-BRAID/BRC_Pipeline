# BRAID Neuroimaging Pipelines

!!! note "Derived Work"
    These pipelines and this documentation are derived from the **BRC Neuroimaging Pipelines**, originally developed by Ali-Reza Mohammadi-Nejad and Stamatios N Sotiropoulos at the University of Nottingham. The BRAID project has adapted them for use within Trusted Research Environments (TREs) under federated analytics frameworks. Original copyright 2018–2024 University of Nottingham.

!!! info "Target Audience"
    Neuroimaging researchers, MRI data engineers, and TRE operators deploying BRAID neuroimaging pipelines.

This site documents the **BRC Neuroimaging Pipelines** — a suite of Bash shell pipelines for end-to-end preprocessing and analysis of multimodal brain MRI data, developed at the University of Nottingham.

The pipelines wrap **FSL**, **FreeSurfer**, **FastSurfer**, **ANTs**, **MRTrix**, and other tools into cohesive workflows that take raw NIfTI data through to analysis-ready outputs.

---

## Pipelines

| Pipeline | Folder | Entry Point | Purpose |
|----------|--------|-------------|---------|
| **Structural** | `BRC_structural_pipeline` | `struc_preproc.sh` | T1/T2 brain extraction, registration, segmentation, lesion detection, surface reconstruction |
| **Diffusion** | `BRC_diffusion_pipeline` | `dMRI_preproc.sh` | DWI preprocessing, eddy correction, DTI/NODDI/DKI model fitting, TBSS, ALPS |
| **Functional** | `BRC_functional_pipeline` | `fMRI_preproc.sh` | fMRI distortion/motion correction, ICA-AROMA denoising, registration, QC |
| **Perfusion** | `BRC_perfusion_pipeline` | `ASL_preproc.sh` | pCASL quantification (CBF), T1 registration, partial volume correction |
| **Functional Group Analysis** | `BRC_func_group_analysis` | `fmri_group_analysis.sh` | Group ICA, dual regression, functional connectivity, FSLNets |
| **IDP Extraction** | `BRC_IDP_extraction` | `idp_extract.sh` | Cohort-level imaging-derived phenotype extraction |

---

## Recommended Processing Order

For a full multimodal dataset, run pipelines in the following order:

```
1. BRC_structural_pipeline    ← prerequisite for all others
2. BRC_diffusion_pipeline     ← depends on T1 registration outputs
3. BRC_functional_pipeline    ← depends on T1 registration outputs
4. BRC_perfusion_pipeline     ← depends on T1 segmentation outputs
5. BRC_IDP_extraction         ← depends on structural + diffusion outputs
6. BRC_func_group_analysis    ← depends on functional pipeline outputs
```

---

## Structural Pipeline Quick Start

```bash
struc_preproc.sh \
  --input /path/to/T1.nii.gz \
  --path /path/to/output \
  --subject Sub_001 \
  --regtype 2
```

See the **Structural Pipeline** section in the navigation for full documentation.

---

## Authors & Citation

**Authors:** Ali-Reza Mohammadi-Nejad & Stamatios N Sotiropoulos

**Copyright:** 2018–2024 University of Nottingham
