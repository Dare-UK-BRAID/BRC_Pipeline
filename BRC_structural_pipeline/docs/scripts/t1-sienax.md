# run_T1_sienax.sh — Brain Volume Estimation

**Location:** `BRC_structural_pipeline/scripts/run_T1_sienax.sh`  
**Authors:** Ali-Reza Mohammadi-Nejad, Stamatios N Sotiropoulos  
**Last updated:** 16 Mar 2021

!!! note
    This script is called automatically by `run_T1_preprocessing.sh` when `--regtype` is 2 or 3. It does not need to be run manually.

---

## Purpose

Implements a **SIENAX-style brain volume estimation pipeline** using FSL tools. It computes a skull-constrained volumetric scaling factor (`vscale`) by registering the T1 skull surface to MNI, then uses this factor to produce normalised brain volume estimates from the FAST partial volume estimates.

---

## Processing Steps

### 1. Skull Surface Extraction

```bash
bet T1 SIENAX/T1_brain -s
```

The `-s` flag extracts the skull surface (`T1_brain_skull`) used for pairreg.

### 2. Skull-Constrained Registration

```bash
pairreg \
    MNI152_T1_2mm_brain \
    T1_brain \
    MNI152_T1_2mm_skull \
    SIENAX/T1_brain_skull \
    SIENAX/T1_to_MNI_linear.mat
```

Registers brain+skull simultaneously to MNI to get a skull-constrained affine.

### 3. Volume Scaling Factor

```bash
avscale SIENAX/T1_to_MNI_linear.mat MNI152_T1_2mm
```

Extracts x, y, z scale factors from the affine decomposition. The volumetric scaling factor is:

```
vscale = xscale × yscale × zscale
```

### 4. Tissue Volumes

For each tissue, the script computes:

```
unnormalised volume = mean(PVE) × voxel_count
normalised volume   = unnormalised × vscale
```

Tissues reported in `report.sienax`:

| Label | Source PVE | Region |
|-------|-----------|--------|
| `pgrey` | `T1_brain_pve_1` | Peripheral grey matter (masked to periph. grey region) |
| `vcsf` | `T1_brain_pve_0` | Ventricular CSF (masked to ventricle region) |
| `GREY` | `T1_brain_pve_1` | Total grey matter |
| `WHITE` | `T1_brain_pve_2` | Total white matter |
| `BRAIN` | GM + WM | Total brain |

Peripheral grey and ventricular masks are warped from MNI to native space using the inverse non-linear warp.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--workingdir` | Path | T1 temp folder (provides `T1`, `T1_brain`, `reg/T1_to_MNI_nonlin_coeff_inv`) |
| `--sienaxtempfolder` | Path | Directory where all SIENAX intermediate and output files are written |
| `--fastfolder` | Path | FAST output folder (provides `T1_brain_pve_{0,1,2}`) |
| `--logfile` | Path | Log file path |

---

## Key Outputs

| File | Description |
|------|-------------|
| `SIENAX/report.sienax` | Text report with `VSCALING`, `pgrey`, `vcsf`, `GREY`, `WHITE`, `BRAIN` volumes in mm³ |
| `SIENAX/T1_to_MNI_linear.mat` | Skull-constrained registration affine |
| `SIENAX/T1_to_MNI_linear_avscale` | Full `avscale` output (contains scale factors) |

---

## Example report.sienax

```
VSCALING 1.0342
tissue             volume    unnormalised-volume
pgrey              453221.23 438375.81 (peripheral grey)
vcsf               12042.11  11644.50 (ventricular CSF)
GREY               672384.12 650191.33
WHITE              498221.44 481738.20
BRAIN              1170605.56 1131929.53
```
