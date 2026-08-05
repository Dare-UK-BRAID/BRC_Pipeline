# Disclosure Thresholds & Output Controls

Disclosure thresholds define the minimum conditions under which statistical outputs from BRAID pipeline analyses may be released from a TRE. They are a core component of **Safe Outputs** (Safe 5 of the DPUK Five Safes framework) and complement cryptographic protections by providing a last line of defence before any aggregate statistic is transmitted to a central aggregator or published.

This page defines thresholds specific to BRAID neuroimaging IDPs, drawing on DPUK governance, UK Office for National Statistics (ONS) guidance, and the Statistical Disclosure Control (SDC) literature for high-dimensional biomedical data.

---

## General Principles

The guiding question is: **does the released statistic allow an adversary to infer whether a specific individual is present in the cohort, or to infer a sensitive attribute of an identified individual?**

Three classes of output require different controls:

1. **Aggregate statistics** (means, medians, variances of IDP distributions across a cohort)
2. **Model parameters** (regression coefficients, neural network weights trained on IDP features)
3. **Count statistics** (number of participants meeting a criterion, e.g. WMH > 50 cm³)

---

## Minimum Cohort Size

### Rule 1: Absolute minimum n ≥ 10 (DPUK / ONS standard)

No aggregate statistic may be released for a (sub)group of fewer than **10 participants**. This is the standard lower bound applied by DPUK data analysts during Safe Outputs review.

### Rule 2: Recommended minimum n ≥ 50 for high-dimensional IDP data

For the ~1,000-dimensional BRAID FreeSurfer IDP vector, the ONS minimum of 10 is insufficient. The **effective degrees of freedom** in the IDP space far exceed 10, meaning that with fewer than ~50 participants, individual contributions are easily inferred from the aggregate mean.

Recommendation: **n ≥ 50 per site** before releasing any IDP summary statistics in a federated context. For diffusion TBSS metrics (450 dimensions), **n ≥ 30** is acceptable.

### Rule 3: Federated round minimum n_total ≥ 100

In federated learning, at least **100 total participants across all sites** must contribute to a global model update before that update is used to update the global model. Sites with fewer than n_local ≥ 20 participants should be excluded from the federated round.

---

## Cell Suppression for Count Statistics

When releasing counts (e.g. number of participants with WMH volume > threshold), apply **primary suppression** (suppress cells with count < 10) and **secondary suppression** (suppress complementary cells to prevent back-calculation).

**Example:** In a cohort of 80 participants, if 7 have WMH > 100 cm³, the count 7 must be suppressed. But if the total is published as 80, the complement (73) is also at risk because the released total allows the suppressed cell to be estimated. Suppress both or round to the nearest 5.

---

## IDP-Specific Output Controls

### Brain Volumes (SIENAX)

| Output | Threshold | Control |
|--------|-----------|---------|
| Mean total brain volume | n ≥ 50 | Standard |
| Mean normalised GM/WM | n ≥ 50 | Standard |
| VSCALING | n ≥ 50 | Round to 3 significant figures |
| Unnormalised volumes | n ≥ 100 | Top-code at 99th percentile; add DP noise ε = 1.0 |

Unnormalised volumes are more sensitive than normalised because they encode absolute head size, which is more identifying.

### White Matter Hyperintensities (BIANCA)

| Output | Threshold | Control |
|--------|-----------|---------|
| Mean WMH volume | n ≥ 30 | Top-code at 95th percentile before computing mean |
| WMH count (n with WMH > threshold) | n ≥ 10 | Primary + secondary suppression |
| Distribution (percentiles) | n ≥ 100 | Release only 25th, 50th, 75th percentiles |

WMH volume distributions in dementia cohorts are heavily right-skewed. An individual with a very large WMH volume (outlier) can shift the mean substantially, making them identifiable. Top-coding at the 95th percentile removes this risk.

### FreeSurfer IDPs (~1,000 scalars)

| Output | Threshold | Control |
|--------|-----------|---------|
| Per-ROI mean (any atlas) | n ≥ 50 per site | Apply DP noise ε = 1.0 per site contribution |
| Full covariance matrix | n ≥ 200 | Regularise; do not release raw covariance |
| Individual IDP row | Prohibited | Never release; this is individual-level data |
| PCA components of IDP space | n ≥ 100 | Release only top k components where k << n |

!!! warning "Covariance matrix risk"
    The full covariance matrix of 1,000 FreeSurfer IDPs has ~500,000 entries. Even for large cohorts, releasing the full covariance enables attribute inference (predicting unobserved IDPs from observed ones) and is not safe. Release only regularised, low-rank approximations.

### TBSS Diffusion IDPs (450 scalars)

