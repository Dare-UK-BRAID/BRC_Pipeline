# Secure Aggregation & MPC

**Secure aggregation** and **multi-party computation (MPC)** are cryptographic techniques that allow multiple parties (TRE sites) to collaboratively compute a function over their combined data without any party, including the central aggregator, seeing the individual inputs. In federated neuroimaging, these protocols protect individual site-level IDP statistics and model gradients from being observed during transmission and aggregation.

---

## Why Secure Aggregation is Needed

Differential Privacy (see [Differential Privacy](differential-privacy.md)) protects the **outputs** of the aggregation, i.e. what the aggregator releases. But it does not protect the **inputs**: if sites transmit their raw local IDP means, gradient vectors, or unprotected model updates, the aggregator (or an adversary who intercepts the transmission) can observe individual site contributions.

This is particularly important in the BRAID federated context because:

1. **Small TRE cohorts**: A site with 20 participants has a local IDP mean that differs only slightly from individual values, seeing the local mean is nearly as identifying as seeing the raw data
2. **Cross-site collusion**: In a federated network where one site is compromised, that site's operator can observe all transmissions it participates in
3. **Gradient inversion**: Unprotected gradient updates in a federated learning model can be inverted to reconstruct individual BRAID IDP values (Zhu et al., 2019)
4. **TRE network perimeter**: Even within accredited TRE infrastructure (e.g. DPUK/SeRP), transmissions between sites pass over networks that may not be fully under each TRE's control

---

## Secure Aggregation Protocol (Bonawitz et al., 2017)

The most widely deployed secure aggregation protocol for federated learning was developed by Bonawitz et al. at Google. It guarantees that the central server learns only the **sum** of site contributions, provided that a threshold number of sites participate and no coalition of sites plus server exceeds a collusion bound.

**Protocol overview:**

1. **Key agreement**: Sites establish pairwise secret keys using Diffie-Hellman key exchange
2. **Masking**: Each site *i* adds a pseudorandom mask s_{ij} (derived from the shared key with site j) to its local IDP update u_i, so the transmitted value is u_i + Σ_j s_{ij}
3. **Cancellation**: The masks are designed to cancel pairwise: site i adds +s_{ij} and site j adds -s_{ij}. When all site transmissions are summed, the masks cancel and the server recovers Σ_i u_i exactly
4. **Dropout handling**: If some sites drop out mid-round, a secret sharing scheme (Shamir, 1979) is used to reconstruct missing masks

The server **never sees** any individual u_i, only the aggregate sum.

**Applicability to BRAID:**

This protocol is directly applicable to federated IDP aggregation. Each site computes its local IDP statistics (mean, count), adds pairwise masks, and transmits the masked vector. The aggregator recovers the sum and divides by total N to obtain the global IDP mean, without observing any site's individual contribution.

---

## Homomorphic Encryption (HE)

**Homomorphic Encryption** allows computation on ciphertext such that decrypting the result yields the same answer as performing the computation on plaintext. In federated neuroimaging, partially homomorphic encryption (PHE) supports the aggregation of site IDP means without decrypting intermediate values.

**How it works for BRAID federated aggregation:**

1. The central aggregator generates a public/private key pair and distributes the public key to all sites
2. Each site encrypts its local IDP summary statistics under the public key: Enc(u_i)
3. Each site transmits Enc(u_i) to the aggregator
4. The aggregator computes the encrypted sum: Enc(Σ u_i) = Π Enc(u_i) (using additive homomorphism)
5. The aggregator decrypts with the private key to obtain Σ u_i

**The aggregator cannot observe individual u_i without factoring the encryption.**

**Relevant schemes:**

| Scheme | Type | Speed | Use Case |
|--------|------|-------|----------|
| Paillier (1999) | PHE (additive) | Fast | IDP mean aggregation |
| BFV / BGV | FHE (fully homomorphic) | Slow | Complex model aggregation |
| CKKS | Approx. FHE | Moderate | Neural network training |

For BRAID IDP mean aggregation (addition only), **Paillier encryption** is sufficient and computationally practical. For federated neural network training on IDP features, **CKKS** (which supports approximate arithmetic on real numbers) is preferred.

---

## Secure Multi-Party Computation (MPC) for TBSS and FreeSurfer IDPs

For computing statistics over the high-dimensional BRAID IDP space (1,000+ dimensions), standard MPC frameworks applicable to federated neuroimaging include:

- **SCALE-MAMBA / MOTION**: General-purpose MPC frameworks supporting arithmetic over real-valued inputs
- **SecureNN / FALCON**: Neural network inference over secret shares
- **PySyft with CrypTen**: Python-friendly MPC for PyTorch-based federated models

