# struc_preproc_part_2.sh — FreeSurfer Reconstruction

**Location:** `BRC_structural_pipeline/scripts/struc_preproc_part_2.sh`

---

## Purpose

Runs FreeSurfer full cortical reconstruction (`recon-all -all`) on the raw T1 image. When a T2-FLAIR image is also provided, it is passed to `recon-all` via the `-FLAIR` flag for improved cortical surface delineation. After reconstruction, the automatically created `fsaverage` symbolic link is removed to keep the output folder clean.

!!! note
    If `--dofreesurfer=no`, this script exits immediately without performing any work.

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--dofreesurfer` | yes/no | Master flag — must be `yes` for any processing to occur |
| `--processedt1folder` | Path | Acts as `SUBJECTS_DIR` for FreeSurfer; reconstruction output goes here |
| `--fsfoldername` | String | Subject name passed to `recon-all -s`; output is `processedt1folder/fsfoldername` |
| `--t2` | yes/no | Whether a T2 image is available for FLAIR-guided reconstruction |
| `--rawt1folder` | Path | Source directory for `T1_orig.nii.gz` |
| `--rawt2folder` | Path | Source directory for `T2_orig.nii.gz` (only when `--t2=yes`) |
| `--logt1folder` | Path | Log file path |

---

## Outputs

| Output | Description |
|--------|-------------|
| `<processedT1Folder>/FreeSurfer/` | Full FreeSurfer subject directory (`surf/`, `mri/`, `label/`, `stats/` sub-directories) |

---

## Commands Executed

**T1 only:**
```bash
recon-all -i T1_orig.nii.gz -s FreeSurfer -all
```

**T1 + T2-FLAIR:**
```bash
recon-all -i T1_orig.nii.gz -s FreeSurfer -FLAIR T2_orig.nii.gz -all
```
