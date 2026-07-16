---
model: scDiffusion
paper_title: "scDiffusion: conditional generation of high-quality single-cell data using diffusion model"
authors: Luo, Hao, Wei, Zhang (Tsinghua)
year: 2024
venue: Bioinformatics (Oxford University Press)
task: generation
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.1093/bioinformatics/btae518
  code: https://github.com/EperLuo/scDiffusion
  weights: not stated (code + datasets on Zenodo https://zenodo.org/doi/10.5281/zenodo.13268742)
---

# scDiffusion

## TL;DR
A latent diffusion model that generates realistic single-cell RNA expression profiles
under user-specified conditions (cell type, organ, developmental stage). It runs the
diffusion process inside the latent space of a pretrained foundation model (SCimilarity)
and steers generation with separately trained classifier-guidance heads, enabling
conditional, multi-condition (out-of-distribution combination), and continuous-trajectory
generation.

## What it is
A latent diffusion model (LDM) with three parts: (1) a pretrained foundation model
(SCimilarity) fine-tuned as the autoencoder that maps a cell's expression to a 128-D
latent and back; (2) a denoising network (a skip-connected MLP, not a U-Net/CNN) that
learns the reverse diffusion process on the latent; (3) one or more classifier-guidance
heads trained on noisy latents that inject condition gradients at inference. Input at use
time is a condition (or two conditions for interpolation) plus Gaussian noise; output is a
generated latent decoded to a full gene-expression profile.

## Training data
No single pretraining corpus; trained per dataset. Five scRNA-seq datasets used:
- **Human Lung Pulmonary Fibrosis (PF)** (Habermann 2020): >110k human lung cells.
- **Tabula Muris** (Schaum 2018): mouse, 12 organs.
- **Tabula Sapiens** (2022): human atlas; a subset of six cell types from five organs used.
- **Waddington-OT** (Schiebinger 2019): mouse embryonic fibroblast reprogramming to iPSC,
  18 days at half-day intervals, with a day-8 redifferentiation branch (trajectory task).
- **PBMC68k** (Zheng 2017): human PBMCs, 11 cell types (CD4+ T helper 2 removed).
- QC across all: drop cells with <10 counts and genes expressed in <3 cells.
- The autoencoder (SCimilarity) itself was pretrained on 22.7M cells across 399 studies
  (external, not this paper), then fine-tuned here (all layers except first and last).
- Organism: human and mouse. Modality: scRNA-seq.

## Core mechanism
The distinguishing idea is doing diffusion in a foundation-model latent space plus flexible
multi-classifier guidance, rather than training a bespoke conditional generator. Using
SCimilarity's encoder both reduces dimensionality (to 128-D) and transforms the expression
distribution into a Gaussian-like shape (Supplementary Fig. S1) that matches the diffusion
prior, so a plain skip-connected MLP denoiser suffices. Because classifiers are trained
separately from the denoiser and only applied at inference, several can be summed with
per-condition weights to control multiple attributes at once, which lets the model generate
condition combinations absent from training (e.g. "mammary B cell" from mammary-T / spleen-B
/ spleen-T training data). The **Gradient Interpolation** strategy replaces a single
condition gradient with a weighted blend of two condition gradients and initializes from a
noised real-cell latent, producing continuous cell-state trajectories (e.g. intermediate
developmental days) between two discrete states.

## Implementation (paper)
- Input representation: expression normalized to 1e4 total counts then log; encoded by
  fine-tuned SCimilarity encoder to a 128-D latent; decoder reconstructs full expression
- Autoencoder: SCimilarity (encoder-decoder MLP), fine-tuned from pretrained weights except
  the first and last layers (gene-count mismatch); 128-D latent as in the pretrained model
- Denoising network: skip-connected fully-connected network ("Cell_Unet"), residual blocks
  with SiLU + LayerNorm and additive time embedding; forward layers with history + reverse
  layers with skip connections. Handles the long/sparse/unordered expression vector where a
  CNN cannot
- Diffusion: T = 1000 steps; forward noise schedule parameterized by βmin, βmax (linear-in-t
  interpolation, Eq. 2); reverse variance uses adjustable weight w = 0.5
- Classifier: 4-layer MLP; trained on latents only between diffusion steps 0 and T/2 (later
  steps too noisy); cross-entropy on condition labels; takes latent + timestep as input
- Guidance: classifier gradient scaled by γ = 2 added at every reverse step (Eq. 5);
  multi-condition = weighted sum of multiple classifiers' gradients
- Gradient Interpolation: blend two classifier gradients with weights γ1, γ2; init latent =
  noised real cell at step t = 600 (Eq. 7), preserving initial-cell characteristics
- Training objective: standard DDPM noise-prediction (ε-prediction) on latents; classifier
  trained separately with cross-entropy
- Optimizer and schedule: not stated in the main text (see Implementation (code))
- Preprocessing: 1e4-count normalization + log; QC as above
- Fine-tuning / adaptation: SCimilarity fine-tuned per dataset; classifiers trained per
  condition type; denoiser trained per dataset
- Inference: start from Gaussian noise (or noised real latent for interpolation), denoise T
  steps under guidance, decode latent to expression
- Compute: not stated (hardware, GPU-hours)
- Reproducibility: full code and datasets released on GitHub and Zenodo; ablation and
  training/inference detail deferred to Supplementary Material

## Implementation (code)
Cloned github.com/EperLuo/scDiffusion (~4.9k Python LOC; a fork of OpenAI guided-diffusion).
Concrete defaults recovered:
- Denoiser training (`cell_train.py`): AdamW (via `train_util.py`), lr 1e-4, weight_decay
  1e-4, `lr_anneal_steps` 500,000, batch_size 128, microbatch disabled, save_interval 200,000.
- Diffusion defaults (`guided_diffusion/script_util.py`): `diffusion_steps` 1000,
  `noise_schedule` "linear", latent `input_dim` 128, denoiser `hidden_dim` [512, 512, 256, 128],
  dropout 0.0 (model) / 0.1 (classifier).
- Denoiser architecture (`guided_diffusion/cell_model.py`): `Cell_Unet` skip-connected MLP;
  note the class default `hidden_num=[2000,1000,500,500]` differs from the `script_util`
  wiring `[512,512,256,128]`; SiLU activations, LayerNorm, timestep embedding added per
  residual block.
- Classifier training (`classifier_train.py`): AdamW, lr 3e-4, weight_decay 0,
  `iterations` 500,000, batch_size 128, `noised=True`, `anneal_lr=False`.
- Per-dataset loaders shipped (`cell_datasets_muris/pbmc/sapiens/WOT/lung.py`); a `VAE/`
  module provides an alternative autoencoder to SCimilarity.

Net: unlike the paper's `not stated` optimizer/compute, the training recipe (optimizer, lr,
steps, batch size) is fully recoverable from the repo. GPU hardware/hours remain unrecorded.

