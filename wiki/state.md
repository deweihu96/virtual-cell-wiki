---
model: State
paper_title: "Predicting cellular responses to perturbation across diverse contexts with State"
authors: Adduri, Gautam, Bevilacqua et al. (Arc Institute)
year: 2025
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.1101/2025.06.26.661135
  code: https://github.com/ArcInstitute/State
  weights: https://huggingface.co/arcinstitute (SE-600M, ST-Tahoe, and others)
---

# State

## TL;DR
A two-module transformer that predicts perturbation responses over *sets* of cells rather
than single cells. A State Transition model (ST) uses set-level self-attention to map a set
of matched control cells to their perturbed counterparts under a distribution-matching (MMD)
loss; a separately pretrained State Embedding model (SE) supplies cell representations robust
to technical variation. Trained on >100M perturbed cells (ST) and 167M observational cells
(SE). Reported as the first model to consistently beat linear baselines on cross-context
perturbation generalization.

## What it is
Multi-scale encoder-decoder. Input to ST: a set of S control-cell profiles (raw HVG
expression or SE embeddings) that share covariates (cell line, batch), plus a perturbation
label. A transformer with bidirectional self-attention over the set predicts the perturbed
profile for each cell in the set. When cells are represented as SE embeddings, ST predicts
output embeddings that a separate MLP decoder maps back to expression. SE is a separate
600M-parameter self-supervised encoder-decoder that turns a cell into an embedding using
ESM-2 protein-language-model gene features.

## Training data
Two corpora, one per module:
- **SE (observational, 167M human cells)**: large public single-cell repositories (scBaseCount/
  Youngblut 2025, Tahoe-100M, scRecount/Program 2025), trained 4 epochs.
- **ST (interventional, >100M perturbed cells)**: large-scale single-cell screens across
  chemical, signaling, and genetic perturbations.
- Benchmark datasets used for the transition task: **Tahoe-100M** (50 cancer cell lines,
  1,138 conditions, 380 drugs), **Parse-PBMC** (90 cytokine perturbations, 12 donors, 18 cell
  types), **Replogle-Nadig** (2,024 genetic perturbations across 4 cell lines, after filtering
  low on-target efficacy).
- Organism: human. Modality: scRNA (Perturb-seq / drug and cytokine screens).

## Core mechanism
Two distinguishing ideas. First, ST operates on **sets of cells with self-attention** instead
of per-cell mapping: it constructs covariate-matched control sets and perturbed sets of fixed
size S and learns the transformation between the two *distributions*, trained with a Maximum
Mean Discrepancy (MMD) loss. This lets it model within-population biological heterogeneity
without explicit distributional assumptions; ablations show removing self-attention, or
collapsing to set size 1 or to pseudobulk mean-pooling, all degrade performance. Second, the
model separates the two noise sources in the authors' generative model of perturbation (true
effect + baseline heterogeneity + technical noise): SE, pretrained on observational data,
absorbs technical/heterogeneity variation into a robust cell embedding, and ST models the
perturbation transition on top. The paper also proves ST generalizes optimal-transport maps:
under regularity conditions the unique continuous OT map between control and perturbed
populations lies in ST's solution family.

## Implementation (paper)
- Input representation (ST): covariate-matched sets of S control cells; each cell = raw
  log-normalized top-2,000 HVG vector or a SE embedding; perturbation and (optional) batch as
  one-hot encodings; no positional encodings (cell order in a set is arbitrary)
- Input representation (SE): per cell, an "expression set" of the top L=2,048 genes by log
  fold expression; each gene featurized by ESM-2 (esm2_t48_15B_UR50D, 5,120-d) averaged over
  transcripts, projected to hidden dim; learnable [CLS] and dataset tokens appended
