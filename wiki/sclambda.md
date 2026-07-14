---
model: scLAMBDA
paper_title: "Modeling and predicting single-cell multi-gene perturbation responses with scLAMBDA"
authors: Wang, Liu et al.
year: 2024
venue: bioRxiv preprint (not peer reviewed)
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.1101/2024.12.04.626878
  code: https://github.com/gefeiwang/scLAMBDA
  weights: not stated
---

# scLAMBDA

## TL;DR
A VAE that predicts single-cell post-perturbation expression by disentangling a basal cell state (learned from expression) from a perturbation-specific salient representation (learned from a pretrained gene-name embedding, GenePT by default), with MINE-based mutual-information regularization and FGSM adversarial training on the perturbation embedding.

## What it is
Deep generative model (variational autoencoder). Two encoders: `E_cell` maps gene expression to a Gaussian basal latent `z ∈ R^d`; `E_perturb` maps a pretrained language-model gene embedding `p ∈ R^p` to a salient latent `s ∈ R^d`. A single decoder `D_cell(z + s)` reconstructs gene expression. A second decoder `D_perturb(s)` reconstructs `p` to prevent perturbation info from leaking into `z`. Multi-gene perturbations combine embeddings by simple sum: `p_ab = p_a + p_b`. Inference: given a control cell and a perturbation label, sample `z` from the control's posterior, add `s = E_perturb(p_new)`, decode.

## Training data
Not a foundation model. Trained per dataset. Datasets used in the paper:
- Adamson 2016 (K562, CRISPRi), 86 target genes, single-gene only.
- Replogle 2022 (RPE1, CRISPRi), 2,393 essential target genes, single-gene, genome-scale.
- Norman 2019 (K562, CRISPRa), 104 single-gene + 130 two-gene perturbations.
All human cell lines. Input restricted to 5,000 highly variable genes per dataset.

## Core mechanism
The distinguishing claim is disentangled representation + LLM-derived perturbation embeddings. Basal `z` and salient `s` are enforced to be independent by minimizing their mutual information via the MINE estimator with an auxiliary network `T_ψ` (using stop-gradient on `s` so only `z` moves). To generalize to unseen perturbations in the sparse LLM-embedding space, the paper adds FGSM adversarial perturbations to `p`: `p̃ = p + ε · sign(∇_p ‖x̂ − x‖²)` with `ε = 0.001·‖p‖`. Compared to GEARS (GO-graph GNN) and scGPT (transformer over expression), scLAMBDA never sees or reasons about the perturbed gene's own expression — it only uses the language-model embedding of the gene name — which is what lets it score truly novel target genes that lack expression measurements.

## Implementation (paper)
- Input representation: 5,000 HVG expression vector per cell; LLM-derived embedding vector for the perturbation label (default GenePT); for two-gene perturbations, `p_ab = p_a + p_b`
- Vocabulary: no gene token vocab; gene identities enter only through the frozen external embedding
- Architecture: VAE. `E_cell` encoder outputs (μ, log σ) of a Gaussian posterior over `z`; `E_perturb` encoder maps `p` to `s`; `D_cell` decoder reconstructs `x` from `z + s`; `D_perturb` decoder reconstructs `p` from `s`; MINE auxiliary network `T_ψ` for mutual-information estimation. Layer counts, hidden widths, and parameter counts not stated in the main text (deferred to supplementary Tables 1-3, not reproduced here)
- Training objective: ELBO (Gaussian reconstruction + KL to N(0, I_d)) minus `λ_MI · Î(z; s)` (MINE-estimated MI) minus perturbation-embedding reconstruction loss `‖p̂ − p‖²`; adversarial `p̃` is substituted for `p` in each iteration before the ELBO forward pass
- Optimizer and schedule: Adam, β = (0.9, 0.999), lr = 5e-4, StepLR gamma = 0.2 every 30 epochs, 200 epochs, batch size 500
- Preprocessing: top-5,000 HVGs per dataset; standard scanpy pipeline; log-normalized expression as input
- Fine-tuning / adaptation: none, single-stage training per dataset
- Inference: for in silico perturbation of a control cell, encode to `z`, add `s = E_perturb(p_new)`, decode; for population generation, sample many controls and repeat
- Hyperparameters: latent dim `d = 30`, MI weight `λ_MI = 200`, adversarial ε = 0.001·‖p‖
- Compute: not stated (no GPU count, no GPU-hours; supplementary Fig. 10 reports relative training time vs baselines but not absolute numbers in the main text)
- Reproducibility: code released at github.com/gefeiwang/scLAMBDA; all three datasets public via GEO (GSE90546, GSE146194, GSE133344)

