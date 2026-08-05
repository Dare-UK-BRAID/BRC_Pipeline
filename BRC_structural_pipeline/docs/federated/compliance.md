# Governance & Compliance

This page describes the governance requirements and compliance checklist for deploying BRAID neuroimaging pipelines in a federated or multi-site analytics framework. It maps requirements against the DPUK Five Safes framework, UK GDPR, and current NHS and research data governance standards.

---

## Regulatory Framework

### UK GDPR and Data Protection Act 2018

Neuroimaging data and imaging-derived phenotypes (IDPs) constitute **special category data** under Article 9 of UK GDPR (health data). This imposes elevated requirements for processing:

- **Lawful basis**: Processing requires both a standard lawful basis (Article 6) and a special category condition (Article 9(2)). For research, Article 9(2)(j) (scientific research) is typically applicable, subject to appropriate safeguards including pseudonymisation and data minimisation.
- **Data minimisation**: Only the specific IDPs approved in the data access application should be computed and aggregated. The full FreeSurfer IDP vector (1,000+ scalars) should not be transmitted if the study requires only, say, hippocampal volumes.
- **Purpose limitation**: IDPs computed for a federated study may not be repurposed for a different analysis without a separate legal basis.
- **International transfers**: If the federated aggregator is hosted outside the UK/EEA, a Transfer Impact Assessment (TIA) is required and standard contractual clauses (SCCs) or an adequacy decision must be in place.

### ISO 27001

DPUK operates within the **SeRP (Secure eResearch Platform)** infrastructure, which holds full ISO 27001 accreditation. Any federated aggregation service receiving outputs from DPUK-hosted pipelines must demonstrate equivalent information security management controls, or the transmission must be protected by secure aggregation protocols (see [Secure Aggregation & MPC](secure-aggregation.md)) such that the aggregator never accesses plaintext data.

### DEA Accreditation

The UK Digital Economy Act 2017 established an accreditation regime for organisations that access de-identified data for research and statistics. DPUK holds DEA accreditation. Federated partners must either hold their own DEA accreditation or operate under an agreement that extends DPUK's accreditation to the federated use.

### NHS Data Security and Protection Toolkit (DSPT)

For cohorts containing NHS-linked data, all participating TREs must meet the current NHS DSPT requirements. The DSPT mandates annual self-assessment against 10 standards covering data storage, access control, incident management, and staff training.

---

## DPUK Five Safes: Federated Mapping

The Five Safes framework must be satisfied for the federated deployment as a whole, not just for individual TRE access:

| Safe | Standard Requirement | Federated Extension |
|------|---------------------|---------------------|
| **Safe People** | Researchers sign Data Access Agreement (DAA) with their organisation | All site PIs and the central aggregator operator must sign DAAs; federated protocol must be named in the DAA |
| **Safe Projects** | Application demonstrates scientific validity and public benefit | Application must include the federated protocol, aggregation methodology, and intended use of aggregated outputs |
| **Safe Settings** | Data accessed only via TRE virtual desktop; cannot be removed by researchers | Raw data remains in each TRE; only approved summary statistics leave via the federated channel; the federated channel itself must be a controlled environment |
| **Safe Data** | Data de-identified before access | IDPs are pseudonymised but not anonymous; federated transmission must not increase re-identification risk beyond what is accepted in the approved study |
| **Safe Outputs** | All results checked by data analysts before release | Federated outputs (aggregated model weights, IDP statistics) must pass output checking at each site AND at the aggregator; SACRO or equivalent should be used |

---

## Data Access Agreement: Required Clauses for Federated Studies

A federated neuroimaging study using BRAID pipelines must include the following in the Data Access Agreement, in addition to DPUK standard clauses:

1. **Federated protocol specification**: Description of the aggregation methodology, including what is transmitted, by whom, to whom, and when
2. **Aggregator identity and jurisdiction**: Full legal name, registered address, and data protection registration of the entity operating the central aggregator
3. **Output controls**: Explicit specification of the disclosure thresholds and DP parameters applied before transmission
4. **Prohibited outputs**: List of files and data types that must never be transmitted (raw IDPs, log files, registration matrices, see [Egress Risk Analysis](egress.md))
5. **Breach notification**: Protocol for notifying DPUK and the ICO within 72 hours if a data breach occurs in the federated channel
6. **Audit rights**: DPUK retains the right to audit the aggregator's logs and security controls
7. **Retention and deletion**: Aggregated statistics may not be retained by the aggregator beyond the study end date without a separate legal basis

---

## AI Triage Requirements (DPUK Process)

Under the DPUK Data Access Process, AI/ML projects undergo an **AI Triage** (step 6). For federated BRAID analyses, the AI Triage should assess:

- [ ] What IDP features are used as model inputs?
- [ ] What is the minimum group size contributing to each training batch?
- [ ] Does the model architecture enable gradient inversion (e.g. linear models are more vulnerable)?
- [ ] What DP parameters (ε, δ, noise multiplier) are applied?
- [ ] Is secure aggregation used?
- [ ] What is the attack surface of the central aggregator?
- [ ] Has a privacy audit (e.g. empirical MIA test) been conducted on the proposed federated scheme?

Projects that cannot satisfactorily answer all of these questions during AI Triage should not be approved until the methodology is strengthened.

---

## Required Modifications to BRAID Pipelines for Federated Compliance

Based on the [Egress Risk Analysis](egress.md), the following code-level changes are required before the BRAID pipelines can be used in a compliant federated deployment. Changes are split into those affecting pipelines that **can** be federated as-is (with modifications) and those requiring **architectural redesign**.

### Pipelines Requiring Architectural Redesign

The **Functional Group Analysis pipeline** (MELODIC → dual regression → FSLNets) cannot be federated without replacing core algorithms:

