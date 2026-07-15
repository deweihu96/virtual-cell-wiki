---
model: X-Cell
paper_title: "X-Cell: Scaling Causal Perturbation Prediction Across Diverse Cellular Contexts via Diffusion Language Models"
authors: Wang, Karimzadeh, Ravindra, Bounds, Alerasool, Huang et al. (Xaira Therapeutics)
year: 2026
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.64898/2026.03.18.712807
  code: https://github.com/xaira-therapeutics/x-cell
  weights: https://huggingface.co/Xaira-Therapeutics/X-Cell
---

# X-Cell

## TL;DR
A set-level diffusion transformer that predicts perturbed single-cell expression profiles from control cell sets, conditioned on multi-modal biological priors of the perturbed gene via cross-attention. Trained on a new 25.6M-cell genome-wide CRISPRi Perturb-seq corpus (X-Atlas/Pisces); scaled to a 4.9B-parameter variant (X-Cell-Ultra).

## What it is
A masked-diffusion transformer over sets of cells. Input is a set of S=64 control-cell expression profiles plus a partial (masked) reveal of the perturbed profile, together with prior-knowledge embeddings of the perturbed gene. Output is the full predicted perturbed expression profile for each cell in the set, refined over T=4 iterative diffusion steps.

## Training data
- X-Atlas/Pisces: 25.6M perturbed single-cell transcriptomes from genome-wide CRISPRi Perturb-seq across 16 cellular contexts (HCT116, HEK293T, HepG2, iPSC, Jurkat resting, Jurkat CD3/CD28 activated, iPSC Multi-Diff with 10 annotated lineages), >152k unique perturbation-context conditions. Median 25,478 UMIs and 6,708 genes per cell; median 78.7% target knockdown.
- X-Cell training corpus (after filtering, protein-coding targets, validation/hold-out exclusions): 10,165,929 cells, 36,816 perturbation-context conditions, 4 screens (HCT116, HEK293T, HepG2, iPSC).
- X-Cell-Ultra training corpus: 20,281,764 cells, 151,180 perturbation-context conditions, 7 screens. Phase-1 high-effect subset: 3,797,171 cells, 4,996 conditions.
- Organism: human. Modality: scRNA (Perturb-seq via FiCS and Flex protocols).

## Core mechanism
Two ideas separate X-Cell from prior single-cell foundation models. First, it is trained directly on interventional (CRISPRi Perturb-seq) data with a distribution-matching objective over sets of cells, not on observational atlases with per-cell reconstruction. Second, the generative process is a discrete masked-diffusion LM over gene-value tokens where the mask fraction is sampled uniformly from {0.25, 0.5, 0.75, 1.0}, and prior knowledge about the perturbed gene (text, protein sequence, PPI, dependency, morphology, scGPT embedding) is injected through cross-attention at every third layer. Inference is coarse-to-fine: 4 cumulative diffusion steps that progressively reveal predicted values as new input.

