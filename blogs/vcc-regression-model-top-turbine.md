---
title: "How did we get a regression model to the top of the Virtual Cell Challenge"
author: "Kris Szalay (Turbine AI)"
source: https://blog.turbine.ai/p/how-did-we-get-a-regression-model
date: 2025-12-09
topic: virtual-cell-challenge
tags: [practical, regression, feature-engineering, no-pretraining, submission-postmortem]
---

Turbine AI's postmortem on placing a plain **regression model** ("first described in
1970") near the top of the Arc Virtual Cell Challenge, beating foundation models. Core
thesis: **match model complexity to *data* complexity, not *problem* complexity.** Not
peer-reviewed.

## The reframe that drove everything

Perturb-Seq colonies are homogeneous, so a condition is effectively **~1 RNA-seq
datapoint, not one per cell**. That collapses the challenge to **~300 total effective
datapoints** across train/leaderboard/eval. They **ignored individual cells and predicted
cluster (pseudobulk) behavior** — using all single cells "adds little to no additional
information" for the cost.

## What they did

- Plain regression, **no pretraining, no foundation models**.
- Fed **external Perturb-Seq datasets (Replogle et al., Nadig et al.) as input features**.
- Weeks of dataset cleaning, alignment, and integration — the real work.
- Strong biological inductive biases + regularization over architectural complexity.

Result: **top-15**, and a top "generalist" model under the revised evaluation criteria.

## What didn't work

- Foundation models / pretraining: "little to no added value."
- Complex architectures without appropriate regularization.
- Treating the data as higher-dimensional than it effectively is.

## On the metrics (PDS)

PDS measures which true perturbation vector a prediction is closest to — but "closest" in
~20,000-dim space is ambiguous. They found **prediction magnitude mattered more than
directional accuracy** for scoring, so PDS can reward metric-gaming over biology. For
their own use they prefer **Pearson delta** on top DEGs. (Aligns with the Ingenix Part 2
metric critique.)

## Hard-won insights

1. **Data bottleneck dominates**: ~500k effective RNA-seq-like datapoints globally vs
   20k-gene targets → regularization > architecture.
2. **Relevant data beats volume**: ~50–100 points closest to the target assay beat
   massive pretraining sets.
3. **Single-cell mystique is overblown**: homogeneous colonies give little real
   single-cell heterogeneity; the value is condition multiplexing, not cellular
   granularity.
4. **Metrics are problematic**: high-dim scoring amplifies artifacts; application-layer
   validation (viability, cytokines, phenotypes) matters more than RNA predictions alone.
5. **Virtual cells remain incomplete**: the challenge didn't show robust cross-assay
   generalization, which a truly general virtual cell model needs.
