---
model: scGPT
paper_title: "scGPT: toward building a foundation model for single-cell multi-omics using generative AI"
authors: Cui, Wang et al.
year: 2024
venue: Nature Methods
task: embedding
modality: multi
organism: human
links:
  paper: https://doi.org/10.1038/s41592-024-02201-0
  code: https://github.com/bowang-lab/scGPT
  weights: https://github.com/bowang-lab/scGPT
---

# scGPT

## TL;DR
Transformer foundation model pretrained on 33M human single cells with a generative gene-expression-prediction objective, then fine-tuned per downstream task (cell type annotation, perturbation prediction, batch integration, multi-omic integration, GRN inference).

## What it is
Stacked self-attention transformer over gene-token sequences. Input per cell is three parallel embeddings summed elementwise: gene-name tokens, binned expression values, and condition tokens (batch, modality, perturbation flag). Output is either a per-gene expression prediction or a cell embedding via a special `<cls>` token. Pretraining is masked/autoregressive gene expression prediction with a custom attention mask; downstream use is task-specific fine-tuning heads on top of the same trunk.

## Training data
- Pretraining: 33M human cells from the CELLxGENE Census (release 15 May 2023), scRNA-seq + snRNA-seq, non-disease only, 51 organs, 441 studies.
- Organ- and cancer-specific variants: blood/bone-marrow model on 10.3M cells (65 datasets, filtered on Homo sapiens + blood/bone marrow + normal/COVID-19/influenza), pan-cancer model on 5.7M cells. Individual organ models (brain 13.2M, lung 2.1M, heart 1.8M, kidney 814k, pancreas 210k, intestine 94.5k) were also trained to study in-context effects.

## Core mechanism
Two pretraining prompts share a single attention-mask formulation:
- Gene-prompt: given expression of some genes in a cell, predict expression of the remaining ("unknown") genes.
- Cell-prompt: given only a `<cls>` cell embedding, iteratively generate whole-genome expression, K rounds, each round revealing the top 1/K most confidently predicted genes and using them as context for the next round.
The attention mask allows each unknown-gene query to attend only to known genes and to itself, never to other unknown genes. This adapts causal masking to non-sequential single-cell data. Fine-tuning composes objectives from a fixed menu: GEP (masked gene expression prediction), GEPC (predict expression from the cell embedding), ECS (elastic cell similarity), DAR (gradient-reversal domain adaptation), DSBN (domain-specific batch norm), and cell-type cross-entropy.

## Implementation (paper)
- Input representation: gene names as tokens; expression values discretized by per-cell equal-count binning into B intervals (B value not stated in main text); log1p transform and HVG selection applied at fine-tuning time before binning; `<cls>` prepended, `<pad>` for padding; only non-zero-expression genes fed at pretraining; condition tokens carry batch, modality, and perturbation flags
- Vocabulary: whole human genome as gene tokens; union of gene sets across studies enables cross-study harmonization; special tokens `<cls>`, `<pad>`
- Architecture: 12 stacked transformer blocks, embedding dim 512, 8 attention heads, FC hidden size 512; FlashAttention used for the self-attention kernel; max input length 1,200 (random subsample of non-zero genes when a cell has more)
- Training objective: masked gene-expression prediction (MSE) on the "unknown" positions; mask/generate ratio uniformly sampled from {0.25, 0.50, 0.75} per iteration; gene-prompt and cell-prompt losses summed within each step
- Optimizer and schedule: Adam, mini-batch 32, initial LR 1e-4, "weight decay of 0.9 after each epoch" (paper wording; this reads as a per-epoch LR decay factor rather than an AdamW weight-decay term). 6 pretraining epochs. Fine-tuning: LR 1e-4 decaying to 90% per epoch, 15 epochs for integration/annotation/perturbation, 25 epochs for multi-omic integration; multi-omic model uses 4 transformer blocks and LR 1e-3 with 0.95 decay
- Preprocessing: for fine-tuning tasks, per-cell library-size normalization, log1p, HVG selection (1,200 HVGs for integration; 3,000 for annotation; 5,000 for perturbation), then binning; peaks for ATAC and proteins for CITE-seq handled through separate token vocabularies
- Fine-tuning / adaptation: pretrained weights initialize trunk; task heads (MLP classifier for cell types; perturb-GEP head for perturbation; batch- and modality-token embeddings for integration) are freshly initialized. Perturbation head uses log1p values (not bins), pairs a random control cell with each perturbed cell as input-target, and appends a binary perturbation flag per gene
- Inference: cell embedding = final-layer embedding at the `<cls>` position; gene-level predictions from per-gene positions
- Compute: not stated (GPU count, GPU-hours, total wall clock all absent from the main text)
- Reproducibility: code and pretrained weights released at github.com/bowang-lab/scGPT under MIT; Zenodo snapshot at 10.5281/zenodo.10466117; processed datasets at figshare 24954519

