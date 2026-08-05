# Membership Inference Attacks

A **membership inference attack (MIA)** is an adversarial technique in which an attacker, given a trained model or a set of aggregate statistics, determines whether a specific individual was present in the dataset used to produce those outputs. In the context of neuroimaging federated analytics, this translates to: *given the aggregated IDP statistics or model weights released from a TRE, can an adversary determine whether a specific patient was included in that site's cohort?*

This is the most practically relevant class of privacy attack for federated neuroimaging pipelines operating within the DPUK / Five Safes framework.

---

## Why Neuroimaging IDPs Are Particularly Vulnerable

### High Dimensionality and Uniqueness

As documented in the [Egress Risk Analysis](egress.md), the BRAID IDP pipeline extracts over 1,000 phenotypes per participant. High-dimensional phenotypic profiles are highly unique. An attacker who has access to a target individual's scan (e.g. from a clinical record or a public dataset) can compute that individual's IDP vector and compare it against aggregate IDP statistics released from a TRE to infer membership.

The **shadow model attack** (Shokri et al., 2017) is the canonical MIA method: an adversary trains a meta-classifier on models trained on known members vs. non-members, then applies it to the target model. For neuroimaging, the analogous attack exploits the fact that **aggregate statistics computed over a population including a highly anomalous brain (e.g. large WMH volume, extreme hippocampal atrophy) shift measurably** compared to a population without that individual.

### The Population Mean is Not Safe

Consider the IDP extraction aggregating TBSS FA values across 50 JHU ROIs for a cohort. If a TRE releases a site-level mean FA vector (the minimal sufficient statistic for a federated mean), an adversary with access to a target individual's FA values can compute the **likelihood ratio** of observing those population means given that the individual is included vs. excluded. For small cohorts (e.g. <30 participants), this test has substantial power.

---

## Attack Scenarios Specific to BRAID Pipelines

### Scenario 1: WMH Volume Attack

BIANCA computes a single WMH volume per subject (`volume.txt`). In a dementia cohort, WMH volumes are often highly skewed, a small number of participants have very large lesion loads. If a site releases the mean WMH volume and an adversary knows that a specific patient (whose scan is externally available) has extreme WMH, the presence of that patient in the cohort will shift the site mean. For cohorts of <50, this shift is detectable with standard statistical tests.

**Attack power:** High for outlier individuals (extreme phenotypes), moderate for typical values.

### Scenario 2: FreeSurfer Subfield Volume Attack

The hippocampal subfield volumes extracted by `gen_subsegmentation()` in `brc_FS_get_IDPs.py` produce 22 per-hemisphere values. In Alzheimer's disease cohorts, hippocampal CA1 subfield atrophy is a sensitive biomarker. An adversary who has run FreeSurfer on a target individual's clinical MRI can compare their subfield volumes against released aggregate statistics.

### Scenario 3: ICA Component Attack

In the functional pipeline, if ICA-AROMA spatial maps are transmitted (e.g. as part of a federated ICA analysis), adversaries can exploit the **spatial uniqueness of noise components** to fingerprint individuals. Demšar et al. (2021) showed that resting-state fMRI connectivity profiles identify individuals across sessions with >90% accuracy.

### Scenario 4: Gradient Leakage in Federated Learning

If BRAID IDPs are used as inputs to a federated machine learning model (e.g. a federated neural network predicting dementia risk), **gradient leakage** becomes a concern. Zhu et al. (2019) demonstrated that raw training data can be reconstructed from gradients with high fidelity. Since each local gradient update in a federated learning round depends on the local training data (i.e. individual IDPs), an aggregator or a compromised participant can reconstruct individual IDP values from unprotected gradient transmissions.

The risk is proportional to the **batch size**: single-sample gradients allow near-perfect reconstruction; larger batches provide partial protection. In small TRE cohorts, effective batch sizes may approach 1.

---

## Attack Surface in the Current Pipeline

| Code Location | MIA Attack Vector | Severity |
|--------------|-------------------|----------|
| `IDPs.txt` (group matrix) | Direct individual rows transmitted | **Critical** |
| `FS_IDPs.txt` per subject | Individual phenotype vector | **Critical** |
| Site-level mean IDP vector | Mean shift inference for outliers | **High** |
| FreeSurfer subfield volumes | Hippocampal fingerprint in AD cohorts | **High** |
| WMH volume (`volume.txt`) | Outlier detection in skewed distributions | **High** |
| TBSS FA/MD/ICVF per ROI | White matter fingerprint | **High** |
| Federated model gradients | Gradient inversion attacks | **High** |
| SIENAX brain volumes | Head size / brain volume membership | **Medium** |
| SNR/CNR metrics | Site fingerprinting enabling linkage | **Medium** |
| fMRI motion parameters | Temporal motion trajectory fingerprint | **Medium** |

