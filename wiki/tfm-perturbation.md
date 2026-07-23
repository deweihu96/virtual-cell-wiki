---
model: TFM-Perturbation
paper_title: "Tabular Foundation Models Are Competitive Cellular Perturbation Predictors Across Biological Scales"
authors: Palla et al. (Biohub)
year: 2026
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.64898/2026.06.28.735106
  code: https://github.com/royerlab/tfm-perturbation
  weights: not stated
---

# TFM-Perturbation

## TL;DR
A recipe, not a new model: take frozen general-purpose tabular foundation models
(TabICL, TabPFN) with no biology-specific pretraining, decompose the perturbation
response into scalar PCA regressions, and use in-context learning to match or beat
specialized single-cell perturbation predictors across cell, pseudobulk, and
organism scales.

## What it is
Not an architecture but an adaptation of off-the-shelf tabular foundation models
(TFMs) — Prior-Fitted Networks trained on synthetic data — to perturbation
prediction. Input is a tabular context set of labeled rows plus a query row;
output is a single scalar per forward pass. A perturbation response is predicted
by running the TFM once per PCA component of the target and inverting the PCA. The
same frozen backbone is reused unchanged across three task scales; only the
featurization of the "row" and the scalar target change.

## Training data
The TFMs themselves are used frozen and were pretrained by their original authors
on synthetic tabular datasets, not on any single-cell data. This paper does no
pretraining. Evaluation (not training) data:
- Cell level: OpenProblems cross-cell-type benchmark, 298K cells, 3 donors, 144
  drug conditions, 4 cell types (train on T cell, predict NK/B/Myeloid).
- Pseudobulk: five Perturb-seq datasets (HepG2, Jurkat, K562 Essential, K562
  Genome-Wide, RPE1 Essential); plus a genome-wide CRISPR screen in primary human
  CD4+ T cells (Zhu et al. 2025), ~11k knockouts, 2 donors.
- Organism: zscape zebrafish developmental atlas, ~2.7M cells, 28 genetic
  perturbations, 5 timepoints (18–72 hpf), 98 cell types.

## Core mechanism
The distinguishing idea is that perturbation prediction is fundamentally posterior-
predictive regression, and PFNs already amortize exactly that. Rather than building
biological inductive bias into an architecture, decompose the high-dimensional
response with PCA into independent well-conditioned scalar regressions, then let a
frozen tabular PFN solve each one in-context (one forward pass, no gradient update).
The claim is that the top-variance subspace of the target plus strong in-context
regression carries the signal, and that biology-specific pretraining is not required.
An ablation replacing PCA with NMF at matched rank actually beats PCA (+0.117),
supporting "any high-variance basis," not PCA specifically.

## Implementation (paper)
- Input representation: task-dependent "row" definition. Cell level: OT-matched
  control cell pair, feature = source cell PCA projection, target = one PCA
  component of matched target cell. Pseudobulk: one perturbation, feature = gene
  embedding of target gene (from PCA of transposed treatment-effect matrix) and/or
  PRESAGE multi-modal embeddings, target = one PCA component of the treatment
  effect δ. Zebrafish: one (perturbation, timepoint), 398-d intervention+phenotype
  feature vector, target = PCA component of 98-d cell-type composition.
- Output decomposition: PCA on the response matrix; pseudobulk dy typically 128
  (plateaus past ~48); dx optimal around 64; zebrafish dx = dy = 32.
- Architecture: no new architecture. Backbones are TabICLv2 and TabPFN 2.6, frozen.
  Both are transformer PFNs interleaving feature attention (columns) and sample
  attention (rows) plus a per-cell MLP.
- Training objective: none. Zero-shot in-context inference; pretrained weights frozen.
- Optimizer and schedule: not applicable (no fine-tuning, no test-time adaptation).
- Cell matching: PCA on pooled control cells, then optimal-transport plan (uniform
  marginals, squared-Euclidean cost in PCA space) pairing source and target control
  cells. Context set ~2,000–5,000 matched pairs. OT beats kNN matching (+0.07–0.11
  Pearson ∆).
- Preprocessing: per-benchmark. 5k-HVG panel (cell level, 2k-HVG variant also used);
  top-5k HVGs (CD4); ~9k-gene union panel (CD4 genome-wide). Pseudobulk = perturbed
  mean minus control mean.
- Inference: dy TFM forward passes per query (one per output component), each on a
  small tabular dataset (~100 rows, ~64 features, 1 target), then inverse PCA.
- Compute: not stated (no GPU-hours or hardware reported).
- Reproducibility: code released; uses published TabICLv2 and TabPFN 2.6 weights.

## Implementation (code)
From `src/tfm_perturbation/models/transcriptomics/tabicl_multiview.py` and
`tabicl_cellmatch.py`:
- TabICL regressor defaults: `n_estimators: 8`, `batch_size: 8`,
  `outlier_threshold: 4.0`, `feat_shuffle_method: "latin"`, `random_state: 42`,
  checkpoint `tabicl-regressor-v2-20260212.ckpt`, auto AMP/FA3/offload. Weights are
  auto-downloaded, not shipped.
