---
model: scDiffusion-X
paper_title: "A multi-modal diffusion model with dual-cross-attention for multi-omics data generation and translation"
authors: Luo, Wei, Hao, Zhang, Liu (Tsinghua / Yale)
year: 2026
venue: Nature Communications
task: generation
modality: multiome
organism: human
links:
  paper: https://doi.org/10.1038/s41467-026-71744-x
  code: https://github.com/EperLuo/scDiffusion-X
  weights: https://figshare.com/articles/dataset/scDiffusion-X/28582061 (pretrained on MiniAtlas; docs at scDiffusionX.readthedocs.io)
---

# scDiffusion-X

## TL;DR
A multi-modal latent diffusion model for paired single-cell multi-omics (scRNA-seq +
scATAC-seq). Its core novelty is a Dual-Cross-Attention (DCA) module that lets the two
modalities attend to each other during denoising instead of being concatenated, giving one
framework for conditional multi-omics generation, cross-modality translation (predict one
modality from the other with uncertainty intervals), in-silico perturbation, and gradient-
based gene regulatory network (GRN) inference.

## What it is
A latent denoising diffusion model with two components: (1) a multimodal autoencoder, two
per-modality MLP autoencoders (negative-binomial likelihood for RNA, Bernoulli for ATAC)
that compress each modality to a low-dimensional latent and decode back; (2) a multimodal
denoising network built from two single-modal denoisers coupled by DCA modules. Input is
paired RNA + ATAC (plus optional condition: cell type, tissue, disease, developmental stage,
embedded into the time-step representation); output is generated/translated RNA and ATAC
profiles. At inference, latents are sampled from Gaussian noise (or seeded from one observed
modality for translation) and decoded.

## Training data
Trained per dataset; one large dataset used for a released pretrained base model. Four paired
RNA+ATAC (multiome) datasets:
- **OpenProblem** (2021 NeurIPS competition): 69,249 cells, 22 cell types; after filtering
  13,431 genes and 36,553 peaks (genes in <10 cells and peaks open in <2% of cells removed).
- **PBMC10k** (10x Genomics): 10,000 cells, 14 cell types; 25,604 genes, 40,086 peaks.
- **BABEL dataset** (DM, HSR, PBMC, GM12878): 40,465 paired cells; 34,861 genes, 223,897
  peaks (used for the translation benchmark, split by Leiden cluster).
- **MiniAtlas** (scMMO-atlas, Cheng 2025): 133,549 cells, 56 cell types, 19 tissues; 36,601
  genes, 165,564 peaks after filtering. Used to train the released pretrained base model.
- ATAC binarized in all datasets; RNA/ATAC used directly on counts (no log transform), since
  the autoencoder models counts explicitly. Organism: human. Modality: paired multiome.

## Core mechanism
The distinguishing idea is the **Dual-Cross-Attention (DCA)** module for modality
integration inside the diffusion denoiser. Prior multi-omics integrators either concatenate
modalities (early/late/intermediate fusion, imposing a fixed interaction pattern) or share a
single VAE latent. DCA instead constructs separate Query/Key/Value projections for each
modality and computes cross-attention in both directions, so RNA borrows information from
ATAC and ATAC from RNA, with a residual connection preserving each modality's own
dimensionality. Three DCA modules sit at the input, middle, and output blocks of the
denoising network. This adaptive, interpretable exchange is reported to beat concatenation
and to enable two things concatenation cannot: (1) cross-modality translation, by seeding the
reverse process with the ground-truth latent of the known modality and Gaussian noise for the
other; and (2) gradient-based GRN inference, by back-propagating through the DCA attention
maps to score gene-peak regulatory relationships and build cell-type-specific heterogeneous
regulatory networks.

## Implementation (paper)
- Input representation: paired RNA + ATAC; each modality encoded to a low-dim latent by its
  own MLP autoencoder; ATAC binarized; RNA/ATAC modeled on raw counts (no log transform)
- Autoencoder: two MLP autoencoders (architecture adapted from CFGen / scVI). RNA decoder
  uses a negative-binomial likelihood (μ = s·ρ, s = size factor, learnable per-gene inverse
  dispersion θ); ATAC decoder uses a Bernoulli likelihood (π = sigmoid). Trained to maximize
  data log-likelihood
- Denoising network: two single-modal denoisers (Input / Middle / Output blocks of residual
  layers), coupled by three DCA modules. Res block = SiLU + LayerNorm + linear layers with
  additive time/condition embedding (Eq. 9). Res-block feature sizes 512/256/128 (input),
  128/256/512 (output), 128 (middle); DCA module feature dims 256/128/512
- DCA: per-modality trainable W matrices produce Q, K, V (R^{n×d}); bidirectional attention
  (RNA←ATAC and ATAC←RNA), softmax(QKᵀ/√d)·V, mapped back to original dimension with residual
  (Eqs. 11-13)
- Conditioning: condition (cell type, tissue, disease, dev stage) embedded and added to the
  time-step embedding (classifier-free style, no separate guidance classifier)