---

## Defences Against Membership Inference

### 1. Minimum Cohort Size Thresholds

Do not release site-level IDP statistics derived from cohorts smaller than a minimum threshold. DPUK and the UK ONS recommend a minimum cell size of **10** for simple statistics; for high-dimensional IDP data, we recommend **n ≥ 50** per site before releasing any aggregate statistics (see [Disclosure Thresholds](thresholds.md)).

### 2. Differential Privacy (DP)

Adding calibrated noise to IDP statistics before transmission provides provable bounds on membership inference success. A DP mechanism with privacy budget ε guarantees that the adversary's advantage in distinguishing a population with vs. without a target individual is bounded by e^ε. See [Differential Privacy](differential-privacy.md) for implementation guidance.

### 3. Secure Aggregation

If individual site contributions must be combined, use **secure aggregation** protocols (e.g. Bonawitz et al., 2017) to ensure that the aggregator sees only the sum of contributions, never individual site updates. This eliminates scenario 4 (gradient leakage) entirely. See [Secure Aggregation & MPC](secure-aggregation.md).

### 4. Output Perturbation for Outliers

For univariate outputs like WMH volume where outlier individuals are at high risk, apply **top-coding** (cap at a threshold, e.g. 95th percentile) before contributing to federated aggregation. This suppresses the most identifiable values.

### 5. Gradient Clipping

In federated learning contexts, **clip gradients before aggregation** to a maximum L2 norm (typically 1.0–10.0). Gradient clipping is a prerequisite for applying DP noise and also independently reduces gradient inversion attack success.

### 6. AI Triage (DPUK Process)

Under the DPUK Data Access Process, AI/ML projects undergo an **AI Triage** (step 6) to assess the risk of data disclosure before any results are released. Federated learning outputs, including aggregated model weights, gradients, and IDP statistics, must pass this triage.

---

## Quantifying MIA Risk: The Empirical Approach

Before deploying a federated IDP analysis, the following empirical evaluation is recommended:

1. **Run a shadow model attack** on synthetic data with the same dimensionality and distribution as the BRAID IDPs to establish baseline MIA success rates
2. **Compute the per-feature sensitivity** (maximum change in the aggregate statistic when a single participant is added or removed), high-sensitivity features should be suppressed or receive more DP noise
3. **Measure the ROC-AUC of a membership classifier** trained on known members vs. non-members using the proposed aggregation scheme, a target AUC < 0.55 indicates adequate protection

---

## References

- Shokri, R. et al. (2017). Membership inference attacks against machine learning models. *IEEE S&P 2017*. [https://doi.org/10.1109/SP.2017.41](https://doi.org/10.1109/SP.2017.41)
- Zhu, L. et al. (2019). Deep leakage from gradients. *NeurIPS 2019*. [https://arxiv.org/abs/1906.08935](https://arxiv.org/abs/1906.08935)
- Carlini, N. et al. (2022). Membership inference attacks from first principles. *IEEE S&P 2022*. [https://doi.org/10.1109/SP46214.2022.9833649](https://doi.org/10.1109/SP46214.2022.9833649)
- Demšar, U. et al. (2021). Functional connectivity fingerprinting across multiple sites. *NeuroImage*.
- Bonawitz, K. et al. (2017). Practical secure aggregation for privacy-preserving machine learning. *ACM CCS 2017*. [https://doi.org/10.1145/3133956.3133982](https://doi.org/10.1145/3133956.3133982)
- Kaissis, G. et al. (2021). End-to-end privacy preserving deep learning on multi-institutional medical imaging. *Nature Machine Intelligence*, 3, 473–484. [https://doi.org/10.1038/s42256-021-00337-6](https://doi.org/10.1038/s42256-021-00337-6)
- Melis, L. et al. (2019). Exploiting unintended feature leakage in collaborative learning. *IEEE S&P 2019*. [https://doi.org/10.1109/SP.2019.00009](https://doi.org/10.1109/SP.2019.00009)