| Output | Threshold | Control |
|--------|-----------|---------|
| Per-tract mean FA/MD | n ≥ 30 per site | Standard; DP noise if n < 50 |
| NODDI metrics (ICVF/ISOVF/ODI) | n ≥ 50 | DP noise ε = 1.0 |
| Individual tract profile | Prohibited | Individual-level data |

### Eddy Outlier Count

| Output | Threshold | Control |
|--------|-----------|---------|
| Mean outlier count | n ≥ 20 | Standard |
| Individual outlier count | Prohibited | Reveals per-subject motion profile |

### Motion Parameters (fMRI)

| Output | Threshold | Control |
|--------|-----------|---------|
| Mean framewise displacement | n ≥ 20 | Standard |
| Mean rotation/translation | n ≥ 20 | Standard |
| Individual motion trace | Prohibited | 6-DOF time series is identifying |

### SNR / CNR

| Output | Threshold | Control |
|--------|-----------|---------|
| Site mean SNR/CNR | n ≥ 20 | These reveal scanner characteristics |
| Individual SNR/CNR | n ≥ 20 | Aggregate with other metrics only |

---

## Log File Controls

Log files produced by all BRAID pipeline scripts contain subject identifiers embedded in directory paths (e.g. `WD:/data/study/sub-001/analysis`). They must be subject to the following controls:

1. **Never transmitted outside the TRE** under any circumstances
2. Log files must be listed as a **prohibited output** in any data sharing agreement
3. Automated log scrubbing should remove subject ID patterns before any log archiving system that communicates outside the TRE perimeter

---

## Registration Matrices and Spatial Data

Registration matrices (`T1_to_MNI_linear.mat`, `T1_to_MNI_nonlin_coeff.nii.gz`) and Jacobian maps encode individual brain morphology and are **prohibited from release** under all circumstances. They are essentially a compact representation of the individual's brain shape and constitute biometric data.

---

## SACRO and Automated Output Checking

The **SACRO** (Statistical and Automated Release for Output Checking) framework, developed for UK TREs, provides automated checking of outputs against disclosure rules. SACRO checks include:

- Minimum cell counts
- Suppression of near-identifying combinations
- Detection of high-leverage individuals (those who substantially influence aggregate statistics)

For BRAID federated deployments, SACRO or an equivalent automated output checker should be integrated into the federated aggregation pipeline as the final gate before any statistic leaves the TRE.

---

## Output Submission Checklist

Before any BRAID-derived statistic is transmitted to a federated aggregator or released as a study output, the following must be confirmed:

- [ ] Cohort size meets minimum threshold for the specific IDP type
- [ ] No individual-level rows are present in any transmitted file
- [ ] Subject identifiers have been removed or stripped from all files (not just pseudonymised)
- [ ] Top-coding has been applied to skewed distributions (WMH, outlier IDPs)
- [ ] DP noise has been applied to site-level contributions where n < 100
- [ ] Log files are excluded from all transmissions
- [ ] Registration matrices and spatial images are excluded
- [ ] SACRO or equivalent automated check has passed
- [ ] A DPUK data analyst (or site-equivalent) has signed off the output (Safe Outputs, Safe 5)

---

## References

- UK Office for National Statistics (2023). *Disclosure Control for Administrative Data*. ONS Methodology. [https://www.ons.gov.uk/methodology/methodologytopicsandstatisticalconcepts/disclosurecontrol](https://www.ons.gov.uk/methodology/methodologytopicsandstatisticalconcepts/disclosurecontrol)
- Hundepool, A. et al. (2012). *Statistical Disclosure Control*. Wiley. [https://doi.org/10.1002/9781118348239](https://doi.org/10.1002/9781118348239)
- DPUK Data Provider Handbook (2024). Dementias Platform UK. [https://portal.dementiasplatform.uk](https://portal.dementiasplatform.uk)
- NHS Digital (2023). *Data Security and Protection Toolkit*. [https://www.dsptoolkit.nhs.uk/](https://www.dsptoolkit.nhs.uk/)
- SACRO: Statistical Controlled Automated Release Output. UK TRE Community, 2023. [https://github.com/AI-SDC/SACRO-ML](https://github.com/AI-SDC/SACRO-ML)
- Domingo-Ferrer, J. & Torra, V. (2001). A quantitative comparison of disclosure control methods for microdata. *Confidentiality, Disclosure, and Data Access*.
- Wachinger, C. et al. (2015). BrainPrint. *NeuroImage*, 109, 232–248. [https://doi.org/10.1016/j.neuroimage.2015.01.007](https://doi.org/10.1016/j.neuroimage.2015.01.007)
