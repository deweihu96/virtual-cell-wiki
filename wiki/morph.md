---
model: MORPH
paper_title: "MORPH Predicts the Single-Cell Outcome of Genetic Perturbations Across Conditions and Data Modalities"
authors: He, Zhang, Dahleh, Uhler
year: 2025
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: multi
organism: human
links:
  paper: https://www.biorxiv.org/content/10.1101/2025.06.27.661992v1
  code: https://github.com/uhlerlab/MORPH
  weights: not stated
---

# MORPH

## TL;DR
A conditional variational autoencoder with a cross-attention head over a learned
gene-program matrix that predicts single-cell responses to unseen genetic
perturbations, trained on unpaired control/perturbed data via an MMD discrepancy
loss.

## What it is
Encoder-decoder VAE. Input: (control cell x, perturbation embedding v^g). The control
cell is embedded by one MLP encoder and the perturbation embedding by a second MLP
encoder; the two latent vectors are concatenated as a query into a cross-attention
block whose keys/values come from a learned gene-program matrix G. The decoder maps
the attended latent back to a predicted perturbed cell. Output modality is either
scRNA (~5000 highly variable genes) or extracted image features. Same architecture is
used for both.

## Training data
No pretraining corpus; trained per dataset:
- **Replogle K562 (essential)**: ~2,033 CRISPRi perturbations, >310k cells, 5000 HVGs.
- **Replogle K562 (genome-wide)**: 9,823 perturbations, >1.9M cells.
- **Replogle RPE1 (essential)**: 2,264 perturbations, >240k cells.
- **Norman K562**: 234 single- and double-gene perturbations, ~110k cells.
- **Dixit BMDCs**: used for interpretability analysis (supplemental).
- **Carlson optical pooled screen (HeLa-TetR-Cas9, Ebola virus)**: 5,230,322 cells,
  20,336 guide-RNA targets, 6 imaging channels, 4 infection-state labels.
- Perturbations with fewer than 16 cells are filtered out.

## Core mechanism
The distinguishing idea is a **cross-attention block over a learned latent gene-program
matrix G**, motivated by a bipartite regulatory-network model of perturbations →
programs → genes. G is shared across cells of the same type and fixed at inference,
so attention scores modulate which programs a given perturbation activates. Combined
with an MMD discrepancy loss between predicted and observed perturbed-cell
distributions (rather than paired matching), this makes the model:
1. modality-agnostic (swap the encoder/decoder MLPs for image features),
2. amenable to bipartite GRN readout via attention manipulation, with a formal
   causal-disentanglement identifiability guarantee under mild assumptions
   (Supplemental Note 4).

The paper explicitly separates a "full" variant with residual connections in the
attention layers (better prediction) from a "no-residual first-layer" variant
(cleaner GRN readout) — a documented interpretability/accuracy trade-off.

## Implementation (paper)
- Input representation: expression is total-count normalized to 1e4 then log1p; top
  5000 highly variable genes selected per dataset. For imaging: 3 channels (DAPI,
  VP35 RNA, VP35 protein) fed through a ViT (ImageNet-21k pretrained), fine-tuned via
  supervised contrastive loss on the 4-state auxiliary classification, then features
  are extracted per cell.
- Vocabulary: perturbation is represented by a prior-knowledge feature vector v^g,
  not a gene token. Four priors evaluated: control-cell expression (1500-dim, sampled
  cells), Geneformer second-to-last-layer averaged over control cells (256-dim),
  GenePT text embeddings from NCBI (1536-dim), NCBI+UniProt (3072-dim), and STRING
  (1536-dim), and DepMap cell-viability columns (1100-dim). DepMap wins consistently.
- Architecture: control encoder MLP → latent dim 50; perturbation encoder MLP → latent
  dim 50; concat as query; 2 cross-attention layers with residual + feed-forward
  (standard transformer block); learned gene-program matrix `G` of shape [504, 50]
  for essential-scale, [100, 50] for genome-wide; decoder MLP. For genome-wide K562,
  encoder latent dims are 200 instead of 50.
