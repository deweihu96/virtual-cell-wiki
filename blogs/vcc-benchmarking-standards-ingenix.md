---
title: "The Virtual Cell Challenge: Towards Better Standards of Single Cell Perturbation Prediction Benchmarking"
author: "Joanna Krawczyk and Remek Kinas (Ingenix.ai)"
source: https://ingenix.substack.com/p/the-virtual-cell-challenge-towards
date: 2025-12-15
topic: virtual-cell-challenge
tags: [practical, benchmarking, data-integration, validation-design, perturb-seq]
---

Competitors' writeup on data understanding and offline-validation design for the Arc
Virtual Cell Challenge. Heavy on *how to not fool yourself* when the public leaderboard
allows only one submission per day. Not peer-reviewed.

## The challenge (as described)

- Predict transcriptomic changes when a gene is silenced via **CRISPRi** in **H1 human
  embryonic stem cells (hESCs)**.
- Judged on **context generalization**: predict perturbation effects for cellular
  contexts absent from training.
- Dataset: ~300k scRNA-seq profiles, 300 CRISPRi perturbations, >50k UMIs/cell.
  Split — train 150 perturbations / val 50 / test 100.
- Arc tooling used: **pdex** (parallel differential expression) and **cell-eval**
  (standardized prediction evaluation).

## Why H1 hESCs

Deliberately chosen as a **true distributional shift** from the immortalized cell lines
that dominate existing perturbation datasets — so models can't win by memorizing prior
patterns. No public dataset profiles H1 hESCs; the closest proxy (Feng et al., 19
pluripotent lines) was judged insufficient.

## Practical findings

- **Perturbation strength categories** (by number of DEGs): negligible (<100), subtle
  (<1,000), strong (>1,000, up to 11,779). The authors **could not reproduce the
  competition's original thresholds** — measurement varies by analysis method, so don't
  trust a single DEG cutoff.
- **Strong-perturbation cluster**: eight genes (DHX36, DNMT1, KDM1A, PRDM14, RNGF,
  SMARCA4, SMARCA5, SOX2) each drive ~10,000 DEGs and form a distinct cluster.
- **No significant batch effects** in training data (Chi² contingency testing).
- **Cross-dataset transfer is weak**: expected moderate agreement in DEGs / expression-
  delta signatures across cellular contexts, found minimal correlation. Perturbation
  responses are **highly context-dependent**; knowledge transfer across cell types is
  hard. This was the headline lesson.

## Validation-design lesson (the useful part)

With only **one leaderboard submission per day**, offline validation design is
everything. What worked best for them:

- **Don't** copy Arc's effect-size binning to pick internal validation genes.
- **Do** use **CRISPR dependency profiles from DepMap** to split the 150 training genes
  into core-essential / contextual-essential / non-essential, then **stratified-sample**.
  This captured biological diversity better and tracked leaderboard performance more
  faithfully.

## External data they tried

CellxGene (scRNA-seq), ARCHS4 + GTEx (bulk), Tahoe-100M + sci-Plex (chemical
perturbations), and five external Perturb-seq datasets. Conclusion: building a unified
high-quality dataset was "crucial" but exposed **fundamental transferability limits** —
other cell types gave limited guidance for H1 hESC predictions.
