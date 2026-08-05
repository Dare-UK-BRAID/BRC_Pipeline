# Federated NeuroImaging Preprocessing Pipelines Requirements

The BRAID pipelines are designed to operate within Trusted Research Environments (TREs), isolated compute enclaves where raw neuroimaging data never leaves the site. When these pipelines are deployed across multiple TREs under a **federated analytics** or **federated learning (FL)** model, a new class of privacy risk emerges: information encoded in the **outputs** of the pipeline (imaging-derived phenotypes, model parameters, gradients, or summary statistics) may be transmitted to a central aggregator, and that transmission can leak private participant information even when raw images remain local.

This section documents the privacy requirements, disclosure risks, and recommended controls for deploying the BRAID neuroimaging pipelines in a federated or multi-site analysis framework, with reference to the [DPUK Five Safes framework](https://portal.dementiasplatform.uk) and current literature on neuroimaging privacy.

---

## What is Federated Neuroimaging?

In a standard federated analytics architecture applied to neuroimaging:

1. Each participating TRE runs the BRAID pipelines locally on its cohort data
2. Each site computes **local statistics or model updates**, typically IDP summary statistics, regression coefficients, or neural network gradients
3. Only these **derived outputs**, not raw images, are transmitted to a central aggregator
4. The aggregator combines results across sites (e.g. via federated averaging) and returns a global model or summary

This architecture maps directly onto the DPUK model: data never leaves the secure environment (Safe Settings), but the outputs that do leave must pass through Safe Outputs checks.

!!! warning "The critical risk"
    The outputs of these pipelines, particularly the **IDP files** (`IDPs.txt`) and **FreeSurfer parcellation tables**, contain hundreds to thousands of scalar phenotypes per participant. These are not "safe" in the sense of being aggregate statistics. They are **individual-level data** and are subject to the same re-identification and disclosure risks as the source images.

---

## Federated Compatibility at a Glance

Not all pipelines can be federated without modification. The table below summarises both the egress risk of each pipeline's outputs and whether the pipeline is architecturally compatible with federated deployment as currently implemented.

| Pipeline | Federable As-Is | Egress Candidate | Risk Level |
|----------|:---:|-----------------|------------|
| Structural (T1/T2) | **Yes** | Registration matrices, Jacobian maps | **Medium** |
| Diffusion (dMRI) | **Yes** | TBSS IDPs: FA/MD/ICVF/ISOVF/ODI × 50 ROIs | **High** |
| Perfusion (ASL/pCASL) | **Yes** | CBF maps, kinetic model parameters | **Medium** |
| IDP Extraction | **Yes*** | `IDPs.txt`, `FS_IDPs.txt` (~1,000+ scalars) | **Critical** |
| Functional (fMRI preprocessing) | **Yes** | Motion parameters, tSNR, ICA components | **Medium** |
| Functional Group Analysis | **No** | MELODIC ICA maps, FC matrices, dual regression outputs | **High** |

\* IDP Extraction is per-subject and can run locally, but the output files require egress controls before transmission, see [Egress Risk Analysis](egress.md).

!!! warning "Functional Group Analysis is not directly federable"
    The MELODIC → dual regression → FSLNets pipeline requires all subjects' preprocessed fMRI data simultaneously. It cannot be run site-locally and aggregated. See [Egress Risk Analysis, Dataset-Wide Computations](egress.md#dataset-wide-computations-federated-architecture-constraints) for details and a proposed redesign path.

---

## Regulatory and Governance Context

Federated deployments of BRAID pipelines must satisfy:

- **DPUK Five Safes**, Safe People, Safe Projects, Safe Settings, Safe Data, Safe Outputs
- **UK GDPR / Data Protection Act 2018**, neuroimaging IDPs are likely special category data (health data) under Article 9
- **ISO 27001**, information security management (required by DPUK/SeRP infrastructure)
- **NHS Data Security and Protection Toolkit**, for NHS-linked cohorts
- **DEA Accreditation**, DPUK operates under Digitial Economy Act accredited infrastructure

The DPUK Data Access Process includes an **AI Triage** stage specifically to assess the risk of data disclosure in AI/ML outputs before release (step 6 of the Data Access Process). Federated learning model updates and IDP-based analyses must pass this triage.

---

## Section Overview

| Topic | Description |
|-------|-------------|
| [Egress Risk Analysis](egress.md) | Code-level audit of specific variables, files, and metadata at risk of leaking participant information |
| [Membership Inference Attacks](membership-attacks.md) | How an adversary can determine whether a specific individual was in a training cohort |
| [Differential Privacy](differential-privacy.md) | Formal privacy guarantees and how to apply DP noise calibration to BRAID IDPs |
| [Secure Aggregation & MPC](secure-aggregation.md) | Cryptographic protocols that protect individual site updates during aggregation |
| [Disclosure Thresholds](thresholds.md) | Minimum cohort sizes, suppression rules, and output controls for IDP release |
| [Compliance Checklist](compliance.md) | Governance and compliance requirements before federated deployment |

---

## References

- Finn, E.S. et al. (2015). Functional connectome fingerprinting: identifying individuals using patterns of brain connectivity. *Nature Neuroscience*, 18(11), 1664–1671. [https://doi.org/10.1038/nn.4135](https://doi.org/10.1038/nn.4135)
- Wachinger, C. et al. (2015). BrainPrint: A discriminative characterization of brain morphology. *NeuroImage*, 109, 232–248. [https://doi.org/10.1016/j.neuroimage.2015.01.032](https://doi.org/10.1016/j.neuroimage.2015.01.032)
- DPUK Data Provider Handbook (2024). Dementias Platform UK, Swansea University. [https://portal.dementiasplatform.uk](https://portal.dementiasplatform.uk)
- UK Five Safes Framework. Health Data Research UK (HDRUK). [https://ukdataservice.ac.uk/help/secure-lab/what-is-the-five-safes-framework/](https://ukdataservice.ac.uk/help/secure-lab/what-is-the-five-safes-framework/)
- McMahan, B. et al. (2017). Communication-efficient learning of deep networks from decentralized data. *AISTATS 2017*. [https://arxiv.org/abs/1602.05629](https://arxiv.org/abs/1602.05629)
- Kaissis, G. et al. (2021). End-to-end privacy preserving deep learning on multi-institutional medical imaging. *Nature Machine Intelligence*, 3, 473–484. [https://www.nature.com/articles/s42256-021-00337-8](https://www.nature.com/articles/s42256-021-00337-8)
- Melis, L. et al. (2019). Exploiting unintended feature leakage in collaborative learning. *IEEE S&P 2019*. [https://doi.org/10.1109/SP.2019.00029](https://doi.org/10.1109/SP.2019.00029)
- Dwork, C. & Roth, A. (2014). The algorithmic foundations of differential privacy. *Foundations and Trends in TCS*, 9(3–4), 211–407. [https://doi.org/10.1561/0400000042](https://doi.org/10.1561/0400000042)