## Implementation (code)
Read from gefeiwang/scLAMBDA (1.2k LOC total), `sclambda/networks.py` and `sclambda/model.py`.
- Encoder and decoder MLPs: 2 hidden Linear layers of width 512 with LeakyReLU(0.2); output is a single linear head (no activation). The VAE encoder has two output heads for mean and log-variance
- Hidden dim: 512, latent dim: 30 (matches paper)
- MINE network: 2 hidden Linear(512) + Linear(1), output clamped to [-50, 50] for numerical stability; not mentioned in the paper
- Training: `training_epochs = 200` (matches paper's stated default); `batch_size = 500`; main optimizer Adam lr=5e-4; StepLR step_size=30, gamma=0.2 (matches paper)
- MINE optimizer is separate: Adam lr=5e-4, weight_decay=1e-4, its own StepLR step=30 gamma=0.2. The paper doesn't mention this weight-decay term
- `lambda_MI = 200` (matches paper)
- Gradient clipping enabled by default via `grad_clip=True`
- Alternative `Net_context` variant adds a third encoder/decoder pair for a "context" covariate (cell type / batch), summing `z + s + c` before decoding. Not described in the main text — suggests batch-aware use is supported in code but not in the paper's benchmarks
- Alternative `Decoder_tgid` head predicts scalar expression of the perturbed target gene itself as an auxiliary output, gated by `use_tg_coord`. Also not in the main paper text
- Device selection: CUDA > MPS > CPU

## Evaluation
Task: predict post-perturbation single-cell expression for held-out target genes.
- Adamson (single-gene, K562, 10 random splits): PCC_delta on HVGs = 0.786 (scLAMBDA) vs 0.775 (GenePert), 0.692 (GEARS), 0.661 (scGPT); W2 on HVGs = 22.552 (scLAMBDA, lowest); AUC for DE-gene identification = 0.969 upregulated / 0.996 downregulated.
- Replogle RPE1 (single-gene, genome-scale, 10 splits): PCC_delta = 0.564 (scLAMBDA) vs 0.547 (GenePert); W2 = 28.737 (scLAMBDA) vs 29.487 (scGPT).
- Norman (single-gene + two-gene, 10 splits): overall PCC_delta = 0.664, W2 = 13.228, both best. Two-gene broken out by GEARS-style 0/2, 1/2, 2/2 seen categories; PCC_delta rises 0.593 → 0.718 → 0.850 as more targets are seen in training.
- Genetic interaction scores (magnitude, model fit, singles-to-double similarity, following Norman 2019 definitions): scLAMBDA has lowest RMSE on all three. Paper works out analytically that GenePert's magnitudes are biased toward ~0.819 due to its linearity plus GenePT's 2-norm structure, and shows this empirically.
- Embedding ablation on Norman: GenePT > scGPT > HyenaDNA / DNABERT-2. Genome-sequence embeddings underperform LLM embeddings.
- Ablation: removing MI regularization or adversarial training both degrade accuracy (Supplementary Fig. 6).
- Compute: supplementary Fig. 10 reports scLAMBDA is faster to train than all baselines except the linear GenePert.

## Claims (authors)
- The authors report state-of-the-art perturbation prediction across three Perturb-seq datasets (Adamson, Replogle, Norman) versus GEARS, scGPT, and GenePert, on both PCC_delta (mean effect) and W2 (single-cell distribution).
- The paper claims scLAMBDA is the only method among those compared that captures the heterogeneity of single-cell responses, not just the mean, and demonstrates this with W2 metrics and per-gene distribution comparisons (STT3A, MRPL55, MT-ATP6).
- The paper claims robust generalization to unseen target genes, driven by leveraging LLM embeddings of gene names (which do not require the gene to have measured expression, unlike scGPT, nor to be present in a GO knowledge graph, unlike GEARS).
- The paper claims scLAMBDA's basal representation captures cell cycle structure (Fig. 4e) as a validation of disentanglement.
- The paper claims computational efficiency: faster training than GEARS and scGPT, similar or better memory profile.

## Limitations
- Author-stated: performance depends on the quality of the gene embedding used (Supplementary Fig. 7). The framework has not been extended to other perturbation types (e.g. chemical) though the authors flag this as future work.
- Reviewer: this is a bioRxiv preprint, not peer reviewed. Architecture-level implementation details (encoder/decoder depth and width, MINE network shape, parameter counts) are pushed to supplementary tables not reproduced in the main text, so reproducibility from the paper alone is incomplete without reading the code. The scGPT comparison uses scGPT's own perturbation-prediction pipeline, which recent independent work (cited as refs 19, 20) argues underperforms simple linear baselines; this weakens the "beats scGPT" claim as a comparison of the underlying methods versus a comparison against scGPT's task-specific fine-tune. The two-gene evaluation category counts don't fully match the standard GEARS naming ("0/1 seen" for single-gene test, then 0/2, 1/2, 2/2 for two-gene). No compute cost stated in absolute terms.

## Take / open questions
Does the GenePT-embedding advantage still hold as newer gene-embedding models come out, and does scLAMBDA transfer to non-CRISPR perturbations (drugs, cytokines) where the "gene name" abstraction breaks? The MI regularizer weight (λ_MI = 200) is large; how sensitive is generalization to this and to the adversarial ε?

## relates_to_prior
- target: GEARS (Roohani 2024); axis: single-cell heterogeneity and generalization to unseen genes; shown (ran as a baseline on all three datasets; reports PCC_delta and W2 advantage)
- target: scGPT (Cui 2024); axis: perturbation prediction quality and single-cell distribution capture; shown (ran as a baseline; also compared scGPT gene embeddings vs GenePT as embedding backbone for scLAMBDA itself)
- target: GenePert (Chen 2024, GenePT-based ridge regression); axis: nonlinearity and single-cell heterogeneity; shown (ran as a baseline; paper derives analytically that GenePert's linearity biases two-gene magnitudes toward ~0.819)
- target: scGen (Lotfollahi 2019), CellOT (Bunne 2023), CPA (Lotfollahi 2023); axis: generalization to novel target genes not present in training; asserted (cited as unable to handle unseen genes, not benchmarked)
- target: contrastiveVI, sparse VAE, SAMS-VAE, GSFA; axis: same generative-modeling class but not prediction-focused; asserted (cited as related generative approaches, not benchmarked)
- target: HyenaDNA, DNABERT-2 (as embedding sources for scLAMBDA); axis: embedding informativeness; shown (swapped in as `p` source; reported to underperform GenePT and scGPT embeddings)