## Evaluation
- Tasks: (1) unconditional generation realism; (2) single-condition cell-type generation;
  (3) multi-condition OOD generation; (4) continuous-trajectory interpolation.
- Baselines: scGAN and cscGAN (GAN generators), scDesign3, SPARSim, SCRIP (statistical
  simulators); linear interpolation for the trajectory task.
- Metrics: SCC, MMD, LISI, QQ-plot (statistical); UMAP, marker-gene expression, CellTypist
  classification accuracy, random-forest AUC and KNN AUC (AUC near 0.5 = indistinguishable
  from real).
- Reported results:
  - Realism (3 datasets): mean SCC 0.984, MMD 0.018, LISI 0.887, RF AUC 0.697; comparable to
    scGAN / scDesign3, above SPARSim / SCRIP. Pretrained-FM autoencoder gives higher quality
    with shorter training than training the autoencoder from scratch (Supplementary Table S1).
  - Single-condition (Tabula Muris, PBMC68k): CellTypist accuracy 0.93 (vs real 0.98,
    scDesign3 0.99, cscGAN 0.04); KNN AUC ~0.5 for scDiffusion vs >0.7 for cscGAN / scDesign3.
    Rare types (Thymus 2.5%, CD34+ 0.4%) generated well.
  - Multi-condition OOD: unseen "mammary B cell" combination, 98% of generated cells
    classified as B cells (real 92%); Tabula Sapiens held-out thymus memory-B and spleen
    macrophage recovered at 96.75% / 96.63% (real 96.91% / 99.53%).
  - Trajectory (Waddington-OT): beats linear interpolation on MMD and LISI; interpolated
    key-gene profiles (Shisa8, Prrx1, Fut9) peak at the correct day where linear interp misses.