- Cell-match pipeline default `n_pca: 50` for the shared control embedding.
- Multiview defaults: `x_pca_n_components: 128`, `y_pca_n_components: 128`; gene-
  embedding builder default `n_components: 64`. Farthest-point-sampling support-set
  selection default `n_pca: 16`.
- Decomposition options implemented: pca, ica, ica_fullrank, nmf, none (matches the
  geometry ablation in the paper).

## Evaluation
- Cell level (OpenProblems 2k-HVG, Donor 1, mean over NK/B/Myeloid): TabICL(OT)
  leads Pearson ∆ 0.570 and discrimination 0.795; TabPFN(OT) leads DE Spearman
  0.682; CatBoost(OT) leads DE recall 0.751. STACK trails on Pearson ∆ (0.474) but
  keeps an edge on direction match (0.491) and Pearson E-distance (0.679).
  Baselines: STACK, CatBoost.
- Pseudobulk (5 Perturb-seq, 5-fold CV): TabICL and TabPFN consistently best among
  learned models on rel-MSE/cosine/AUROC/Recall@10; scGPT near the Mean baseline,
  scLAMBDA intermediate. Bootstrap is an oracle upper bound. Baselines: PRESAGE,
  scGPT, scLAMBDA, CatBoost, Mean, Bootstrap.
- CD4 genome-wide (within-donor holdout, 2 donors x 5 folds): TabPFN is the only
  model with positive top-20-DE cosine (+0.108); all others cluster near Mean
  (-0.026). Absolute metrics modest (rel-MSE ~0.85), attributed to weak per-
  perturbation signal and noisy pooled guide-level pseudobulk.
- Zebrafish (strict timepoint holdout, 5-fold): TabPFN best (Spearman 0.808, R2
  0.704), CatBoost next (0.763 / 0.662), Prophet comparable R2 (0.651) but lower
  Spearman (0.533), TabICL trails on R2 (0.604). Baselines: Prophet, CatBoost, Mean.

## Claims (authors)
- The authors report that TabICL/TabPFN match or outperform every specialized
  baseline tested across all four settings, despite no biology-specific pretraining.
- The authors claim the load-bearing property is the top-variance subspace of the
  target, not PCA itself (NMF beats PCA; random-orthogonal rotation is a no-op;
  bottom-PCs collapse performance).
- The authors report TFMs are data-efficient: TabICL at 25% training perturbations
  often matches PRESAGE/CatBoost at 100%; farthest-point sampling recovers most of
  the full-context signal from a quarter of the perturbations.
- The authors assert the advantage comes from the in-context inference step rather
  than a reusable per-perturbation encoding.

## Limitations
- Author-stated: the study is described as preliminary, limited to applying and
  evaluating PFNs; it does not exploit the full predictive distribution or
  interpretability. In-vivo systems (CD4) show modest absolute accuracy, pointing to
  a need for models of cell- and sample-level variability.
- Reviewer: compute (hardware, wall-clock) is not stated, though the frozen-weights
  recipe makes this less critical for reproduction. scGPT and scLAMBDA are fine-
  tuned foundation models used here on pseudobulked data, which the authors note may
  disadvantage them ("potential inability to train on pseudobulked scRNA-seq") — the
  comparison may under-serve those baselines rather than reflect an architectural
  ceiling. STATE/PRESAGE-family newer models and CellFlow are cited but not all run
  as baselines in every setting. The cell-level pipeline assumes the cross-cell-type
  mapping is perturbation-invariant, valid only when the perturbation shift is small
  relative to the cell-type shift.

## Take / open questions
The provocative result is that a frozen synthetic-data regressor beats models
pretrained on millions of cells, but the win is mediated entirely by the PCA/OT
featurization the authors hand-built around it — so the "no biological inductive
bias" framing understates the biological engineering in the wrapper. Open question:
does the TFM advantage survive when baselines are given the same OT-matching and
PCA-decomposition scaffolding, i.e. is the gain the PFN or the recipe?

## relates_to_prior
- target: STACK; axis: generalization / in-context learning; shown (ran as baseline,
  TabICL(OT) beats it on Pearson ∆ and discrimination at cell level).
- target: PRESAGE; axis: data efficiency, scale; shown (ran as baseline on
  pseudobulk, TFMs beat it; TabICL at 25% data matches PRESAGE at 100%). Reuses
  PRESAGE's multi-modal feature set and benchmark protocol.
- target: scGPT; axis: value of large-scale single-cell pretraining; shown (ran as
  baseline, performs near Mean on pseudobulk) but caveated as possibly disadvantaged
  by pseudobulk input.
- target: scLAMBDA; axis: pretraining vs in-context regression; shown (ran as
  baseline, intermediate).
- target: Prophet; axis: functional-readout prediction; shown (ran as baseline on
  zebrafish, TabPFN beats it on Spearman, comparable R2).
- target: specialized-architecture paradigm generally; axis: necessity of biological
  inductive bias; asserted + partially shown (headline thesis; supported by the four
  benchmarks but not by matched-scaffold controls).
