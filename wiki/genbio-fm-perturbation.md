---
model: GenBio-FM-Perturbation
paper_title: "Foundation Models Improve Perturbation Response Prediction"
authors: Cole et al. (GenBio AI)
year: 2026
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://www.biorxiv.org/content/10.64898/2026.02.18.706454v1
  code: https://github.com/genbio-ai/foundation-models-perturbation
  weights: not stated
---

# GenBio-FM-Perturbation

## TL;DR
A benchmark study of >600 foundation-model embedding variants for genetic and chemical
perturbation response prediction, plus an attention-based fusion model that integrates
multiple embeddings and, on some cell lines, matches the estimated experimental error
bound.

## What it is
This paper is primarily a benchmark. Perturbations are represented as fixed-length
embeddings drawn from foundation models over several modalities (scRNA, DNA, protein
sequence/structure, small molecule, interactome, gene function text). A simple
translator (default kNN regression, sometimes Lasso or MLP) maps embeddings to
per-gene log-fold-change (LFC) vectors or DEG classifications. The paper also introduces
an attention-based fusion head (Simple and Full variants) that consumes the set of
per-perturbation embeddings and predicts LFC.

## Training data
Benchmark datasets, not a single pretraining corpus:
- **Essential** (main benchmark, genetic CRISPR knockdown): 964,451 cells across 4
  cell lines (K562, Hep-G2, Jurkat, hTERT-RPE1), ~2000 essential-gene perturbations
  per line, 56 small batches.
- **Norman** (genetic, CRISPRa, K562): 105 single-gene perturbations, 8 large batches.
- **Tahoe-100M** (chemical): ~100M cells, 50 cell lines, 379 drugs at 3 doses (highest
  used, 5 uM), ~60k genes, 14 large batches.
- **Sciplex3** (chemical): 799,317 cells, 3 cell lines, 188 drugs at 4 doses (highest
  used, 10 uM), restricted to 18,590 protein-coding genes, 52 medium batches.

All datasets normalized to total counts 1e4 then log1p.

## Core mechanism
Two ideas separate this work from prior FM-vs-baselines benchmarks:
1. **Embedding-centric decomposition.** Perturbation prediction is factored into (a) an
   embedding of the perturbation from any FM, and (b) a simple translator (kNN by
   default) from embedding to expression change. This makes ~600 FM variants
   comparable on the same downstream task and shows that modality of pretraining data,
   not model architecture, is the dominant factor.
2. **Attention-based multimodal fusion.** A transformer head consumes the set of
   per-source embeddings for a perturbation (with per-source projection to a shared
   dim, dropout, a cell-line token, and a CLS token) and predicts LFC from the CLS
   output. Trained jointly across cell lines.

## Implementation (paper)
This paper does not train a single FM. Details below are for the fusion model and the
main translator/finetuning experiments; the ~600 evaluated FMs are cited from prior
work.

**Simple Fusion Model (main proposed method)**
- Input representation: set of per-source embeddings `{z_k^(e)}` for perturbation k;
  each passed through a per-source linear projection `f_theta(e): R^d_e -> R^d`, then
  LayerNorm + Dropout(p=0.1). A learnable cell-line vector is added elementwise; a
  learnable CLS token is appended.
- Architecture: transformer with `d_model = 100`, 5 attention heads, depth tuned via
  Optuna over 1–8 layers.
- Prediction head: linear map from CLS output to `R^G` (LFC per gene) or per-gene
  class logits (DEG).
- Training objective: L2 loss (LFC regression) or classification loss (DEG).
- Optimizer and schedule: Adam, batch size 64; learning rate tuned in [1e-5, 1e-3]
  (log). ~100 Optuna trials (50 for Tahoe DEG). Tuned on one fold, reused across folds.
- Preprocessing: standard dataset normalization (counts to 1e4, log1p).
- Compute: total study ~5500 GPU-hours (~7.6 GPU-months) on a mix of H100 and A100.
  Latent Diffusion and Schrödinger Bridge alone consumed ~1000 GPU-hours each.