- Diffusion: T = 1000 steps, linear-in-t β schedule with βmin = 0.0001, βmax = 0.02;
  reverse variance weight (Eq. 8, exp(0.5·βt))
- Training objective: standard DDPM noise prediction on the multimodal latents (autoencoder
  trained first, then the denoiser)
- Optimizer and schedule: not stated in the main text (see Implementation (code))
- Preprocessing: gene/peak filtering per dataset (see Training data); no expression transform;
  for scalability experiments OpenProblem subsampled to top-2500 HVGs and peaks open in >30%
- Fine-tuning / adaptation: a base model pretrained on MiniAtlas can be used directly or
  fine-tuned on user data
- Inference: sample latents from Gaussian noise, denoise T steps (DPM-Solver++ available),
  decode per modality. Translation: replace the known modality's noisy latent with its
  ground-truth latent at each reverse step, generate the other from noise. Uncertainty: repeat
  translation with independent noise (N=100) to build per-gene prediction intervals
- Compute: all experiments on a single NVIDIA 80 GB A100 GPU; GPU-hours not stated (Table S1
  reports train/inference time by model size)
- Reproducibility: code on GitHub, docs on readthedocs; pretrained weights + data on figshare;
  all four datasets publicly available

## Implementation (code)
Cloned github.com/EperLuo/scDiffusion-X (~14.3k Python LOC; diffusion backbone forked from
guided-diffusion, autoencoder from a CFGen-style Hydra project). Concrete defaults recovered:
- Autoencoder configs (`script/training_autoencoder/configs/encoder/`): default multimodal
  RNA dims [512, 300, 100], ATAC dims [1024, 512, 100] (100-D latent per modality); large
  variant RNA [1024, 512, 150] / ATAC [2048, 1024, 200]; batchnorm; lr 1e-3, weight_decay
  1e-4; `is_binarized: True` for ATAC.
- Diffusion model defaults (`DiffusionBackbone/multimodal_script_util.py`): `num_channels` 128,
  `num_res_blocks` 2, `num_heads` 4, `cross_attention_resolutions` "2,4,8",
  `cross_attention_windows` "1,4,8", `cross_attention_shift` True, dropout 0.0,
  `use_scale_shift_norm` True, `diffusion_steps` 1000, `noise_schedule` "linear".
- Diffusion training (`script/training_diffusion/py_scripts/multimodal_train.py`): AdamW,
  `t_lr` 1e-4 (the active learning rate; `lr` defaults to 0.0), weight_decay 0,
  `lr_anneal_steps` 0, `schedule_sampler` "uniform", save_interval 10,000, batch_size default
  1 (placeholder; real per-run batch not in configs), microbatch disabled.
- Ships scripts for training the autoencoder and diffusion, sampling, sampling with attention
  maps, modality translation, and perturbation.

Net: architecture shapes, DCA wiring, and the autoencoder recipe are fully recoverable; the
paper's `not stated` diffusion optimizer is AdamW at lr 1e-4. Exact per-dataset batch size and
step counts are not pinned in the released configs.

## Evaluation
- Tasks and baselines:
  - Multi-omics generation (OpenProblem, PBMC10k): vs MultiVI, CFGen, scDesign3, and
    single-modal scVI (RNA) / peakVI (ATAC).
  - Scalability (filtered OpenProblem, 5k-69k cells): vs scDesign3, CFGen, MultiVI.
  - Conditional generation by cell type: vs CFGen, MultiVI (per-condition AUC).
  - Rare-cell augmentation: cDC2 / plasma cells added, random-forest cell-type classifier.
  - Modality translation (BABEL dataset + external GM12878): vs BABEL.
  - In-silico perturbation (knock out gene → predicted ATAC change direction): vs MultiVI, on
    a real Perturb-seq-style perturbation dataset (PerturBase) and a real multiome perturbation.
  - GRN inference: vs random-pair, upregulated+nearest-peak, RE+nearest-gene, MultiVI+correlation,
    and SCENIC+; validated against ENCODE H3K27ac and HiChIP loops.
- Metrics: PCC, SCC, MMD, LISI, AUC (RF, ~0.5 = realistic), marker-gene KLD, UMAP, pseudotime
  correlation (PTC). Translation adds per-gene prediction-interval coverage. GRN uses precision
  vs HiChIP loops. DCA analysis uses information entropy (IE) and MSE of attention maps.