- Architecture (ST): shared hidden dim h across modules; transformer backbone is a HuggingFace
  LLaMA (dense datasets) or GPT-2 (sparse datasets like Replogle-Nadig), modified to
  bidirectional attention, word embeddings disabled, no dropout. Encoders fcell/fpert are
  4-layer MLPs (GELU + LayerNorm), fbatch an embedding layer, frecon a linear head, fdecode a
  3-layer MLP. Per-dataset configs: Tahoe set_size 256 / h 1,488 / 4 enc + 4 dec layers / 12
  heads / 244M params (LLaMA); Parse set_size 512 / h 1,440 / 12 heads / 244M (LLaMA);
  Replogle-Nadig set_size 32 / h 128 / 8 heads / 10M (GPT-2)
- Architecture (SE): 600M-parameter encoder-decoder; smaller specialized decoder for
  log-normalized gene expression
- Vocabulary: no fixed gene-token vocab; genes enter via ESM-2 embeddings (SE) or HVG index
  (ST); genes outside the embedding vocabulary ignored at inference
- Training objective (ST): MMD (energy-kernel) loss between predicted and observed perturbed
  cell sets; with SE embeddings, weighted combo L = MMD(emb) + 0.1*MMD(decoded expression)
- Training objective (SE): self-supervised gene-expression prediction plus a dataset-token
  classification term and a cross-cell gene-level correlation term
- Optimizer and schedule (SE): AdamW, max LR 1e-5, weight decay 0.01, zclip gradient clipping,
  3% linear warmup then cosine anneal to 30% of max, 4 epochs
- Optimizer and schedule (ST): trained end-to-end with the above losses; backbone initialized
  from N(0, 0.02^2), other weights Kaiming Uniform. Per-run LR/batch/steps not stated in the
  main text beyond Tables 3-4
- Preprocessing: top-2,000 HVGs (ST expression mode); top-2,048 expressed genes per cell (SE)
- Compute: SE trained on multiple nodes each with 8 NVIDIA H100 GPUs; PyTorch Lightning, DDP,
  automatic mixed precision. Exact node count / GPU-hours not stated
- Fine-tuning: transfer by reinitializing the dataset-specific perturbation encoder fpert while
  reusing shared weights
- Reproducibility: code (State, Cell-load, Cell-eval, State-reproduce) on GitHub; SE-600M and
  ST checkpoints on HuggingFace

## Implementation (code)
Cloned github.com/ArcInstitute/State (~20k Python LOC). Concrete ST defaults recovered:
- Training (`src/state/configs/training/default.yaml`): optimizer Adam
  (`src/state/tx/models/base.py:405`, `torch.optim.Adam(lr=self.lr)`), lr 1e-4, weight_decay
  5e-4, batch_size 16 (cell *sets*), max_steps 400,000, gradient_clip_val 10, val every 2,000
  steps, gradient_accumulation 1. Note: the main text describes SE as AdamW; the released ST
  code uses plain Adam.
- ST base model config (`src/state/configs/model/state.yaml`): hidden_dim 768, cell_set_len 512,
  LLaMA backbone with `bidirectional_attention: true`, 8 hidden layers, 12 attention heads,
  head_dim 64, intermediate_size 3072, no rotary embeddings, dropout 0.0; n_encoder_layers 1,
  n_decoder_layers 1; `predict_residual: true`, softplus output; loss `energy`.
- The "best" Tahoe config (`model/tahoe_best.yaml`): hidden_dim 1440, intermediate 4416, 4 hidden
  layers, 12 heads, head_dim 120, 4 encoder + 4 decoder layers. This matches the paper's Parse
  row (h=1440) but the paper's Tahoe row lists h=1488, a small paper-vs-code discrepancy.
- Loss (`src/state/tx/models/state_transition.py`): geomloss `SamplesLoss` with
  `loss="energy"`, blur 0.05, by default (the paper frames this as MMD; energy distance is the
  default kernel). A `CombinedLoss` option blends Sinkhorn (weight 0.01) + energy.
- Ships full preprint config TOMLs (`state_preprint_toml_files/`: tahoe/parse/replogle splits,
  including explicit zeroshot vs fewshot perturbation lists), Cell-eval, and reproduction code.

Net: unlike scFoundation, State's exact training recipe and per-dataset model configs are fully
recoverable from the repo. SE (600M embedding model) internals live in the same package but were
not fully traced on this pass.