- Training objective: L = E[α·L_recon + β·L_KL] + γ·L_discrep, where L_recon is L2
  reconstruction of the control cell, L_KL is the standard VAE KL to a standard
  Gaussian, and L_discrep is a sum of MMDs (Gaussian kernel, averaged over multiple
  bandwidths) between predicted and observed perturbed distributions per gene, plus
  λ·MMD on the control distribution. Selected values: α=2, β=2, γ=0, λ=1 for
  scRNA-seq essential-scale; α=1, β=2, γ=1, λ=0 for genome-wide K562; α=2, β=1e-4,
  γ=0, λ=1 for imaging.
- Optimizer and schedule: Adam, learning rate 1e-4, batch size 32; early stopping if
  validation loss does not improve for 20 epochs.
- Preprocessing: total-count normalization (1e4) + log1p; 5000 HVGs via scanpy's
  highly_variable_genes; perturbations with <16 cells filtered.
- Fine-tuning / adaptation: cross-cell-line transfer works by training on all
  perturbations in cell line 1, then fine-tuning on only control cells (or optionally
  a small subset of perturbations) from cell line 2 by minimizing the same objective
  (paper reports RPE1→K562 with 100 fine-tune perturbations reaches performance of
  training on 1500 K562 perturbations from scratch).
- Inference: given a control cell and v^g, run the encoders + attention + decoder to
  get one predicted perturbed cell; sample many control cells to build a predicted
  perturbed distribution.
- Compute: not stated (no GPU-hour or hardware budget reported).
- Reproducibility: full code released on GitHub; hyperparameter search space and
  selected values reported in Supplemental Note 1; per-dataset hyperparameters
  documented.

## Implementation (code)
Not pulled. The paper's Supplemental Note 1 already enumerates the tuned search space
(attention layers ∈ {1,2,3}, latent dims ∈ {50, 200}, gene-program matrix shape
∈ {[20,50], [50,50], [100,50]}, α ∈ {1,2}, β ∈ {1e-4, 2}, γ ∈ {0, 1}, λ ∈ {0, 1}, LR
1e-4, batch 32) and selected values per dataset, so the paper is already
reproducibility-grade. Code repo is <https://github.com/uhlerlab/MORPH>.

## Evaluation
Setup:
- **Splits**: (i) standard 5-fold cross-validation over perturbations, and (ii) an
  "outlier distribution split" that holds out the 5 Leiden clusters of
  perturbation pseudobulks farthest from control (stress-tests generalization).
- **Metrics**: MMD (full-distribution), RMSE of feature means, Pearson correlation of
  delta-expression vs control, and fraction of genes with correct sign of change.
  Reported on top-50 DE genes per perturbation and on all genes. For imaging, L1 loss
  between predicted and observed 4-state distribution.
- **Baselines**: (a) control distribution (no perturbation effect), (b) mean-shift
  transfer (for cross-line), (c) GEARS (GNN, [Roohani 2024]), (d) linear model from
  [Ahlmann-Eltze 2024] adapted to use DepMap embeddings, (e) SALT additive
  double-perturbation model.

Findings the paper reports:
- MORPH beats GEARS, linear-DepMap, and the control baseline on all four metrics
  across K562/RPE1 essential and genome-wide, in both 5-fold and outlier splits.
- DepMap prior wins consistently over Geneformer, GenePT variants, and raw control
  expression; a mixture-of-experts head combining priors offers no substantial gain
  (Supplemental Table 1).
- On Norman doubles, MORPH improves MMD by 33% and RMSE by 22% over the best
  baseline in the mixed 0/1/2-unseen setting; on 5-fold splits stratified by
  interaction type, MORPH's largest gains are on redundant (~40% over best SOTA) and
  potentiation interactions.
- Cross-line transfer: RPE1-trained model fine-tuned on K562 controls only
  ≈ K562-scratch on 100 perturbations; RPE1-trained + fine-tuned on 100 K562
  perturbations ≈ K562-scratch on 1500 perturbations (~7% of data).
- Active learning with prediction-loss-maximization on MORPH's learned latent beats
  random and prior-knowledge-latent selection; MORPH-adaptive beats GEARS-adaptive
  on both MMD/RMSE and top-25/top-35 impact-perturbation precision.
- On the Ebola optical pooled screen, MORPH beats the linear-DepMap baseline by 33%
  on the top-ranked (most impactful) perturbations.
- On simulated data with a known bipartite regulatory network, MORPH's attention
  weights recover perturbation modules, gene programs, and the sign of regulatory
  edges (activation vs inhibition).

