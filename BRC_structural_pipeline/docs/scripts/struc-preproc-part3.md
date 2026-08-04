# struc_preproc_part_3.sh — FastSurfer + Output Organisation

**Location:** `BRC_structural_pipeline/scripts/struc_preproc_part_3.sh`

---

## Purpose

Final processing stage that:

1. Optionally runs **FastSurfer** cortical reconstruction
2. Calls `output_organization.sh` to move all intermediate outputs into the canonical BRC folder hierarchy
3. Calls `Show_version.sh --showdiff=yes` to record total pipeline runtime in the log

!!! warning "T2 + FastSurfer"
    If both `--t2=yes` and `--dofastsurfer=yes` are set, the script logs a `WARNING` stating that T2 data will be ignored by FastSurfer (which currently accepts only T1 input).

---

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `--dofastsurfer` | yes/no | Whether to run FastSurfer |
| `--fastsurferfoldername` | String | Subject name for FastSurfer output directory |
| `--starttime` | Unix timestamp | Pipeline start time (seconds since epoch); used to compute total runtime |
| `--subid` | String | Subject identifier written to the timing log |
| `--processedt1folder` | Path | FastSurfer `SUBJECTS_DIR` |
| `--t2` | yes/no | Whether T2 data is present (passed to output_organization.sh) |
| `--rawt1folder` | Path | Raw T1 folder (passed to output_organization.sh) |
| *(all other paths)* | Path | Complete set of folder paths forwarded to `output_organization.sh` |
| `--logt1folder` | Path | Log file path |

---

## Calls

```
struc_preproc_part_3.sh
  ├── run_fastsurfer.sh          (only if --dofastsurfer=yes)
  ├── output_organization.sh     (always)
  └── Show_version.sh --showdiff=yes
```

---

## FastSurfer Command

```bash
run_fastsurfer.sh \
  --t1 <rawT1Folder>/T1_orig.nii.gz \
  --sid <FastSurferFolderName> \
  --sd <processedT1Folder>
```

The Python module `brcpython-img` (cluster mode) or `brcpython` (local mode) is loaded before invoking FastSurfer.