## Evaluation
- Task: underrepresented-context generalization. Each model sees 30% of a test context's
  perturbations during training and predicts the rest in the held-out context (held-out cell
  lines / organs / donors / cell lines respectively for Tahoe / Parse / Replogle-Nadig).
- Baselines: a simple linear model (Ahlmann-Eltze 2024), CPA, scVI, the foundation model scGPT,
  plus two mean baselines ("context mean", "perturbation mean") that explain much of the
  variance in context-generalization tasks.
- Metrics: Cell-Eval framework spanning (1) expression counts (Pearson delta), (2) DE-gene
  identification (AUPRC, top-DEG overlap, LFC Spearman), and (3) perturbation effect size
  (Spearman on DEG count, confusion matrix).
- Embedding comparison: SE embeddings compared against scFoundation and TranscriptFormer cell
  embeddings as inputs to ST; SE reported to consistently win.
- Reported results: >30% improvement in effect discrimination on large datasets; consistently
  beats linear and mean baselines; ST+SE gives >17% Spearman improvement on effect-size ranking
  for Parse-PBMC / Replogle-Nadig and detects strong responses in unperturbed contexts.

## Claims (authors)
- State is, to the authors' knowledge, the first model to consistently outperform simple linear
  baselines on cross-context perturbation generalization.
- Set-level self-attention is essential: pseudobulk mean-pooling, set size 1, and no-attention
  ablations all degrade validation loss (Fig. 1D).
- The SE embedding, trained on observational data, enables zero-shot detection of strong
  perturbation responses in cell contexts never seen perturbed in training.
- SE embeddings outperform scFoundation and TranscriptFormer embeddings for downstream
  perturbation prediction.
- ST theoretically generalizes optimal transport: the continuous OT map lies in its solution
  family under mild conditions.

## Limitations
- Author-stated: accuracy of individual DE-gene predictions is sensitive to dataset size;
  performance still depends on how much the test context resembles training contexts (the
  underrepresented-context task deliberately leaks 30% of test-context perturbations).
- Reviewer: the headline evaluation gives every model 30% of the test context's perturbations,
  so it measures few-shot adaptation, not true zero-shot cross-context transfer; the "zero-shot"
  claim is separated to the SE-embedding experiments. Per-run ST training hyperparameters
  (LR, batch size, steps) and exact compute (node count, GPU-hours) are not in the main text.
  Baselines are strong for context generalization (linear, CPA, scVI, scGPT) but the genetic
  benchmark leans on just four cell lines (Replogle-Nadig), and no newer perturbation-specific
  model (e.g. TxPert, CellFlow, X-Cell) is run head-to-head. Not yet peer reviewed.

## Take / open questions
State's central bet, that modeling the control->perturbed transition over *sets* of cells with
MMD beats per-cell mapping, is the cleanest test of "distributions vs point predictions" in
the current crop, and the self-attention ablation is the evidence. Open question: how much of
the win is the set/MMD formulation versus the SE embedding, since the two are entangled in the
strongest results. The 30%-leak evaluation also makes cross-paper comparison to strict-OOD
benchmarks (scPerturBench, TxPert) non-trivial.

## relates_to_prior
- target: linear baselines (Ahlmann-Eltze 2024); axis: evaluation rigor / generalization; shown
  - ran as a baseline and reports consistently beating it, framed as the first model to do so.
- target: CPA / scVI; axis: generalization; shown - ran as baselines across all benchmark
  datasets.
- target: scGPT; axis: perturbation vs cell-type representation; shown - ran as a baseline and
  argues cell FMs optimized for cell-type discrimination miss subtle perturbation signal.
- target: scFoundation / TranscriptFormer; axis: embedding quality for perturbation; shown -
  compared SE against their cell embeddings as ST inputs, reports SE wins.
- target: optimal-transport methods (CellOT, etc.); axis: generalization; shown (theory) -
  proves ST contains the continuous OT map as a special case.
- target: GEARS-style control-to-perturbed mapping; axis: biological fidelity / heterogeneity;
  asserted - argues per-cell random matching fails when heterogeneity exceeds perturbation
  signal.