## Claims (authors)
- The authors report MORPH outperforms GEARS and a linear DepMap baseline on
  single-gene perturbation prediction across K562 essential/genome-wide and RPE1.
- The paper claims cross-cell-line perturbation effects can be transferred by
  fine-tuning only on control cells of the target line, and that transfer quality
  correlates (r=0.52) with pathway-similarity between source and target lines.
- The authors report MORPH is uniquely robust on non-additive gene-interaction
  types (redundant, potentiation) where SOTA methods degrade.
- The paper claims MORPH's learned latent produces a stronger active-learning
  acquisition signal than GEARS or a prior-based latent.
- The authors report the attention maps recover a bipartite gene regulatory network
  on both simulated data and Replogle Perturb-seq, with a formal causal-
  disentanglement identifiability guarantee (Supplemental Note 4).
- The paper claims modality-agnosticism: same architecture predicts Ebola infection-
  state distributions from an optical pooled screen, beating the control and linear
  baselines on high-impact perturbations.

## Limitations
- Author-stated: transfer quality drops for cell-line-specific pathways (e.g.
  erythroid differentiation between K562 and RPE1). Full-model attention maps are
  hard to interpret because residual connections mix perturbation and program
  signals; a no-residual variant is used for GRN readout at the cost of some
  prediction accuracy (Supplemental Note 3). The full-model gene-program readout is
  only approximate because two attention layers make combined effects non-linear.
- Reviewer: Compute (GPU-hours, hardware) is not stated. The main FM baseline is
  GEARS; the paper does not benchmark against expression FMs used for perturbation
  (scGPT, Geneformer, scFoundation) as end-to-end predictors — Geneformer appears
  only as a source of prior embeddings, not as a competing predictor. The "linear
  (DepMap)" baseline is an adaptation the authors introduced (the original
  Ahlmann-Eltze linear model uses PCA of target genes) — this leaves it unclear how
  much of MORPH's advantage is architecture vs the shared DepMap prior. The
  cross-line transfer story is only shown on RPE1↔K562, both from the same Replogle
  lab and both essential-gene panels; harder transfers (across labs, disease
  contexts) are not evaluated. The Ebola imaging benchmark reduces cell state to a
  4-way logistic-regression classifier fit on control cells; the ranking-based L1
  loss and the chi-squared-based de-noising step both favor methods that predict
  something-different-from-control on the same axis the baselines predict null.

## Take / open questions
The main open question is whether MORPH's advantage over GEARS survives against
end-to-end fine-tuned scRNA foundation models (scGPT, Geneformer, AIDO.Cell), which
this paper does not compare. Given that recent benchmarks ([[genbio-fm-perturbation]])
find interactome-derived embeddings dominate perturbation prediction, testing MORPH
with STRING/interactome priors in place of DepMap would be a natural next step.

## relates_to_prior
- target: GEARS (Roohani et al. 2024); axis: generalization to unseen and combinatorial
  perturbations; shown (run as a baseline, MORPH reports better MMD/RMSE/Pearson on
  K562/RPE1 single-gene and Norman double-gene tasks, and better active-learning
  acquisition performance).
- target: linear model from Ahlmann-Eltze et al. 2024; axis: linear-baseline
  sufficiency claim; shown but with a caveat (baseline was adapted by the authors to
  use DepMap embeddings; MORPH still wins).
- target: SALT (Gaudelet et al. 2024); axis: additivity assumption for double
  perturbations; shown (SALT included in the GI-type-stratified comparison; MORPH
  wins particularly on non-additive interaction types).
- target: scGen (Lotfollahi 2019), CPA (Lotfollahi 2023); axis: paired-latent
  perturbation modeling; asserted (positioned as prior work assuming additive
  effects and paired data; not run as a baseline).
- target: scGPT (Cui 2024), Geneformer (Theodoris 2023); axis: end-to-end FM
  perturbation prediction; asserted (Geneformer only appears as an embedding source,
  not as a competing predictor).
- target: Bunne et al. 2023 (neural OT for perturbation); axis: unpaired modeling;
  asserted (cited as prior distribution-matching approach; not run as a baseline).
- target: Zhang et al. 2024 (causal disentanglement identifiability); axis:
  theoretical grounding; foundational (paper reuses and adapts the identifiability
  guarantees; not a baseline).