- Reproducibility: code and data released on GitHub (genbio-ai repo).

**Full Fusion Model.** Applied only to Essential LFC; more complex and harder to
tune; architectural details deferred to Supplemental Methods (not stated in main text).

**Fine-tuning experiments**
- AIDO.Cell (3M), Indexing mode: two-hidden-layer MLP head (1024, 2048), LoRA, 20
  epochs, batch size 8 with 8 perturbations per batch, LR tuned over
  {1e-2, 1e-3, 1e-4, 1e-5}, Adam or Muon (no significant difference).
- AIDO.Cell (3M), In-Silico KO mode: MLP head (64, 32), LoRA, 250 epochs, same
  batching and LR tuning.
- STRING GNN fine-tune: initialized from link-prediction-pretrained GNN; linear head;
  LR in {1e-3, 1e-4, 1e-5}, batch size in {16, 32, 64}; best config per fold selected
  on validation loss.

**Translators (baseline evaluation setup).** kNN regression is default; Lasso and a
non-linear MLP considered as ablations. Full hyperparameters deferred to Supplemental
Methods.

**LFC ground truth.** Batch-aware ATE (BA-ATE). For large-batch datasets: per-batch
(perturbed mean − control mean), averaged across batches containing both. For small-
batch Essential: per-batch perturbed mean − global control mean, averaged. Called
"log fold-change" as an abuse of terminology.

**Evaluation split.** 5-fold cross-validation over unseen perturbations within seen
cell lines. STATE comparison uses a separate cross-context protocol
(Supplemental Methods, not stated in main text).

## Evaluation
- **Genetic perturbations (Essential LFC).** kNN + FM embedding, compared against
  Train Mean, No Change, PCA, Random Embeddings, an Idealized Baseline (PCA on the
  full ground-truth matrix including test perturbations), and an Experimental Error
  lower bound estimated via hierarchical bootstrap at q=0.9. Baselines cover
  expression FMs (scGPT, scPRINT, Geneformer, TranscriptFormer, AIDO.Cell 3M/10M/100M,
  Positional), DNA (AIDO.DNA), protein (ESM2, AIDO.ProteinIF-16B, AIDO.ProteinRAG-16B
  with/without structure, AIDO.StructureTokenizer, STRING Sequence), prior-knowledge
  (DepMap, GenePT variants, GenotypeVAE, STRING Network/Spectral/GNN/WaveGC).
  Interactome-derived embeddings (STRING WaveGC/GNN/Network) top the ranking on all
  four cell lines. On K562, the best embedding closes 77% of the gap to the
  experimental error bound.
- **Genetic perturbations (Norman).** All embeddings look similar (Figure S1), matching
  the Ahlmann-Eltze / Wu et al. "no clear benefit" conclusion; the paper argues Norman
  is too small to resolve differences.
- **Chemical perturbations (Tahoe, Sciplex).** In LFC regression, no embedding
  convincingly beats negative controls. In the DEG formulation, target-based
  embeddings (simple target encoding, AIDO.Cell 100M, scPRINT via targets) and
  ECPF:2 fingerprints outperform negative controls; SMILES FMs mostly underperform.
- **Advanced translators.** Latent Diffusion, Flow Matching, and Schrödinger Bridge
  (using the same conditioning embedding as kNN) do not beat kNN with the best
  embedding, at ~1000 GPU-hours each.
- **Comparison to published models.** GEARS run as a baseline. STATE compared on a
  cross-context task (Essential, Tahoe); an embedding-based MLP matches or beats
  STATE in 3/6 dataset-metric pairs, but a one-hot-perturbation ablation performs
  nearly the same, so the embeddings are not the driver in that setting.
- **Fusion.** Attention fusion always beats the best single embedding on Essential
  LFC; on K562 and Jurkat it matches the experimental error bound.

Metrics: primarily L2 error on per-gene LFC (Eq. 7), macro-F1 for DEG. MAE, MSE,
Spearman, Pearson considered in Figure S11.