- Reported results:
  - scRNA generation: improved MMD and LISI by 33.3% and 15.5% over the second-best method.
  - scATAC generation: AUC 0.846 → 0.575 (closer to the 0.5 ideal) vs best baseline.
  - Rare-cell augmentation: rare-type F1 from 0% to >80% while keeping high overall accuracy.
  - Translation: LISI 0.55 (RNA→ATAC) / 0.67 (ATAC→RNA) vs BABEL 0.05 / 0.31; MMD 0.26 / 0.18,
    83% / 54% lower than BABEL; lower marker-gene KLD; captures bimodal genes (e.g. ODC1) BABEL
    misses.
  - Perturbation: >80% correct change direction among top 20-50 differential peaks (ZAP70, CD3E,
    CD4), ~75% at top 100; beats MultiVI.
  - Uncertainty: average 93.94% coverage of nominal 95% prediction intervals; PI length
    correlates with real gene-expression SD (PCC 0.95, SCC 0.95).
  - GRN: peak-gene pairs reach 0.64 precision vs HiChIP loops, surpassing all baselines
    including SCENIC+.
  - DCA ablation: with-DCA beats without on MMD/LISI/AUC for both modalities; performance
    improves from 1 → 3 DCA modules; replacing DCA with a concatenating linear layer is worse;
    3 DCA modules chosen for the compute/quality trade-off.

## Claims (authors)
- The authors report state-of-the-art realistic multi-omics generation, preserving cellular
  heterogeneity and global structure, and scaling better than scDesign3 to large datasets.
- They claim DCA is a more flexible and interpretable integration than fixed early/late/
  intermediate fusion or concatenation, supported by ablations (with vs without DCA; DCA vs
  linear-layer concatenation; 1/3/5 modules).
- They claim scDiffusion-X uniquely enables accurate cross-modality translation with robust
  uncertainty quantification, outperforming BABEL, including well-calibrated prediction
  intervals.
- They claim in-silico perturbation on one modality yields correct regulatory change directions
  in the translated modality.
- They claim the gradient-based interpretability over DCA yields cell-type-specific
  heterogeneous GRNs that surpass SCENIC+ in precision against HiChIP loops.

## Limitations
- Author-stated: regulatory-relationship inference and heterogeneous-network construction are
  preliminary and intended as interpretability, not definitive biological evidence; the
  framework has no explicit tunable parameter controlling the strength of a learned regulatory
  link (future work: sparsity-inducing constraints on DCA weights); larger-scale benchmarks and
  statistical-significance analysis are needed; data augmentation may encourage dataset-specific
  structure rather than true biology, so the rare-cell experiment shows downstream discrimination
  gain, not confirmed novel biology; methods focus on shared cross-modality information and may
  underuse modality-specific signal.
- Reviewer: the diffusion optimizer/schedule and per-run batch/steps are not in the paper body
  (recoverable only from code). Only RNA+ATAC (multiome) is demonstrated despite the "any
  modality" framing; no protein/CITE-seq results. GRN precision (0.64) is validated against
  HiChIP loops and ENCODE marks as proxy ground truth, not perturbational causality. Translation
  and perturbation baselines are single (BABEL; MultiVI); no comparison to other recent
  cross-modality translators (e.g. scButterfly) in the main results. Realism leans on
  distribution-matching metrics (MMD/LISI/AUC≈0.5) that reward matching the training
  distribution. Compute is a single A100 but total GPU-hours are not stated.

## Take / open questions
scDiffusion-X extends the same group's single-modal scDiffusion to paired multi-omics, and the
real contribution is DCA as a learned, bidirectional, interpretable alternative to
concatenation, doubling as both a generator coupling and a GRN-inference substrate. The two
most consequential claims, well-calibrated translation uncertainty and DCA-derived GRNs beating
SCENIC+, are also the ones a follow-up should stress-test: the uncertainty is validated by
coverage on held-out data (good) but the GRN is validated only against proxy epigenomic ground
truth, not perturbation. Open question: does DCA's advantage hold for modalities with weaker
shared signal (protein, spatial) than the RNA-ATAC pair, where cross-attention has the most to
borrow?

## relates_to_prior
- target: MultiVI (Ashuach 2023); axis: integration (shared VAE latent vs DCA) / generation /
  perturbation; shown - ran as a baseline across generation, conditional generation, and
  perturbation; reports beating it.
- target: CFGen (Palma 2025); axis: generative approach (flow matching vs diffusion) /
  scalability; shown - ran as a baseline on generation and scalability; argues flow-based
  invertibility limits expressiveness and cross-modality relationship modeling.
- target: scDesign3 (Song 2024); axis: scalability of statistical simulators; shown - ran as a
  baseline; reports it fails to scale beyond ~10k cells while scDiffusion-X improves with size.
- target: scVI / peakVI (Lopez 2018 / Ashuach 2022); axis: single- vs multi-modal generation;
  shown - ran as single-modal generation baselines.
- target: BABEL (Wu 2021); axis: cross-modality translation; shown - ran as the translation
  baseline; reports lower MMD/LISI/KLD and better calibration.
- target: SCENIC+ (Bravo González-Blas 2023) and correlation/nearest-based GRN baselines; axis:
  GRN inference precision; shown - compared against them on peak-gene precision vs HiChIP loops,
  reports 0.64 surpassing all.
- target: concatenation / fixed-fusion multi-omics integration; axis: interpretability and
  flexibility; shown (ablation) - DCA beats a concatenating linear layer and no-DCA variants.
- target: scDiffusion (Luo 2024, same group); axis: single-modal → multi-modal; extended -
  builds on the latent-diffusion-for-scRNA idea and adds paired multi-omics via DCA.
