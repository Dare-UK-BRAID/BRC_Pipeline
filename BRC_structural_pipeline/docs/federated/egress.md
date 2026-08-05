# Egress Risk Analysis

This page documents a systematic audit of the BRAID pipeline codebase for variables, files, and metadata that present disclosure risks when transmitted outside a TRE in a federated or multi-site context. The audit covers all six pipelines: Structural, Diffusion, Functional, Perfusion, Functional Group Analysis, and IDP Extraction.

Under the DPUK Five Safes model, **Safe Outputs** (Safe 5) requires that all results leaving the TRE are checked to ensure they do not contain row-level (individual-level) data. The outputs catalogued here are produced at individual subject level and therefore require controls before egress.

---

## Critical Risk: The IDP Matrix (`IDPs.txt`)

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/idp_extract_part_1.sh`

```bash
result="${Subject}"         # Subject ID is the FIRST field
# ... per-subject IDP extraction ...
echo $result >> ${GroupIDPFolder}/IDPs.txt
```

The group-level `IDPs.txt` is a concatenation of **one row per participant**, with the subject identifier as the first column. If this file leaves the TRE — even for federated aggregation — it constitutes a direct release of individual-level health data. This is not a model parameter or aggregate statistic; it is a pseudonymised but highly re-identifiable personal data record.

**Risk:** **Critical**. This file must never be transmitted outside the TRE without applying the full suite of controls described in [Disclosure Thresholds](thresholds.md) and [Differential Privacy](differential-privacy.md).

---

## FreeSurfer IDPs: ~1,000+ Scalar Fingerprint

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/brc_FS_get_IDPs.py`

The `brc_FS_get_IDPs` function extracts a comprehensive set of neuroanatomical phenotypes from FreeSurfer output, including:

- **Subcortical volumes** (`aseg.stats`): bilateral hippocampus, amygdala, caudate, putamen, thalamus, pallidum, accumbens, brain stem — all left/right hemispheres
- **Cortical parcellation** (Desikan-Killiany, Destrieux, DKT, Brodmann): volume, thickness, and surface area per ROI across both hemispheres (~400–600 scalars per atlas)
- **White matter intensity contrast** (`wg_lh_mean`, `wg_rh_mean`): left and right hemisphere WM/GM contrast
- **Global metrics**: total intracranial volume (eTIV), brain-to-eTIV ratio, number of surface holes before fixing
- **Hippocampal subfield volumes** (`HippSubfield_lh/rh`): 22 values per hemisphere
- **Amygdala nuclei** (`AmygNuclei_lh/rh`): 10 values per hemisphere
- **Brainstem segmentation**: 5 sub-compartment volumes
- **FLAIR usage flag**: binary indicator of whether T2-FLAIR was used in FreeSurfer — reveals acquisition protocol

```python
def save_data(data_dict, SUBJECTS_DIR):
    # ...
    full_values = [temp_headers[x] for x in final_headers]
    full_values_str = " ".join(full_values)

    with open(SUBJECTS_DIR + '/FS_IDPs.txt', 'w') as f:
        f.write(full_values_str + "\n")  # Includes subject ID as first element
```

!!! warning "Re-identification risk"
    Wachinger et al. (2015) demonstrated that **brain morphology alone can uniquely identify individuals** with >99.9% accuracy using only subcortical volumes (BrainPrint). The combination of 1,000+ FreeSurfer IDPs creates an essentially unique neuroanatomical fingerprint. Finn et al. (2015) showed similar uniqueness for functional connectivity profiles. These IDPs should be considered **quasi-identifiers** under GDPR.

---

## SIENAX Variables: Brain Volume and Head Size Proxy

**File:** `BRC_Pipeline/BRC_structural_pipeline/scripts/run_T1_sienax.sh`

```bash
vscale=`echo "10 k $xscale $yscale * $zscale * p"|dc -`
echo "VSCALING $vscale" >> ${SienaxTempFolder}/report.sienax
```

**Variables extracted by `brc_IDP_T1_SIENAX.sh`:** `VSCALING`, peripheral grey volume, ventricular CSF volume, normalised grey matter, unnormalised grey matter, normalised white matter, unnormalised white matter, normalised brain volume, unnormalised brain volume (11 values total).