## Implementation (paper)
- Input representation: set of S=64 cells; per gene position g, token = learned gene identity embedding + continuous log1p-CP10K value encoded through a two-layer MLP (ReLU, LayerNorm) + learned mask-state embedding. Genes subsampled to context length G' ≤ 4,000 out of ≤19,408 protein-coding genes per minibatch. Learned [CLS] token prepended.
- Vocabulary: ~19,408 protein-coding gene tokens plus special tokens.
- Architecture (X-Cell): 12 layers, hidden dim 512, 55M parameters, Post-LN, ReLU FFN, 4 cross-attention layers (every 3rd + final). Initialized from scGPT encoder weights.
- Architecture (X-Cell-Ultra): 34 layers, hidden dim 2560, 4.87B parameters, LLaMA-style Pre-LN with RMSNorm, SwiGLU FFN (4d intermediate), no attention bias, QK-Norm; 11 cross-attention layers (every 3rd + final); tied embedding output projection with per-gene bias; trained from scratch (no scGPT init).
- Cross-attention context: 6 prior knowledge embeddings per perturbed gene (GenePT text, ESM-2 protein LM, STRING PPI, DepMap gene dependency, JUMP Cell Painting morphology, scGPT gene identity), each projected to d via a two-layer LeakyReLU + LayerNorm MLP; auxiliary autoencoder reconstruction loss on the projections.
- Training objective: sum of seven terms, equally weighted at 1.0 - MMD (energy distance via geomloss), fold-change CCC, per-gene CCC, delta-norm magnitude penalty (1-rho)^2, pseudobulk MSE on delta mu, InfoNCE contrastive on CLS embeddings (temperature 0.1), and prior-knowledge reconstruction MSE. X-Cell adds L1 output regularization at weight 1.0; X-Cell-Ultra sets L1 to 0 and drops per-gene CCC in phase 2. Masking ratio sampled per-sample from {0.25, 0.5, 0.75, 1.0}.
- Optimizer and schedule (X-Cell-Ultra): AdamW, phase 1 LR = 3e-4 with cosine decay and 10% linear warmup, 10 epochs on the 5% highest-effect perturbations, optimizer step size 128 (6,144 cells per update), 16 nodes / 128 H200 GPUs; phase 2 LR = 4e-5 (square-root batch scaling), fresh optimizer state, 3% warmup, full corpus. Gradient clipping norm 1.0. Mixed precision bf16, PyTorch FSDP2 per-layer sharding, activation checkpointing (TorchTitan ordering). ~41% MFU on 64-128 H200 GPUs.
- Optimizer and schedule (X-Cell 55M): fine-tuning uses AdamW, LR = 3e-4, 20 epochs, set size 64; base pre-training optimizer specifics not stated separately from X-Cell-Ultra.
- Preprocessing: raw counts stored, normalized during training with CP10K log1p. Cells filtered by expected dual-guide pair, per-GEM UMI/gene thresholds. Target perturbations restricted to protein-coding genes.
- Fine-tuning / adaptation: full fine-tune on downstream benchmarks (Replogle-Nadig, Parse-1M) at LR 3e-4 for 20 epochs. Optional test-time adaptation (TTA) for zero-shot cell types using MMD-only self-supervision on control-to-control set pairs (200-1024 sets of 64 cells at LR 1e-6-1e-5); self-attention layers adapted, cross-attention frozen and skipped during TTA then re-enabled at inference.
- Inference: 4 cumulative diffusion steps. Each step reveals 25% additional predicted values (clamped >=0) into the input; single forward pass at T=1 also supported (used during TTA).
- Compute: 16 nodes x 8 H200 = 128 H200 GPUs for X-Cell-Ultra; total GPU-hours not stated.
- Reproducibility: X-Cell (55M) weights released on Hugging Face; code and tutorials on GitHub; dataset released on Hugging Face under CC BY-NC-SA 4.0. X-Cell-Ultra (4.9B) weights not stated.

## Implementation (code)
Cloned from https://github.com/xaira-therapeutics/x-cell (repo size 1.8 MB, ~58 files). At time of read, the repo is a public stub: `src/xcell/model.py` exposes an `XCell` class whose `from_pretrained` and `predict` methods both raise `NotImplementedError` with the message "Full inference code is coming soon". `configs/` and `scripts/` contain only `.gitkeep`. There is no training code, no config yaml, and no released architecture code beyond docstrings.

What the code and MODEL_CARD add on top of the paper:
- Inference API defaults (`src/xcell/model.py`): `n_cells=64`, `n_diffusion_steps=4`, `batch_size=8`. Matches paper.
- Input contract: expression must be pre-normalized log1p CP10k; genes outside the X-Cell vocabulary are zero-imputed. Multiple `.h5ad` files are pooled via `ad.concat(..., merge="same")`.
- Only the 55M "mini" variant is exposed via `SUPPORTED_VARIANTS = ("mini",)`. X-Cell-Ultra (4.9B) is not present in the code and not on the model hub.
- MODEL_CARD reports a scaling-law exponent `L(N) ~ N^-0.32` with R^2 = 0.96, and per-screen cell / perturbation counts: HCT116 3.4M / 18,924, HEK293T 4.5M / 18,312, HepG2 2.6M / 9,735, iPSC 4.2M / 10,095, Jurkat Resting 2.8M / 10,872, Jurkat Active 2.8M / 10,878, iPSC Multi-Diff 5.1M / 12,175.
- Cross-attention style described in MODEL_CARD as "Flamingo-style"; not stated in the paper section read.
- Nothing in the released code contradicts paper hyperparameters; no code-vs-paper delta can be measured until the implementation lands.

