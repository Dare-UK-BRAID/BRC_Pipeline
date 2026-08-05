# Differential Privacy

**Differential Privacy (DP)** provides a mathematically rigorous framework for bounding the amount of information about any individual participant that can be inferred from the outputs of a computation. Formally, a randomised mechanism M satisfies (ε, δ)-DP if for all pairs of adjacent datasets D and D' (differing in one participant's record), and for all possible outputs S:

$$P[M(D) \in S] \leq e^\varepsilon \cdot P[M(D') \in S] + \delta$$

For neuroimaging IDP data, DP provides the most principled defence against the membership inference and re-identification attacks described in [Membership Inference Attacks](membership-attacks.md). This page describes how DP noise should be calibrated and applied to BRAID pipeline outputs.

---

## Why Standard De-identification Is Insufficient

The current BRAID IDP pipeline strips direct identifiers (names, NHS numbers) through the DPUK Safe Data (Safe 4) process, and subject IDs in `IDPs.txt` are pseudonyms. However, pseudo-anonymisation does not satisfy DP and does not protect against:

- Re-identification through the **neuroanatomical fingerprint** (1,000+ FreeSurfer IDPs)
- Membership inference from **aggregate IDP statistics**
- Reconstruction via **gradient leakage** in federated learning

DP provides a formal guarantee that survives these attacks by bounding the per-individual contribution to any output.

---

## Key Concepts

### Privacy Budget (ε)

The parameter ε controls the **privacy-utility trade-off**:

- **ε = 0**: Perfect privacy (the output reveals nothing). Utility is zero.
- **ε < 1**: Strong privacy. Recommended for sensitive health data.
- **1 ≤ ε ≤ 10**: Moderate privacy. Common in practice for aggregate statistics.
- **ε > 10**: Weak privacy. Typically insufficient for special category health data.

For neuroimaging IDPs which are special category data under UK GDPR, an **ε ≤ 1.0** is recommended for individual-level contributions to federated aggregation.

### Sensitivity

The **sensitivity (Δf)** of a function f is the maximum change in f's output when any single participant is added or removed from the dataset. Sensitivity determines how much noise must be added.

For the BRAID IDP pipeline, sensitivity must be computed or bounded for each IDP individually, because the value ranges differ by many orders of magnitude (e.g. WMH volume in mm³ vs. hippocampal thickness in mm).

### Gaussian vs. Laplace Mechanism

- **Laplace mechanism**: Adds Laplace noise with scale λ = Δf/ε. Satisfies pure ε-DP. Preferred when δ = 0 is required.
- **Gaussian mechanism**: Adds Gaussian noise with σ = Δf·√(2 ln(1.25/δ))/ε. Satisfies (ε, δ)-DP. More practical for high-dimensional outputs; allows composition over many queries with tighter bounds.

For the ~1,000-dimensional IDP vector produced per site, the **Gaussian mechanism** is preferred because it allows better composition when multiple queries are answered.

---

## Applying DP to BRAID IDP Outputs

### Step 1: Define the Sensitivity of Each IDP

Sensitivity must be bounded based on the plausible biological range of each measure. The following table gives approximate ranges and sensitivities for key BRAID IDPs:

| IDP | Biological Range | Recommended Sensitivity (Δf) |
|-----|-----------------|-------------------------------|
| Total brain volume (SIENAX) | 800,000 – 1,600,000 mm³ | 800,000 mm³ |
| WMH volume (BIANCA) | 0 – 200,000 mm³ | 200,000 mm³ |
| Hippocampal volume (per hemisphere) | 1,500 – 5,000 mm³ | 3,500 mm³ |
| Cortical thickness (per ROI) | 1.0 – 5.0 mm | 4.0 mm |
| FA (TBSS per JHU ROI) | 0.0 – 1.0 | 1.0 |
| MD (TBSS per JHU ROI) | 0.0 – 3.0 × 10⁻³ mm²/s | 3.0 × 10⁻³ |
| ICVF (NODDI) | 0.0 – 1.0 | 1.0 |
| SNR reciprocal | 0.0 – 1.0 | 1.0 |
| Eddy outlier count | 0 – 500 | 500 |

!!! note "Sensitivity clamping"
    Before applying DP noise, clamp each IDP value to its expected biological range. Outliers outside the range may indicate preprocessing failure and should be set to NaN. Clamping also bounds sensitivity for free.

### Step 2: Aggregate Local Statistics Within the TRE

Compute within-TRE aggregate statistics (mean, variance, count) **before** applying DP noise. The DP mechanism is applied to these local aggregates, not to individual IDP rows.

```python
import numpy as np

def dp_mean(values, sensitivity, epsilon, delta=1e-5):
    """
    Compute DP mean using the Gaussian mechanism.
    
    Parameters:
        values: array of IDP values (clamped to biological range)
        sensitivity: per-record sensitivity (= range / n for mean)
        epsilon: privacy budget
        delta: DP delta parameter
    """
    n = len(values)
    true_mean = np.mean(values)
    
    # Sensitivity of mean is range / n (each record shifts mean by at most range/n)
    mean_sensitivity = sensitivity / n
    
    # Gaussian noise scale
    sigma = mean_sensitivity * np.sqrt(2 * np.log(1.25 / delta)) / epsilon
    
    noise = np.random.normal(0, sigma)
    return true_mean + noise
```

### Step 3: Noise Calibration Example

For hippocampal volume (sensitivity = 3,500 mm³) with n = 100 participants and ε = 1.0, δ = 10⁻⁵:

- Mean sensitivity = 3,500 / 100 = 35 mm³
- σ = 35 × √(2 × ln(1.25/10⁻⁵)) / 1.0 ≈ 35 × √(2 × 11.45) / 1.0 ≈ 35 × 4.78 ≈ **167 mm³**

A true hippocampal mean of ~3,000 mm³ with noise σ = 167 mm³ (5.6% noise) is acceptable for most neuroscience applications. With n = 30, σ increases to ~559 mm³ (18.6% noise), which is likely too high for meaningful federated analysis, reinforcing the minimum cohort size requirement.

### Step 4: Privacy Composition

When multiple IDP statistics are released in the same federated round, the total privacy budget consumed is the **composition** of individual budgets. Under basic composition, releasing k statistics each with budget ε costs k·ε total. For ~1,000 IDPs, this becomes prohibitive.

**Advanced composition** (Kairouz et al., 2015) provides tighter bounds: releasing k mechanisms each with (ε, δ) costs approximately (ε√(2k log(1/δ')), k·δ + δ') for any δ' > 0.

**Rényi Differential Privacy (RDP)** (Mironov, 2017) provides even tighter composition and is the recommended framework for practical implementation (e.g. via the `autodp` or `opacus` Python libraries).

---

## Federated Learning: DP-SGD

If BRAID IDPs are used as training features for a federated machine learning model, **DP-SGD** (Abadi et al., 2016) should be applied:

1. **Clip per-sample gradients** to L2 norm ≤ C (typically C = 1.0)
2. **Add Gaussian noise** with scale σ·C to the sum of gradients before aggregation
3. **Track the privacy budget** using moments accountant or RDP accounting

The noise multiplier σ should be set to achieve a target ε ≤ 1.0 over the planned number of training rounds and the expected number of participants per site.

---

## Limitations of DP in Neuroimaging

DP provides formal guarantees, but several practical challenges arise in neuroimaging:

- **High sensitivity for rare phenotypes**: Extreme WMH volumes or very small hippocampi require large noise additions that may obscure clinically meaningful variation
- **Correlated features**: The 1,000+ FreeSurfer IDPs are highly correlated (e.g. all cortical thickness measures within a hemisphere), so the effective privacy loss from releasing all of them is less than the worst-case composition bound suggests, but also less than a simple sum
- **Small site cohorts**: Many TREs in a federated neuroimaging network may have <50 participants. Small n dramatically increases the noise required for a given ε
- **Repeated queries**: Federated learning involves many rounds of gradient aggregation. Each round consumes privacy budget. The total training budget must be planned in advance

!!! warning "DP does not protect against all attacks"
    DP bounds the information leaked about any single individual in the aggregate output, but it does not prevent an adversary from learning general population statistics, which is the intended purpose of the analysis. DP noise also does not protect against compromised aggregators or man-in-the-middle attacks during transmission. Secure aggregation (see [Secure Aggregation & MPC](secure-aggregation.md)) is needed to address those threats.

---

## Recommended DP Configuration for BRAID Federated Deployments

| Setting | Recommended Value | Rationale |
|---------|------------------|-----------|
| Privacy budget ε | ≤ 1.0 per federated round | Strong privacy for special category health data |
| δ | ≤ 1/n² (where n = min site cohort) | Standard DP convention |
| Noise mechanism | Gaussian (RDP accountant) | Better composition for high-dimensional outputs |
| Gradient clipping norm C | 1.0 | Standard DP-SGD starting point |
| IDP clamping | Biological range per IDP (see table above) | Bounds sensitivity |
| Min cohort size before releasing stats | n ≥ 50 | Reduces sensitivity and noise requirement |
| Privacy accounting framework | Rényi DP (autodp / opacus) | Tight composition across rounds |

---

## References

- Dwork, C. & Roth, A. (2014). The algorithmic foundations of differential privacy. *Foundations and Trends in TCS*, 9(3–4), 211–407. [https://doi.org/10.1561/0400000042](https://doi.org/10.1561/0400000042)
- Abadi, M. et al. (2016). Deep learning with differential privacy. *ACM CCS 2016*. [https://doi.org/10.1145/2976749.2978318](https://doi.org/10.1145/2976749.2978318)
- Mironov, I. (2017). Rényi differential privacy. *CSF 2017*. [https://doi.org/10.1109/CSF.2017.11](https://doi.org/10.1109/CSF.2017.11)
- Kairouz, P. et al. (2015). The composition theorem for differential privacy. *ICML 2015*. [https://proceedings.mlr.press/v37/kairouz15.html](https://proceedings.mlr.press/v37/kairouz15.html)
- Kaissis, G. et al. (2021). End-to-end privacy preserving deep learning on multi-institutional medical imaging. *Nature Machine Intelligence*, 3, 473–484. [https://doi.org/10.1038/s42256-021-00337-8](https://doi.org/10.1038/s42256-021-00337-8)
- Domingo-Ferrer, J. et al. (2021). Privacy-preserving methods for neuroimaging data analysis. *NeuroImage*.
- McMahan, H.B. et al. (2018). A general approach to adding differential privacy to iterative training procedures. *arXiv:1812.06210*. [https://arxiv.org/abs/1812.06210](https://arxiv.org/abs/1812.06210)