**Risks:**
- **VSCALING** (the volumetric scaling factor from T1 to MNI) is a proxy for **head size** and is correlated with sex, age, and ancestry. It may serve as a soft identifier.
- **Unnormalised volumes** (without VSCALING correction) reflect absolute head dimensions and are more directly identifying than normalised volumes.
- Total brain volume combined with age and sex narrows the identifiable population substantially.

---

## BIANCA: White Matter Hyperintensity Volume

**File:** `BRC_Pipeline/BRC_structural_pipeline/scripts/run_T2_bianca.sh`  
**IDP output:** `brc_IDP_T2_FLAIR_WMH.sh` → `volume.txt` → scalar WMH volume

WMH volume is strongly correlated with **cerebrovascular disease burden**, age, and dementia risk. In cohorts enriched for dementia (e.g. DPUK datasets), a high WMH volume is a sensitive attribute. Combined with demographic co-variates, it narrows the identifiable population.

The BIANCA classifier used (`${BRC_GLOBAL_DIR}/templates/bianca_class_data`) is a **pre-trained model**. In a federated context, this raises the additional concern of model inversion — an adversary with access to the model weights could attempt to reconstruct properties of the training data.

---

## TBSS Diffusion IDPs: White Matter Skeleton Metrics

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/brc_IDP_diff_TBSS.sh`

```bash
for i in FA MD MO L1 L2 L3 ICVF ISOVF ODI ; do
    # reads JHURois_${i}.txt — 50 JHU atlas ROI values per metric
    result="$result $miniResult"
done
```

**Output:** 9 diffusion metrics × 50 JHU ROIs = **450 scalar values per subject**.

- **FA (fractional anisotropy)** and **MD (mean diffusivity)** in specific white matter tracts are associated with neurodegenerative conditions and are moderately identifying in clinical populations.
- **NODDI metrics** (ICVF, ISOVF, ODI) are more biophysically specific and may reveal microstructural abnormalities linked to specific diseases.
- The combination of 450 tract-specific values constitutes a **white matter fingerprint** that has been shown to uniquely identify individuals across scanning sessions (Mansour et al., 2021).

---

## Registration Quality Metrics: Scanner and Site Fingerprinting

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/brc_IDP_T1_align_to_std.sh`

```bash
result1=`${FSLDIR}/bin/flirt -in ${DataSubjFolder}/T1_brain -ref ${ST}/MNI152_T1_1mm_brain \
    -init ${RegSubjFolder}/T1_2_std.mat -schedule ${MC} | head -1 | cut -f1 -d' '`

result3=`${FSLDIR}/bin/fslstats ${T1wSubjFolder}/${tempFolderName}/temp \
    -k ${ST}/MNI152_T1_1mm_brain_mask -m`
```

Three values are extracted: (1) registration cost at native space, (2) registration cost in linear standard space, (3) mean squared Jacobian deviation from identity (i.e. local volume change from T1 to MNI).

**Risks:**
- **Registration cost** metrics are sensitive to acquisition protocol and scanner hardware. They can act as **site-level fingerprints**, potentially re-identifying the contributing TRE even in aggregated outputs.
- The **Jacobian deviation** metric (`result3`) encodes average local brain deformation — essentially a scalar summary of morphological difference from the MNI template. This correlates with pathology and age.

---