## Implementation (code)
Read from bowang-lab/scGPT, `scgpt/preprocess.py`, `scgpt/data_collator.py`, `scgpt/model/generation_model.py`, `scgpt/trainer.py`, and `tutorials/Tutorial_Perturbation.ipynb`.
- Expression bins: `n_bins = 51` (`data_collator.py:90`, `preprocess.py`); the value not stated in the paper's main text
- Fine-tune trunk (perturbation tutorial defaults, matching the pretrained checkpoint): `embsize = 512`, `d_hid = 512`, `nlayers = 12`, `nhead = 8`
- Fine-tune training: `lr = 1e-4`, `batch_size = 64`, `epochs = 15`, optimizer `torch.optim.Adam` with no explicit `weight_decay`, LR schedule `StepLR(gamma=0.9, schedule_interval=1)`. The "weight decay of 0.9 after each epoch" in the paper is this scheduler gamma, not an AdamW weight-decay term.
- Trainer takes `mask_ratio` from a config object (no baked-in default in `trainer.py`); tutorials set it per-task (e.g. 0.4 for integration as stated in the paper)
- Perturbation head uses a `pert_encoder = nn.Embedding(3, d_model)` (three states: unperturbed, perturbed, pad), confirming the binary-flag-per-gene input format described in the paper
- Attention: FlashAttention with a fallback path in `scgpt/model/flash_attn_compat.py`
- Pretrained vocabularies shipped with the repo: `default_gene_vocab.json` and `default_census_vocab.json` under `scgpt/tokenizer/`

## Evaluation
Multiple tasks, each with its own held-out split and baselines.
- Cell-type annotation on MS, myeloid, human-pancreas datasets. Baselines: scBERT, TOSICA. Metrics: accuracy, precision, recall, macro F1. scGPT reported to lead across all datasets and metrics.
- Perturbation prediction on Adamson (87 one-gene perturbations), Norman (131 two-gene + 105 one-gene), Replogle (1,823 one-gene), all K562. Baselines: GEARS, linear regression. CPA was excluded by the authors on the grounds that its intended setting (unseen cell types) differs from theirs. Metric: Pearson_delta and Pearson_delta on top-20 DE genes. Reports 5-20% margin over GEARS and linear.
- Reverse perturbation on a 20-gene Norman subset (210 combos, 39 training, 7 test). Top-K retrieval; scGPT reports 91.4% relevant recall at top-1 and 65.7% exact recall at top-8 across 7 test cases.
- scRNA-seq batch integration on PBMC 10k, perirhinal cortex, COVID-19. Baselines: Seurat, Harmony, scVI. Metrics: NMIcell, ARIcell, ASWcell (aggregated as AvgBIO), plus ASWbatch and GraphConn (AvgBATCH).
- scMultiomic integration on 10x Multiome PBMC (RNA+ATAC), BMMC (RNA+protein), ASAP PBMC (mosaic RNA/ATAC/protein). Baselines: scGLUE, Seurat v4, scMoMat.
- GRN inference validated qualitatively against HLA and CD gene groupings, Reactome pathway enrichment against a Pearson coexpression baseline, and attention-based most-influenced-gene selection validated against ChIP-Atlas for DDIT3 and BHLHE40 perturbations.
- Scaling / in-context ablation: pretraining data sizes from 30k to 33M cells with fixed parameter count show monotone downstream improvement; organ-matched pretraining beats organ-mismatched pretraining at similar data size (blood-pretrained beats brain-pretrained on COVID-19 integration by 8% despite similar corpus sizes).

