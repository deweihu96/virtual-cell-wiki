---
title: "The Virtual Cell Challenge: Towards Better Standards of Single Cell Perturbation Prediction Benchmarking — Part 2: Metrics"
author: "Joanna Krawczyk, Remek Kinas, Przemyslaw Pietrzak (Ingenix.ai)"
source: https://ingenix.substack.com/p/the-virtual-cell-challenge-towards-80b
date: 2026-01-16
topic: virtual-cell-challenge
tags: [practical, metrics, evaluation, des, pds, mae, perturb-seq]
---

Part 2 of the Ingenix series ([Part 1](vcc-benchmarking-standards-ingenix.md) was data +
validation design). This one dissects the three challenge **metrics** and where they can
be gamed or mislead. Not peer-reviewed.

## The three metrics

1. **MAE** — post-perturbation pseudobulk expression accuracy.
2. **DES** (Differential Expression Score) — recovery of differentially expressed genes.
3. **PDS** (Perturbation Discrimination Score) — ability to distinguish perturbations
   from one another.

## Reference numbers

- **Mean-baseline**: DES = 0.106, PDS = 0.514, MAE = 0.027.
- **Performance ceiling** (perturbation-stratified 50/50 split): DES = 0.427, PDS =
  0.943, MAE = 0.01 — and no leaderboard submission exceeded DES ≈ 0.38.
- **DepMap cosine-similarity baseline** (match genes by essentiality profile): DES =
  0.22, PDS = 0.62, MAE = 0.04 — comparable to the State model baseline. The DepMap
  signal is strong (echoes Part 1's validation-split finding).

## Where the metrics mislead (the useful part)

- **DES is inflated by trivial changes**: 56% of FDR-significant genes have |log₂FC| ≤
  0.25 — statistically significant but biologically negligible. Recovering these pumps
  DES without real signal.
- **Weak perturbations have unstable DEGs**: bootstrap (80% subsampling, 20 iterations)
  shows small/borderline expression changes are highly sensitive to sampling — so DEG-
  based scoring is noisy exactly where perturbations are weak.
- **MAE can be gamed**: top submissions posted **raw MAE > 10** (biologically
  implausible outputs) while still optimizing DES/PDS. High combined score ≠ sensible
  predictions.

## Proposed fix

**Weighted MSE (WMSE)** — DEG-aware weighting that prioritizes meaningful genes, so
models "learn the true underlying biological signal instead of collapsing to the mean"
on weak perturbations.