## Claims (authors)
- The authors report that some foundation models significantly outperform simple
  baselines for perturbation prediction; interactome-based FMs perform best for
  genetic perturbations.
- The paper claims embedding utility is primarily driven by pretraining modality, not
  model architecture, supported by a CCA-based clustermap grouping embeddings by
  source.
- The authors claim attention-based multimodal fusion reaches the estimated
  experimental error bound on 2 of 4 Essential cell lines and always beats the best
  single embedding on Essential LFC.
- The paper claims that expensive distribution-matching methods (Latent Diffusion,
  Flow Matching, Schrödinger Bridge) provide no practical benefit over kNN when
  starting from the same embedding, under the LFC formulation studied.
- The authors claim off-the-shelf small-molecule FMs are weak because they target
  chemical, not biological, function; a biologically grounded small-molecule FM is
  missing from the field.
- The paper claims larger expression FMs perform better (AIDO.Cell 3M < 10M < 100M),
  citing this as evidence that model scaling helps for gene embeddings.

## Limitations
- Author-stated: fine-tuning is highly susceptible to overfitting given current
  perturbation dataset sizes, so frozen embeddings may be more robust than fine-tuning.
  Fusion nearly saturates Essential LFC, so harder train/test splits will be needed
  going forward. Fusion did not help in the chemical DEG setting.
- Reviewer: The "modality drives performance" claim is entangled with architecture
  choice (transformers for expression, GNNs for interactome), and the paper concedes
  this ("some combination of information in the modality... and the model classes
  that tend to be used"); no controlled ablation swapping architecture within a
  modality is reported. The Idealized Baseline uses PCA over ground-truth LFCs that
  include the test perturbations, so it is not achievable in practice; the reader
  should be careful reading "% of the gap closed" numbers relative to this bound.
  STATE comparison uses a different (cross-context) setup than the main benchmark,
  and the ablation showing one-hot encoding matches embedding performance undercuts
  the study's own framing in that regime. Chemical results are largely negative under
  LFC and only positive under DEG; the paper does not clearly reconcile which
  formulation reflects biological reality. Full-Fusion architectural details are
  deferred entirely to Supplemental Methods.

## Take / open questions
Does the interactome advantage survive controlled architecture swaps, or is it a
GNN-vs-transformer effect? Would a biologically supervised small-molecule FM (e.g.
trained to predict LINCS or Tahoe responses directly) close the chemical gap that
SMILES FMs and target encodings both miss? Does fusion still help once train/test
splits are made harder (e.g., held-out pathways instead of held-out genes)?

## relates_to_prior
- target: Ahlmann-Eltze et al. 2025 (linear-baseline critique); axis: evaluation
  rigor / dataset scale; shown (reproduce their null result on Norman, then show
  differences emerge on the larger Essential dataset).
- target: Wu et al. PerturBench 2024; axis: same as above; shown (same reasoning).
- target: GEARS (Roohani et al. 2024); axis: generalization to unseen perturbations;
  shown (ran as a baseline in the advanced-methods comparison; did not beat kNN +
  best embedding).
- target: STATE (Adduri et al. 2025); axis: cross-context prediction; shown but
  qualified (embedding-based MLP matches or beats STATE in 3/6 settings, but a
  one-hot ablation matches embedding performance).
- target: scGPT, Geneformer, scFoundation, TranscriptFormer, scPRINT, AIDO.Cell;
  axis: utility of scRNA foundation models for perturbation prediction; shown
  (evaluated as embeddings; underperform interactome-based embeddings but larger
  AIDO.Cell variants clearly beat PCA).
- target: CellFlow, Squidiff, ARTEMIS, dbDiffusion, DEPARTURES; axis: value of
  distribution-matching generative models; asserted (only "simple implementations"
  of Latent Diffusion, Flow Matching, Schrödinger Bridge are compared, not the
  specific published algorithms).
- target: SMILES foundation models (ChemBERTa, Uni-Mol, MolT5, MiniMol); axis:
  biological function transfer; shown (evaluated on Tahoe/Sciplex DEG; mostly
  underperform target-based and even fingerprint approaches).