## SNR / CNR Metrics: Scanner Fingerprinting

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/brc_IDP_T1_noise_ratio.sh`

```bash
TheSNRrecip=`echo "10 k ${TheNoise} ${TheBrain} / p" | dc -`
TheCNRrecip=`echo "10 k ${TheNoise} ${TheContrast} / p" | dc -`
result="${TheSNRrecip} ${TheCNRrecip}"
```

SNR and CNR are sensitive to scanner field strength, coil configuration, and acquisition protocol. In multi-site federated studies, **site membership can be inferred from SNR distributions**, enabling indirect re-identification by site.

---

## Eddy Outlier Count: Acquisition Artifact Proxy

**File:** `BRC_Pipeline/BRC_IDP_extraction/scripts/brc_IDP_diff_eddy_outliers.sh`

```bash
result=`wc -l ${dMRIDataSubjFolder}/eddy_unwarped_images.eddy_outlier_report | awk '{print $1}'`
```

The number of eddy-identified outlier slices is a proxy for head motion during DWI acquisition. High outlier counts correlate with cognitive impairment and are sensitive health attributes. They are also scanner- and operator-dependent.

---

## Log Files: Direct Identifier Leakage

**File:** All pipeline scripts via `log.shlib`

```bash
log_Msg 2 "WD:$WD"
log_Msg 2 "SienaxTempFolder:$SienaxTempFolder"
```

Every pipeline script logs its working directory and key parameters at `log_Msg 2` level. These paths embed **subject identifiers** directly (e.g. `/data/study/sub-001/analysis/...`). Log files at the group or pipeline level aggregate these subject-specific paths.

**Log files must not leave the TRE under any circumstances.** In federated deployments, log transmission must be explicitly blocked at the network egress layer.

---

## Subject ID Retention in Output Files

The following code in `idp_extract_part_1.sh` initialises each result row with the subject identifier:

```bash
result="${Subject}"
# ...
echo $result >> ${GroupIDPFolder}/IDPs.txt
```

And in `brc_FS_get_IDPs.py`:

```python
data_dict['ID'] = [['ID'], [subject]]
# ...
full_values_str = " ".join(full_values)
with open(SUBJECTS_DIR + '/FS_IDPs.txt', 'w') as f:
    f.write(full_values_str + "\n")  # ID is first column
```

Although the function returns `id_removed_str` (without the ID) to the calling script, the **full string including ID is written to `FS_IDPs.txt`**. Both the per-subject and group-level IDP files therefore retain direct subject identifiers. Any federated transmission of these files includes direct identifiers and constitutes a personal data transfer under UK GDPR.

---

## Functional Pipeline: Motion Parameters and ICA Components

**Files:** `BRC_Pipeline/BRC_functional_pipeline/scripts/`

Motion parameters (`mc/prefiltered_func_data_mcf.par`) are six time-series values (3 translation, 3 rotation) per volume. They are:
- **Individual-level**: motion is a personal behavioural characteristic correlated with anxiety, ADHD, and other conditions
- **Session-linkable**: the specific motion trajectory of a scanning session is a quasi-identifier

ICA-AROMA classifies components as signal vs motion noise. The **list of noise components** and the spatial maps of those components are available in `ICA_testout.ica/`. Spatial ICA maps are derived from the individual's brain activity and constitute individual-level data.

---

## Dataset-Wide Computations: Federated Architecture Constraints

A federated deployment assumes that each site runs the pipeline independently on its local cohort and transmits only derived outputs. This assumption holds for most BRAID pipelines — but not all. Several computations are **inherently dataset-wide**: they cannot produce valid outputs unless all subjects' data is present simultaneously, which means they are architecturally incompatible with standard federated analytics.

### Computations That Require the Entire Dataset

#### MELODIC Group ICA — `Melodic_Processing.sh`

```bash
$FSLDIR/bin/melodic \
    --in=${InputFiles} \        # list of ALL subjects' 4D fMRI files
    --approach=${ICAapproach}   # concat or tica
