# Script Reference

This section provides detailed technical documentation for each script in the BRC Structural Pipeline.

---

## Script Inventory

| Script | Location | Role |
|--------|----------|------|
| [`struc_preproc.sh`](struc-preproc.md) | Root | **Entry point** — argument parsing, directory setup, dispatching |
| [`struc_preproc_part_1.sh`](struc-preproc-part1.md) | `scripts/` | Stage 1 dispatcher — T1 and T2 preprocessing |
| [`struc_preproc_part_2.sh`](struc-preproc-part2.md) | `scripts/` | Stage 2 — FreeSurfer `recon-all` |
| [`struc_preproc_part_3.sh`](struc-preproc-part3.md) | `scripts/` | Stage 3 — FastSurfer + output organisation |
| [`run_T1_preprocessing.sh`](t1-preprocessing.md) | `scripts/` | Core T1 processing (BET, registration, bias, FAST, FIRST) |
| [`run_T2_preprocessing.sh`](t2-preprocessing.md) | `scripts/` | Core T2 processing (co-registration, bias, BIANCA) |
| [`run_T1_sienax.sh`](t1-sienax.md) | `scripts/` | Brain volume estimation (SIENAX approach) |
| [`run_T2_bianca.sh`](t2-bianca.md) | `scripts/` | White matter lesion detection (BIANCA) |
| [`output_organization.sh`](output-organization.md) | `scripts/` | Moves outputs to canonical BRC folder hierarchy |
| [`FSL_anat.sh`](fsl-anat.md) | `scripts/` | Full FSL anatomical preprocessing (fsl_anat-style) |
| [`run_preprocessing.sh`](run-preprocessing.md) | `scripts/` | FreeSurfer-based intensity normalisation |
| [`move_rename.sh`](move-rename.md) | `scripts/` | Legacy output organisation for fsl_anat-based path |

---

## Argument Parsing Convention

All sub-scripts use a shared `getopt1` function for parsing `--key=value` style arguments:

```bash
getopt1()
{
    sopt="$1"
    shift 1
    for fn in $@ ; do
        if [ `echo $fn | grep -- "^${sopt}=" | wc -w` -gt 0 ] ; then
            echo $fn | sed "s/^${sopt}=//"
            return 0
        fi
    done
}
```

Arguments are passed positionally as `--key=value` pairs and extracted by name. An unmatched key returns an empty string.
