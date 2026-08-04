# Environment Variables Reference

This page provides a complete reference for all environment variables used by the BRC Structural Pipeline.

---

## Required Variables

These must be set before running any part of the pipeline:

| Variable | Description | Example |
|----------|-------------|---------|
| `FSLDIR` | FSL installation directory | `/usr/local/fsl` |
| `BRC_GLOBAL_SCR` | Path to BRC global utility scripts (`log.shlib`, etc.) | `/opt/brc/global/scripts` |
| `BRC_SCTRUC_SCR` | Path to structural pipeline `scripts/` directory | `/opt/brc/BRC_structural_pipeline/scripts` |
| `BRC_GLOBAL_DIR` | Path to BRC global data directory (templates, configs) | `/opt/brc/global` |
| `BRCDIR` | Root BRC directory (`Show_version.sh` lives here) | `/opt/brc` |

---

## Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLUSTER_MODE` | `NO` | Set to `YES` to submit pipeline stages as cluster jobs |
| `JOBSUBpath` | — | Path to `jobsub` binary. Required when `CLUSTER_MODE=YES` |
| `FAST_t` | `1` | FSL FAST image type: `1` = T1w, `2` = T2w, `3` = PD |
| `RUN` | *(blank)* | Command prefix for execution (blank for local) |
| `Subject` | — | Subject identifier used to name cluster job IDs |

---

## Conditionally Required Variables

These are required only when specific pipeline options are used:

| Variable | Required When | Description |
|----------|--------------|-------------|
| `ANTSPATH` | `--regtype 3` | Path to ANTs binaries directory (e.g. `/opt/ANTs/bin`) |
| `C3DPATH` | `--regtype 3` | Path to c3d binaries directory (e.g. `/opt/c3d/bin`) |

---

## Setting Up the Environment

A typical setup script might look like:

```bash
# FSL
export FSLDIR=/usr/local/fsl
source ${FSLDIR}/etc/fslconf/fsl.sh

# BRC Pipeline
export BRCDIR=/opt/brc
export BRC_GLOBAL_SCR=${BRCDIR}/global/scripts
export BRC_GLOBAL_DIR=${BRCDIR}/global
export BRC_SCTRUC_SCR=${BRCDIR}/BRC_structural_pipeline/scripts

# Optional: ANTs (for --regtype 3)
export ANTSPATH=/opt/ANTs/bin
export C3DPATH=/opt/c3d/bin

# Optional: cluster mode
export CLUSTER_MODE=NO
export FAST_t=1
```