## Evaluation
- Benchmarks: iPSC-200 and HepG2-200 held-out validation sets (200 held-out perturbations per screen); Replogle-Nadig dataset (four training cell lines + 380 HepG2 test perturbations); Parse-1M; Tahoe subset (chemical perturbations, zero-shot); melanocyte progenitors (held-out cell type, zero-shot); primary human CD4+ T cells (multi-donor, zero-shot); Jurkat CD3/CD28 activated (T-cell inactivation prediction).
- Metrics (cell-eval package): Pearson delta (log-fold-change correlation), MAE, DE Overlap@N, DE Direction Match, DE Spearman LFC, Centroid Accuracy, auROC on discovery.
- Baselines: STATE (a 2B "SE+ST" variant, retrained on a 5.5M-cell subset of X-Atlas/Pisces to match X-Cell's pre-training data), scGPT (fine-tuned from public human weights), Cell2Sentence (zero-shot readout only, due to compute constraints). Mean-perturbation and simple baselines discussed but explicit numbers not consistently tabulated in the section read.
- Reported result: on iPSC-200 and HepG2-200, X-Cell attains Pearson delta 0.51 and 0.48, versus STATE at 0.10 - a >5x improvement on the paper's headline metric. Tahoe zero-shot Pearson delta 0.31 for X-Cell vs 0.10 for STATE trained on the 3.1M public-only corpus.

## Claims (authors)
- The authors report state-of-the-art performance on genome-wide CRISPRi perturbation prediction, up to 5x higher Pearson delta than STATE on iPSC-200 and HepG2-200.
- The paper claims perturbation prediction follows LLM-like power-law scaling in model size, and that X-Cell-Ultra (4.9B) generalizes zero-shot to unseen iPSC-derived melanocyte progenitors and primary human CD4+ T cells, outperforming all baselines after self-supervised test-time adaptation.
- The paper attributes cross-context generalization to coordinated scaling of both interventional data (X-Atlas/Pisces) and model capacity, arguing that scaling parameters alone (as in prior single-cell foundation models) is insufficient.
- The authors report that STRING and ESM-2 are the dominant cross-attention priors driving predictions, based on attention weights and knowledge-source randomization.
- The paper claims zero-shot prediction of T-cell-inactivating perturbations in CD3/CD28-stimulated Jurkat cells, with expected genes (CD3D, CD3E, CD3G, CD247) recovered.

## Limitations
- Author-stated: predictions restricted to protein-coding genes to align with ESM-2 priors; STATE benchmark trained on a 5.5M-cell subset of X-Atlas/Pisces due to compute constraints; Cell2Sentence run zero-shot only; power-law fit is over a limited range of model sizes.
- Reviewer: reference implementation and weights are released only for the 55M X-Cell; the 4.9B X-Cell-Ultra weights are not stated as released, so the headline scaling and zero-shot claims are not independently reproducible. The paper's "5x" gain rests on a retrained STATE baseline at Pearson delta 0.10, an unusually low absolute value - it is not clear the STATE baseline was tuned to match X-Cell's training budget or set-size convention. Base X-Cell 55M pre-training optimizer/schedule are described only partially (LR/warmup/steps are given for X-Cell-Ultra phases); the exact 55M pre-training schedule is not stated. Ablations distinguishing contributions of individual prior sources rely on attention weights and randomization, not on retraining without a given prior, so the STRING/ESM-2 dominance claim is asserted more than causally established. GEARS, a standard baseline for perturbation prediction, is not among the reported comparators.

## Take / open questions
Does the 4.9B model's zero-shot advantage survive when other baselines are given the same X-Atlas/Pisces training budget rather than a 5.5M-cell subset? How much of the reported gain comes from the diffusion masking objective versus from cross-attention priors versus from set-level MMD training? Both questions would need the X-Cell-Ultra weights or a matched training run, neither of which is currently released.

## relates_to_prior
- target: STATE; axis: evaluation rigor / scale; shown (ran as a baseline on iPSC-200, HepG2-200, Replogle-Nadig, Parse-1M, Tahoe subset; reports >5x higher Pearson delta on the headline benchmark, though STATE was retrained on a subset)
- target: scGPT; axis: interventional vs observational pre-training; shown (fine-tuned as a baseline, and used both as encoder init for X-Cell and as one of the six cross-attention priors); the paper asserts scaling observational foundation models is insufficient
- target: Cell2Sentence (C2S); axis: perturbation prediction quality; shown (zero-shot readout only, not fine-tuned; presented as underperforming)
- target: single-cell foundation models generally (Geneformer/scGPT-family); axis: causality; asserted (paper argues observational atlases conflate correlation with causation; not directly benchmarked against Geneformer)
- target: GEARS and graph-based perturbation predictors; axis: generalization; asserted (referenced as prior work but not run as a baseline)
- target: mean-perturbation and simple baselines critiqued by refs [31, 32]; axis: evaluation floor; asserted (the paper cites the concern but does not tabulate mean-perturbation numbers alongside its main results)