**SACRO and SACRO-ML**, the output checking tools developed for UK TRE environments, provide a layer of Safe Outputs enforcement that complements MPC by checking that no individual-level data is present in what the aggregator releases, even after secure aggregation.

---

## Trusted Execution Environments (TEEs)

A complementary approach to cryptographic MPC is to perform aggregation within a **Trusted Execution Environment** (TEE), such as Intel SGX or AMD SEV. TEEs provide hardware-enforced isolation: code running inside the enclave cannot be observed or tampered with even by a privileged OS or cloud provider.

In a federated BRAID deployment with TEEs:

1. The aggregator runs inside a TEE enclave
2. Sites verify the enclave's integrity via remote attestation before sending their local statistics
3. The enclave performs the aggregation and releases only the intended output (e.g. a DP-protected global IDP mean)

TEEs have been used in medical imaging federated learning (Kaissis et al., 2021) and are increasingly supported by major cloud providers (Azure Confidential Computing, AWS Nitro Enclaves).

---

## Communication Security for BRAID Federated Deployments

Beyond secure aggregation, the communication channel between TRE sites and the aggregator must be secured:

| Requirement | Implementation |
|-------------|---------------|
| Encryption in transit | TLS 1.3 minimum for all federated communications |
| Mutual authentication | Client certificates (mTLS) for site-to-aggregator connections |
| No logging of payloads | Aggregator infrastructure must not log decrypted site contributions |
| Network isolation | Aggregator endpoint accessible only to approved TRE IP ranges |
| Audit trail | All federated rounds logged with hashes of contributions (but not content) |

Under the DPUK/SeRP infrastructure, inter-site communication must comply with ISO 27001 information security standards. Any federated aggregation service should be accredited as part of the overall TRE system.

---

## Recommended Architecture for BRAID Federated IDP Aggregation

```
┌─────────────────────────────────────────────────────────┐
│  TRE Site A (DPUK SeRP)              TRE Site B          │
│  ┌──────────────────┐                ┌───────────────┐   │
│  │ BRAID Pipelines  │                │ BRAID Pipelines│  │
│  │ → IDPs.txt       │                │ → IDPs.txt    │   │
│  │ → Local mean u_A │                │ → Local mean  │   │
│  │ + DP noise       │                │ + DP noise    │   │
│  │ + SecAgg mask    │                │ + SecAgg mask │   │
│  └────────┬─────────┘                └──────┬────────┘   │
│           │  Enc(u_A + mask_A)              │             │
└───────────┼─────────────────────────────────┼────────────┘
            │                                 │
            ▼                                 ▼
     ┌──────────────────────────────────────────────────┐
     │  Central Aggregator (TEE or MPC environment)     │
     │  Receives: Enc(u_A + mask), Enc(u_B + mask)     │
     │  Computes: Σ Enc(u_i) → decrypts → Σ u_i / N  │
     │  Applies: SACRO output check                     │
     │  Releases: DP-protected global IDP statistics    │
     └──────────────────────────────────────────────────┘
```

---

## References

- Bonawitz, K. et al. (2017). Practical secure aggregation for privacy-preserving machine learning. *ACM CCS 2017*. [https://doi.org/10.1145/3133956.3133982](https://doi.org/10.1145/3133956.3133982){target="_blank"}
- Paillier, P. (1999). Public-key cryptosystems based on composite degree residuosity classes. *EUROCRYPT 1999*. [https://doi.org/10.1007/3-540-48910-X_16](https://doi.org/10.1007/3-540-48910-X_16){target="_blank"}
- Zhu, L. et al. (2019). Deep leakage from gradients. *NeurIPS 2019*. [https://arxiv.org/abs/1906.08935](https://arxiv.org/abs/1906.08935){target="_blank"}
- Kaissis, G. et al. (2021). End-to-end privacy preserving deep learning on multi-institutional medical imaging. *Nature Machine Intelligence*, 3, 473–484. [https://doi.org/10.1038/s42256-021-00337-6](https://doi.org/10.1038/s42256-021-00337-6){target="_blank"}
- Rieke, N. et al. (2020). The future of digital health with federated learning. *npj Digital Medicine*, 3, 119. [https://doi.org/10.1038/s41746-020-00323-1](https://doi.org/10.1038/s41746-020-00323-1){target="_blank"}
- SACRO: Statistical Controlled Automated Release Output. UK TRE Community, 2023. [https://github.com/AI-SDC/SACRO-ML](https://github.com/AI-SDC/SACRO-ML){target="_blank"}