## Claims (authors)
- The authors report state-of-the-art on cell-type annotation, perturbation prediction, batch correction, and multi-omic integration among the compared baselines.
- The paper claims a scaling effect: fine-tuned downstream performance monotonically improves with pretraining corpus size, mirroring the LLM scaling law.
- The paper claims context-aligned pretraining (organ-matched) improves downstream results at fixed data size.
- The paper claims the generative attention-masking scheme unifies encoder-style masked prediction and decoder-style autoregressive generation for non-sequential single-cell data.
- The paper claims attention maps recover known transcription-factor target gene sets (DDIT3, BHLHE40 targets validated against ChIP-Atlas) and known gene groupings (HLA class I vs II, CD3 T-cell complex, CD79 B-cell signaling).

## Limitations
- Author-stated: pretraining does not inherently mitigate batch effects, so zero-shot performance is limited on datasets with strong technical variation; evaluation is complicated by the frequent absence of biological ground truth; future work should add multi-omic and diseased pretraining data.
- Reviewer: several key hyperparameters not stated in the main text, most notably the number of expression bins B and the compute (GPU count, GPU-hours) for the 33M-cell pretraining run, so the pretraining run is not reproducible from the paper alone. The Adam "weight decay of 0.9" reads as ambiguous between a per-epoch LR decay factor and an atypical AdamW weight-decay term. The perturbation-prediction comparison excludes CPA on interpretive grounds rather than running it, weakening the "SOTA" framing. Cell-type annotation baselines are limited to scBERT and TOSICA; Geneformer, mentioned as a peer in the intro, is not benchmarked. GRN-inference validation is largely qualitative (visual similarity networks, small ChIP-Atlas overlaps) rather than quantitative against a benchmark like BEELINE.

## Take / open questions
Does the reported scaling law hold beyond 33M cells, and does organ-matched pretraining still win when the target dataset is genuinely out-of-distribution (rare tissues, disease states not in pretraining)? How much of the perturbation-prediction lead over GEARS is the pretraining vs the log1p-target and control-cell pairing choices specific to scGPT's fine-tune head? Both are answerable with the released code.

## relates_to_prior
- target: GEARS (Roohani 2023); axis: generalization to unseen perturbed genes and to combinatorial perturbations; shown (ran as a baseline on Adamson, Norman, Replogle; reports Pearson_delta advantage)
- target: Geneformer (Theodoris 2023); axis: single-cell foundation model design (rank-based tokenization vs binned values + gene-prompt/cell-prompt generative pretraining); asserted (cited in intro as motivation, not benchmarked head-to-head)
- target: scBERT (Yang 2022); axis: cell-type annotation; shown (ran as a baseline on three annotation datasets)
- target: TOSICA (Chen 2023); axis: cell-type annotation; shown (ran as a baseline)
- target: CPA (Lotfollahi 2023); axis: perturbation-response prediction; asserted (excluded from perturbation benchmark on the grounds of different intended setting)
- target: scVI, Seurat, Harmony; axis: scRNA-seq batch integration; shown (ran as baselines with AvgBIO/AvgBATCH metrics)
- target: scGLUE (Cao 2022), Seurat v4, scMoMat; axis: paired and mosaic multi-omic integration; shown (ran as baselines)
