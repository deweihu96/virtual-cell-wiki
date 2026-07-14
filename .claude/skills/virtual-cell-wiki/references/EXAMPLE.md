<!--
Illustrative example only. ExampleNet is a fictional model invented to show the page
format and grain. None of the numbers below are real. Do not cite this page.
-->

---
model: ExampleNet
paper_title: "ExampleNet: a masked cell foundation model for expression prediction"
authors: Doe et al.
year: 2025
venue: preprint (arXiv)
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://arxiv.org/abs/0000.00000
  code: https://github.com/example/examplenet
  weights: not stated
---

# ExampleNet

## TL;DR
A transformer trained on masked expression prediction over single-cell RNA, adapted to
predict post-perturbation cell states via a fine-tuned head.

## What it is
A masked-language-model style transformer. Input is a single cell's expression profile
tokenized by gene; output is a per-cell embedding, or a predicted expression vector under a
given perturbation when the perturbation head is attached.

## Training data
Trained on a reported 12M human cells drawn from 380 public datasets. Gene coverage limited
to ~19k protein-coding genes. Spatial and ATAC modalities excluded. Cell-count breakdown per
tissue is not stated.

## Core mechanism
Cells are encoded as sets of gene tokens whose values are discretized into expression bins.
The model is pretrained to reconstruct masked expression bins, so it learns co-expression
structure without labels. The distinguishing choice versus prior masked cell models is a
perturbation-conditioning token prepended at fine-tune time.

## Implementation (paper)
- Input representation: expression values binned into 51 tokens; genes ordered by input;
  [CLS] token prepended
- Vocabulary: ~19k gene tokens plus special tokens
- Architecture: 12-layer transformer, hidden dim 512, 8 heads, ~50M params, context length
  1200 genes
- Training objective: masked expression-value prediction, masking ratio 0.15
- Optimizer and schedule: AdamW, peak LR 1e-4, linear warmup; batch size not stated
- Preprocessing: library-size normalization then log1p; 1200 HVGs selected per cell
- Fine-tuning: frozen embeddings + linear probe for annotation; full fine-tune for
  perturbation
- Inference: [CLS] embedding used as the cell representation
- Compute: not stated
- Reproducibility: inference code released; training code and configs not stated

## Evaluation
Benchmarked on a held-out perturbation set using Pearson delta and MMD, against GEARS and a
linear baseline. Did not compare against CPA or scGPT. Baseline selection and split
construction are described only briefly.

## Claims (authors)
- The authors report state-of-the-art Pearson delta on their held-out perturbation set.
- The paper claims the perturbation-conditioning token is the source of the gain, though no
  ablation removing it is reported.

## Limitations
- Author-stated: limited to the ~19k gene vocabulary; unseen genes cannot be represented.
- Reviewer: training compute and batch size are not stated, so the run is not reproducible
  from the paper. The masking-ratio-only ablation leaves the central claim (conditioning
  token drives the gain) unsupported. No comparison to CPA or scGPT despite both being
  standard baselines for this task.

## Take / open questions
Does the reported gain survive against CPA and scGPT, and does it hold when the conditioning
token is ablated? Both are answerable with the released inference code.

## relates_to_prior
- target: GEARS; axis: generalization; shown (ran as a baseline, reports higher Pearson delta)
- target: masked cell models generally; axis: biological fidelity; asserted (claims the
  conditioning token adds perturbation awareness, no direct comparison)
