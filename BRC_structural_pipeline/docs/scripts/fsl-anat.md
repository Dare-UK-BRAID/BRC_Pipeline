# FSL_anat.sh — FSL Anatomical Preprocessing

**Location:** `BRC_structural_pipeline/scripts/FSL_anat.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 02 Oct 2018

---

## Purpose

A comprehensive standalone FSL anatomical preprocessing script closely modelled on FSL's `fsl_anat` tool, customised for the BRC pipeline. It accepts a single or list of structural MRI images (T1, T2, or PD) and performs the full FSL anat pipeline ending in an `<output>/temp.anat/` directory, whose outputs are subsequently organised by `move_rename.sh`.

---

## Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `-i <image>` | — | Input structural image |
| `-o <name>` | — | Output name stem; creates `<name>/temp.anat/` |
| `-d <dir>` | — | Use an existing `.anat` directory |
| `-s <smooth>` | `20` | Bias field smoothness in mm |
| `-m <mask>` | — | Lesion mask (inverted for bias field and registration) |
| `-t <T1\|T2\|PD>` | `T1` | Image type for FSL FAST tissue model |
| `--list <csv\|file>` | — | Comma-separated list or file of images to average |
| `--clobber` | off | Overwrite existing `.anat` output directory |
| `--noreorient` | off | Skip reorientation |
| `--nocrop` | off | Skip auto-cropping |
| `--nobet` | off | Skip brain extraction |
| `--noreg` | off | Skip registration to standard space |
| `--nononlinreg` | off | Skip non-linear registration |
| `--noseg` | off | Skip tissue segmentation |
| `--nosubcortseg` | off | Skip subcortical segmentation |
| `--nobias` | off | Skip bias field correction |
| `--strongbias` | off | Use strong bias correction (niter=5, smooth=10) |
| `--weakbias` | off | Use weak bias correction *(default: niter=10, smooth=20)* |
| `--betfparam <f>` | `0.1` | BET fractional intensity threshold |
| `--nocleanup` | off | Retain all intermediate files |
| `--anatbasedFS` | off | Use FreeSurfer brain mask in place of FNIRT-derived mask |
| `-logfile <path>` | — | Log file path |
| `-mridir <path>` | — | FreeSurfer `mri/` directory *(required with `--anatbasedFS`)* |
| `-orig <image>` | — | Original (pre-crop) T1 for FOV extraction reference |

---

## Pipeline Stages

### Averaging Multiple Scans *(optional)*

If `--list` is provided, images are averaged using `AnatomicalAverage`.

### Negative Range Fix

```bash
fslstats T1 -p 0   # min value
fslstats T1 -p 100 # max value
```

If negative values are present among positive ones, the minimum is subtracted. If all values are negative, they are made positive while preserving zeros.

### Reorientation

```bash
fslreorient2std T1 > T1_orig2std.mat
```

Forward and inverse reorientation matrices are saved for downstream transforms.

### Auto-Cropping

```bash
robustfov -i T1_fullfov -r T1 -m T1_roi2nonroi.mat
```

Crops the FOV and records the crop-to-full-FOV transform.

### Lesion Masking

If a lesion mask is provided, it is reoriented and cropped to match the processed T1. A binary inverse mask (`lesionmaskinv`) is used to exclude lesion voxels from bias estimation and registration.

### Bias Field Correction

=== "Weak bias (default)"

    A single BET pass gets a rough brain mask, followed by two rounds of FSL FAST bias estimation:

    ```bash
    bet T1 T1_initfast2_brain -m -f 0.1
    fast -o T1_fast -l 20 -b -B -t 1 --iter=10 --nopve --fixed=0 T1_initfast2_maskedrestore
    ```

=== "Strong bias (--strongbias)"

    An extra large-scale correction step using `quick_smooth` precedes the FAST-based correction:

    ```bash
    # Stage 1: large-scale (subsampling-based smoothing)
    fslmaths T1 -div T1_s20 T1_hpf
    # Stage 2: detailed FAST-based correction
    fast -o T1_initfast -l 10 -b -B -t 1 --iter=5 --nopve --fixed=0 T1_hpf2_maskedbrain
    ```

### Registration and Brain Extraction

```bash
flirt -interp spline -dof 12 -in T1_biascorr -ref MNI152_T1_2mm \
      -omat T1_to_MNI_lin.mat
fnirt --in=T1_biascorr --ref=MNI152_T1_2mm --fout=T1_to_MNI_nonlin_field \
      --cout=T1_to_MNI_nonlin_coeff --aff=T1_to_MNI_lin.mat
```

The MNI brain mask is warped back to native space via the inverse FNIRT warp for brain extraction.

### Tissue-Type Segmentation

```bash
fast -o T1_fast -l 20 -b -B -t 1 --iter=10 T1_biascorr_maskedbrain
```

The bias field is refined using FAST, and the corrected image is re-warped to MNI.

### Skull-Constrained Volume Estimation

```bash
pairreg MNI152_T1_2mm_brain T1_biascorr_bet \
        MNI152_T1_2mm_skull T1_biascorr_bet_skull \
        T12std_skullcon.mat
avscale T12std_skullcon.mat  # → vscale
```

Grey and white matter volumes are scaled by `vscale` and written to `T1_vols.txt`.

### Subcortical Segmentation

```bash
first_flirt T1_biascorr T1_biascorr_to_std_sub
run_first_all -i T1_biascorr -o first_results/T1_first \
              -a T1_biascorr_to_std_sub.mat
```

---

## Internal Helper Functions

| Function | Description |
|----------|-------------|
| `quick_smooth(in, out)` | Fast Gaussian smoothing approximation via 4× subsampling + FLIRT resampling. Used for large-scale bias estimation. |
| `get_opt1(arg)` | Extracts option name (before `=`) from a `--key=value` argument |
| `get_arg1(arg)` | Extracts value (after `=`) from `--key=value`; exits if no value |
| `get_imarg1(arg)` | Like `get_arg1` but also strips the image extension using `remove_ext` |
| `get_arg2(opt, val)` | Extracts value from `-key value` two-token pair |
| `run(cmd...)` | Logs the command at level 2 then executes it |