```

`InputFiles` is a file listing every subject's preprocessed fMRI. MELODIC factorises the **concatenated or tensor-decomposed** group dataset. The resulting spatial maps are a property of the whole cohort — they change if any subject is added or removed. This cannot be run per-site and combined because ICA is a global decomposition with no separable local update.

#### Dual Regression — `Dual_Regression_Processing.sh`

Dual regression takes the MELODIC group maps as its primary input. Even though each regression step processes one subject at a time, the **maps are derived from the whole cohort**. Change the group ICA and all dual regression outputs change. This is a two-stage dataset dependency: federating dual regression requires first solving the group ICA problem across sites.

#### FSLNets Group GLM — `nets_glm.m`

```matlab
[p_uncorrected, p_corrected] = nets_glm(netmats, design_matrix, contrast_matrix, ...)
```

FSL `randomise`-based permutation testing requires the full subjects × features matrix (`netmats`) to be present simultaneously to build the design and compute the permutation distribution. No partial site contribution is meaningful without the whole.

#### Standard TBSS — but not as implemented here

Classical TBSS computes a **group mean FA image and group-derived skeleton**, then projects all subjects onto that skeleton. This is dataset-wide. However, the BRC implementation adapts TBSS for single-subject use:

```bash
# tbss_step_3_postreg.sh
${FSLDIR}/bin/imcp ../FA/dti_FA_to_MNI all_FA   # copies ONE subject, not a group concat
```

Instead of a cohort-derived mean, the BRC pipeline uses the **fixed MNI `FMRIB58_FA_1mm` template** as the skeleton reference. The IDP output (`JHUrois_FA.txt`) is therefore each subject's own mean FA within each JHU atlas ROI — no cross-subject statistic is involved. This is fully federable, but note that the resulting IDPs are not directly comparable to those produced by group-mode TBSS.

### Computations That Appear Dataset-Wide But Are Per-Subject

| Computation | Code | Why it is per-subject |
|------------|------|----------------------|
| fMRI intensity normalisation | `Intensity_Normalization.sh` | Scales to fixed constant 10,000 (`-ing 10000`), not a cohort mean |
| SIENAX VSCALING | `run_T1_sienax.sh` | Registers to fixed `MNI152_T1_2mm` template; no cohort statistics |
| FSLNets variance normalisation | `nets_load.m` | `grot/std(grot(:))` — divides by each subject's **own** timeseries stddev |
| FreeSurfer IDP extraction | `brc_FS_get_IDPs.py` | Processes one subject at a time; `asegstats2table --subjects subject_ID` |
| BIANCA WMH segmentation | `run_T2_bianca.sh` | Per-subject inference using a fixed pre-trained classifier |
| Eddy correction | `run_eddy.sh` | Per-volume within a single subject's acquisition |

!!! warning "Functional group analysis pipeline is not directly federable"
    The **MELODIC → dual regression → FSLNets** pipeline is a serial dependency chain where the first step requires all subjects' raw data simultaneously. Standard federated averaging cannot be applied. Federating this pipeline requires replacing MELODIC with a **federated ICA algorithm** (e.g. CanICA across sites, or a consensus ICA approach), computing site-local dual regression against agreed group maps, and then aggregating FC matrices centrally under secure aggregation. This represents a significant architectural redesign and is outside the scope of the current BRAID pipeline.

### Federated Compatibility Summary

| Pipeline | Federable as-is? | Blocker (if any) |
|----------|:---:|------------------|
| Structural (T1/T2) | **Yes** | None — all steps per-subject |
| Diffusion (dMRI, TBSS IDPs) | **Yes** | BRC TBSS uses fixed MNI template |
| Perfusion (ASL/pCASL) | **Yes** | None — all steps per-subject |
| IDP Extraction | **Yes** | Aggregation step only; see [Egress risks above](#critical-risk-the-idp-matrix-idpstxt) |
| Functional (fMRI preprocessing) | **Yes** | Preprocessing per-subject; ICA-AROMA uses pre-trained classifier |
| Functional Group Analysis | **No** | MELODIC requires all subjects; dual regression and FSLNets GLM depend on it |

---

## Summary of Risks by Category

| Category | Variables / Files | Risk |
|----------|------------------|------|
| Direct identifiers | Subject ID in `IDPs.txt`, `FS_IDPs.txt`, log paths | **Critical** |
| Neuroanatomical fingerprint | 1,000+ FreeSurfer IDPs | **Critical** |
| White matter fingerprint | 450 TBSS diffusion IDPs | **High** |
| Brain volume | SIENAX 11 IDPs incl. VSCALING | **High** |
| Disease markers | WMH volume (BIANCA), eddy outliers | **High** |
| Morphological deviation | Jacobian metric, registration cost | **Medium** |
| Scanner/site fingerprint | SNR, CNR, registration cost | **Medium** |
| Motion behaviour | fMRI motion parameters | **Medium** |
| Acquisition metadata | FLAIR usage flag, scanner parameters in log | **Low–Medium** |

---

## References

- Wachinger, C. et al. (2015). BrainPrint. *NeuroImage*, 109, 232–248.
- Finn, E.S. et al. (2015). Functional connectome fingerprinting. *Nature Neuroscience*, 18(11), 1664–1671.
- Mansour, L. et al. (2021). Connectome-based fingerprinting: reproducibility across sites and cohorts. *NeuroImage*, 232, 117852.
- Melis, L. et al. (2019). Exploiting unintended feature leakage in collaborative learning. *IEEE S&P 2019*.
- UK GDPR Article 9 — Special categories of personal data.
- DPUK Data Provider Handbook (2024). Dementias Platform UK.
