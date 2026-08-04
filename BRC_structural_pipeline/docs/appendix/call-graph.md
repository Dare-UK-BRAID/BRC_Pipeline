# Pipeline Call Graph

This page shows the complete call hierarchy for the BRC Structural Pipeline.

---

## Standard (Local) Execution

```
struc_preproc.sh
│
├── struc_preproc_part_1.sh
│     │
│     ├── run_T1_preprocessing.sh
│     │     ├── [fslreorient2std]
│     │     ├── [robustfov + fslmaths]       – FOV crop
│     │     ├── [bet -R]                     – initial brain extraction
│     │     ├── [standard_space_roi]         – FOV reduction
│     │     ├── [flirt -dof 12]              – linear registration
│     │     ├── [fnirt] OR [antsRegistrationSyN.sh]  – non-linear reg
│     │     ├── [applywarp + invwarp]        – atlas brain extraction
│     │     ├── [flirt + fslmaths]           – de-facing
│     │     ├── [fast -b]                    – bias field estimation
│     │     ├── [fslmaths -div]              – bias correction
│     │     ├── [fast PVE threshold]         – tissue masks
│     │     ├── [flirt + applywarp]          – apply registration
│     │     ├── [run_first_all]              – subcortical seg (optional)
│     │     └── run_T1_sienax.sh            – brain volume (RegType 2/3)
│     │           ├── [bet -s]
│     │           ├── [pairreg]
│     │           ├── [avscale]
│     │           └── [fslstats × tissue]
│     │
│     └── run_T2_preprocessing.sh           – (only if T2 provided)
│           ├── [flirt -dof 6]              – T2→T1 co-registration
│           ├── [convert_xfm -concat]       – T2→MNI transform
│           ├── [applywarp]                 – T2 in MNI space
│           ├── [flirt + fslmaths]          – de-facing T2
│           ├── [fslmaths -div]             – bias correction T2
│           └── run_T2_bianca.sh           – lesion detection (RegType 2/3)
│                 ├── [make_bianca_mask]
│                 ├── [bianca]
│                 ├── [fslmaths -mul -thr -bin]
│                 └── [fslstats -V]
│
├── struc_preproc_part_2.sh
│     └── [recon-all -all]                 – FreeSurfer (optional)
│
└── struc_preproc_part_3.sh
      ├── [run_fastsurfer.sh]              – FastSurfer (optional)
      └── output_organization.sh
            ├── [immv × data files]
            ├── [fast -S 2]               – multi-channel seg (T2 present)
            ├── [immv × SIENAX files]
            ├── [immv × BIANCA files]
            └── [imrm × temp files]
```

---

## Cluster Execution

In cluster mode (`CLUSTER_MODE=YES`), the three parts are submitted as dependent jobs:

```
[Job 1] struc_preproc_part_1.sh   (submitted first)
    ↓
[Job 2] struc_preproc_part_2.sh   (FreeSurfer — runs in parallel with Job 1 if --freesurfer)
    ↓
[Job 3] struc_preproc_part_3.sh   (waits on Job 1, and Job 2 if FreeSurfer enabled)
```

If FreeSurfer is not enabled, the dependency chain is simply Job 1 → Job 3.

---

## Legacy Path (fsl_anat-based)

An alternative path using `FSL_anat.sh` and `move_rename.sh` exists but is not the default:

```
FSL_anat.sh
  └── [reorient, crop, bias, reg, seg, FIRST, SIENAX via FSL tools]

move_rename.sh
  └── [organise FSL_anat outputs into BRC folder structure]
  └── [epi_reg for T2→T1 co-registration]
  └── [fast -S 2 for multi-channel segmentation]
```