## Claims (authors)
- The authors report scDiffusion generates scRNA-seq data comparable to state-of-the-art
  generators (scGAN, scDesign3) and better than SPARSim / SCRIP.
- They claim superior conditional generation: generated cells of a given type are
  indistinguishable from real cells to KNN/RF classifiers, unlike GAN- and scDesign3-generated
  cells, including for rare cell types.
- They claim multi-condition guidance can generate out-of-distribution cell types (unseen
  condition combinations) by composing known conditions.
- They claim the Gradient Interpolation strategy uniquely generates realistic intermediate
  cell states along a developmental trajectory, better than linear interpolation, and that
  this functionality is unique to scDiffusion.
- They report the pretrained foundation-model autoencoder improves both generation quality
  and training efficiency.

## Limitations
- Author-stated: no established way to assess the confidence of generated (especially unseen)
  data; existing metrics rely on known data and may not suit generated novel cells (they note
  MMD of unseen cells correlates with training MMD as a partial proxy). Better/universal
  evaluation metrics are needed; different metrics suit different use cases.
- Reviewer: training compute (hardware, GPU-hours) and the optimizer/schedule are not in the
  paper body (recoverable only from code). Much of the quantitative evaluation (SCC/MMD/LISI
  tables, ablations) is in Supplementary Material rather than the main text. Realism is judged
  largely by "indistinguishability" classifiers (RF/KNN AUC near 0.5) and CellTypist accuracy,
  which reward matching the training distribution and do not verify novel biological validity
  of OOD or interpolated cells; the OOD claim rests on classifier labels, not experimental
  confirmation. Baselines are generation/simulation methods; no comparison to the newer
  diffusion/flow generators used later in the same group's scDiffusion-X. The autoencoder is a
  frozen external model (SCimilarity), so representation quality is inherited, not evaluated.

## Take / open questions
scDiffusion is a clean demonstration that a foundation-model latent space plus DDPM plus
classifier guidance transfers the image-generation playbook to scRNA-seq. The most novel
piece is Gradient Interpolation for continuous trajectories, and it is also the least
validated: intermediate states are scored against held-out real days by MMD/LISI, which is
suggestive but not proof of biological correctness. The unresolved question the authors raise
themselves, how to gauge confidence in generated cells with no real counterpart, is the crux
for any downstream use (augmentation, in-silico perturbation). This model is the single-modal
predecessor to scDiffusion-X (same group), which extends the idea to paired multi-omics.

## relates_to_prior
- target: scGAN / cscGAN (Marouf 2020); axis: generation quality / conditional control; shown
  - ran as baselines; reports comparable unconditional quality but far better conditional
  realism (KNN AUC ~0.5 vs >0.7; CellTypist 0.93 vs 0.04).
- target: scDesign3 (Song 2024) / SPARSim / SCRIP; axis: realism and scalability; shown - ran
  as baselines; comparable to scDesign3 on realism, above SPARSim/SCRIP; argues statistical
  simulators struggle with conditions not in their original design.
- target: scVI / VAE generators (Lopez 2018); axis: purpose; asserted - argues VAE models
  target downstream tasks (batch correction, clustering) rather than high-fidelity generation.
- target: single-cell foundation models (SCimilarity, scGPT, scFoundation, Geneformer); axis:
  role as autoencoder / guidance; adopted - uses SCimilarity as the LDM autoencoder and notes
  any gradient-producing classifier (incl. scFoundation) could guide generation.
- target: linear interpolation; axis: trajectory fidelity; shown - beats it on MMD/LISI and on
  recovering the correct peak timing of key developmental genes.