| Component | Current implementation | Federated replacement required |
|-----------|----------------------|-------------------------------|
| MELODIC group ICA | Requires all subjects' 4D fMRI simultaneously | Federated ICA (e.g. CanICA across sites, or consensus ICA from site-local PCA outputs) |
| Dual regression | Depends on centralised group ICA maps | Site-local regression against pre-agreed group maps; maps must not themselves encode individual data |
| FSLNets GLM / `nets_glm` | Permutation test across full subject matrix | Federated GLM or meta-analysis approach using site-level summary statistics |

Until this redesign is implemented, the Functional Group Analysis pipeline must not be deployed in a federated context. Preprocessing of individual fMRI data (the `BRC_functional_pipeline`) is unaffected and remains federable.

### Per-Subject Pipelines (Modification Required, Not Redesign)

### 1. Strip Subject Identifiers Before Transmission (IDP Pipeline)

In `idp_extract_part_1.sh`, the subject ID is the first field of every row in `IDPs.txt`. A pre-transmission sanitisation step must remove or replace this column with a site-level pseudonym that cannot be linked back to an individual without the site's key.

```bash
# Current (non-compliant for federated transmission):
result="${Subject}"

# Required: use a site-specific pseudonym, or strip ID entirely
result="${SITE_ID}_${ROUND_ID}"
```

### 2. Apply Clamping Before Aggregation

All IDP values should be clamped to their biological plausible range before computing site-level statistics. This bounds sensitivity for DP noise calibration and removes preprocessing outliers.

### 3. Apply DP Noise to Site-Level Means

A DP noise injection step should be inserted after local IDP statistics are computed but before transmission. See [Differential Privacy](differential-privacy.md) for calibration parameters.

### 4. Exclude Prohibited Files from Transmission

The following files must be explicitly excluded from any federated transmission mechanism:

- `log/log.txt` (all pipelines)
- `IDPs.txt` (per-subject and group, the raw files, not aggregated statistics derived from them)
- `FS_IDPs.txt` (per-subject FreeSurfer file)
- `*.mat` registration matrices
- `*_warp*.nii.gz` warp field images
- `eddy_unwarped_images.eddy_outlier_report` (raw report, not the count IDP)
- `bianca_mask.nii.gz` and `final_mask.nii.gz`

### 5. Implement Minimum Cohort Size Gates

The federated orchestration layer should enforce minimum cohort sizes at the site level before soliciting contributions. Sites with fewer than the required minimum should be excluded from the current round and notified.

---

## Compliance Checklist

### Pre-deployment

- [ ] Data Access Agreement signed by all sites and aggregator, including federated clauses
- [ ] AI Triage completed and approved by DPUK
- [ ] Ethics approval covers the federated use of the data
- [ ] ISO 27001 / DSPT status confirmed for all participating TREs
- [ ] DEA accreditation confirmed or extended agreement in place
- [ ] Data Flow Map submitted to each site's Data Protection Officer
- [ ] Transfer Impact Assessment completed if aggregator is outside UK/EEA

### Technical (per site)

- [ ] Subject IDs stripped from all outputs before transmission
- [ ] IDP values clamped to biological ranges
- [ ] Minimum cohort size threshold met (n ≥ 50 for FreeSurfer IDPs)
- [ ] DP noise applied to site-level IDP means (ε ≤ 1.0)
- [ ] Secure aggregation protocol configured and tested
- [ ] Prohibited file list implemented at network egress
- [ ] SACRO or equivalent automated output check passing
- [ ] Site data analyst has reviewed and signed off the transmission

### Ongoing

- [ ] Privacy budget tracked across federated rounds (total ε ≤ agreed cap)
- [ ] Incident response plan in place and tested
- [ ] Annual security review of federated infrastructure
- [ ] Results cite dataset DOIs as required by DPUK (step 9 of Data Access Process)

---

## References

- DPUK Data Provider Handbook (2024). Dementias Platform UK. [https://portal.dementiasplatform.uk](https://portal.dementiasplatform.uk){target="_blank"}
- UK GDPR / Data Protection Act 2018. ICO guidance. [https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/){target="_blank"}
- NHS Digital (2023). *Data Security and Protection Toolkit*. [https://www.dsptoolkit.nhs.uk/](https://www.dsptoolkit.nhs.uk/){target="_blank"}
- UK Digital Economy Act 2017, Accredited researcher status and data sharing. [https://www.legislation.gov.uk/ukpga/2017/30/contents/enacted](https://www.legislation.gov.uk/ukpga/2017/30/contents/enacted){target="_blank"}
- ISO/IEC 27001:2022, Information security management systems. [https://www.iso.org/standard/82875.html](https://www.iso.org/standard/82875.html){target="_blank"}
- Health Data Research UK. *Five Safes Framework*. [https://www.hdruk.ac.uk/access-to-health-data/five-safes/](https://www.hdruk.ac.uk/access-to-health-data/five-safes/){target="_blank"}
- SACRO: Statistical Controlled Automated Release Output. UK TRE Community, 2023. [https://github.com/AI-SDC/SACRO-ML](https://github.com/AI-SDC/SACRO-ML){target="_blank"}
- Rieke, N. et al. (2020). The future of digital health with federated learning. *npj Digital Medicine*, 3, 119. [https://doi.org/10.1038/s41746-020-00323-1](https://doi.org/10.1038/s41746-020-00323-1){target="_blank"}
- Kaissis, G. et al. (2021). End-to-end privacy preserving deep learning on multi-institutional medical imaging. *Nature Machine Intelligence*, 3, 473–484. [https://doi.org/10.1038/s42256-021-00337-6](https://doi.org/10.1038/s42256-021-00337-6){target="_blank"}
