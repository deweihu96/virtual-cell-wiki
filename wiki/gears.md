---
model: GEARS
paper_title: "Predicting transcriptional outcomes of novel multigene perturbations with GEARS"
authors: Roohani, Huang, Leskovec
year: 2023
venue: Nature Biotechnology
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.1038/s41587-023-01905-6
  code: https://github.com/snap-stanford/GEARS
  weights: not stated
---

# GEARS

## TL;DR
A graph neural network that predicts post-perturbation single-cell gene expression for single- or multi-gene CRISPR perturbations, including combinations where one or both genes were never perturbed in training, by leaning on a gene coexpression graph and a Gene Ontology graph.

## What it is
Supervised GNN, trained per dataset (not a foundation model). Input: unperturbed gene expression vector of a cell plus the set of perturbed gene identities. Output: predicted post-perturbation expression vector over the same genes.

## Training data
Trained and evaluated on public Perturb-seq datasets, one model per dataset. Main datasets used:
- Norman et al. 2019 (K562, CRISPRa): 131 two-gene combinations plus single-gene perturbations across 102 genes.
- Replogle et al. 2022: K562 (1,092 perturbations, >170k cells) and RPE-1 (1,543 perturbations, >170k cells).
- Additional evaluation datasets: Dixit et al. 2016, Adamson et al. 2016, Jost et al. 2020, Tian et al. 2019, Replogle et al. 2020, Horlbeck et al. 2018.
All human cell lines. Not pretrained on a broad corpus; each model is trained on one screen.

## Core mechanism
Two learnable embeddings per gene: a "gene" embedding and a "gene perturbation" embedding. A gene coexpression graph (Pearson-correlation edges) and a GO-derived perturbation-similarity graph (Jaccard over shared GO pathways) supply the graph structure. A GNN propagates information over each graph, letting the model transfer predictions to genes that were never experimentally perturbed by leaning on their graph neighbors. Perturbation embeddings are summed across the perturbation set and combined with each gene's embedding, then decoded with a cross-gene MLP and gene-specific heads.

## Implementation (paper)
- Input representation: unperturbed expression vector plus a set of perturbed gene indices; composition of multi-gene perturbations by summing perturbation embeddings then MLP
- Vocabulary: per-dataset gene set (K genes); no fixed cross-dataset vocab
- Architecture: two GNN encoders (one over coexpression graph, one over GO graph), sum + MLP composition operator, cross-gene MLP, gene-specific linear decoders; layer counts, hidden dim d, and parameter count not stated in the main text
- Graph construction hyperparameters: top-Hgene coexpressed neighbors above threshold δ; top-Hpert GO neighbors by Jaccard. Values of Hgene, Hpert, δ not stated in the main text
- Training objective: autofocus direction-aware loss L = L_autofocus + λ L_direction. Autofocus raises the per-gene error to exponent (2+γ) to upweight differentially expressed genes; direction-aware term penalizes sign mismatch versus control. Values of γ and λ not stated in the main text
- Optional Bayesian head: extra gene-specific layer predicts log-variance, trained with a Kendall-Gal-style heteroscedastic loss, giving a per-prediction uncertainty score
- Optimizer and schedule: stochastic gradient descent; specific optimizer, learning rate, batch size, and epoch count not stated
- Preprocessing: not stated in the main text
- Fine-tuning / adaptation: not applicable; a fresh model is trained per dataset. Multi-gene prediction requires some combinatorial perturbations in training
- Inference: predict per-gene perturbation effect vector ẑ and add to a randomly sampled unperturbed control cell to get post-perturbation expression
- Compute: not stated
- Reproducibility: model code released at github.com/snap-stanford/GEARS; reproducibility code at github.com/yhr91/GEARS_misc; training configs not stated in the paper text

## Implementation (code)
Read from snap-stanford/GEARS, `gears/model.py` and `gears/gears.py`. These are library defaults, not necessarily the exact values used for every figure.
- Hidden dim d: 64 (`hidden_size`)
- Decoder hidden dim: 16 (`decoder_hidden_size`)
- GNN layers: 1 for the coexpression graph, 1 for the GO graph; layer type is `SGConv` (Simplified Graph Convolution, K=1) from PyTorch Geometric
- Graph construction: Hgene = 20 top coexpressed neighbors, Hpert = 20 top GO-Jaccard neighbors, coexpression Pearson threshold δ = 0.4
- Direction-loss weight λ = 1e-1 (`direction_lambda` in `model_initialize`; note the utils `loss_fct` default is 1e-3, overridden here)
- Autofocus loss uses `(y - ŷ)^2 * |y|^γ`-style weighting via a `pert_track` dict summing perturbation embeddings, then `pert_fuse` MLP applied to the sum
- Optimizer: Adam, lr = 1e-3, weight_decay = 5e-4
- LR schedule: StepLR, step_size = 1, gamma = 0.5 (LR halves every epoch)
- Epochs: 20 (default)
- Batch size: user-supplied at `get_dataloader` time (no library default)
- Gene and perturbation embeddings: `nn.Embedding` with `max_norm=True`; positional embedding added at fixed weight 0.2
- Uncertainty head: optional MLP producing per-gene log-variance, gated by `uncertainty` flag

## Evaluation
Task: predict post-perturbation expression for held-out perturbations.
- Single-gene: on Replogle K562 and RPE-1, evaluated by top-20 DE gene MSE, all-gene Pearson correlation of delta expression, and fraction of top-20 DE genes predicted in the wrong direction. Baselines: no-perturbation (zero change), a GRN baseline adapted from CellOracle (SCENIC-inferred network with linear propagation), and CPA. Reports ~30-50% MSE reduction on top-20 DE, ~2x Pearson on all genes, and lower wrong-direction rate versus all baselines.
- Two-gene (Norman): three regimes by how many of the two genes were seen in training (0/2, 1/2, 2/2 unseen). Reports >30% MSE reduction across all three regimes, up to 53% when both genes are unseen; up to +102.9% Jaccard on top-DE gene set versus CPA.
- Genetic-interaction subtypes (synergy, suppression, neomorphism, redundancy, epistasis, defined by vector-geometry rules on delta expression): precision@10 improved by >40% on four of five subtypes versus CPA; >90% on redundancy and epistasis. Top-10 accuracy roughly doubled. GI-score R^2 versus truth reported as ~0.36-0.53 for GEARS vs ~0-0.31 for CPA.
- New-phenotype claim: predicted an erythroid-marker-high cluster among 5,151 pairwise combinations of 102 Norman-et-al. genes that was not present in training, compared descriptively to Tabula Sapiens proerythroblasts. Not experimentally validated.
- Cell-fitness cross-validation: predicted GI scores align in sign with a separate combinatorial cell-fitness screen (Horlbeck-style), one-sided t-test p<0.0013 for synergistic vs additive, p<4e-5 for buffering vs additive.

Notable omission: no comparison against scGen or later cell foundation models (scGPT, etc.); CPA is the only deep-learning baseline in the main figures.

## Claims (authors)
- The authors report that GEARS predicts outcomes for perturbations involving genes never experimentally perturbed, by using coexpression and GO graphs as inductive bias.
- The paper claims 30-50% top-20 DE MSE improvement over CPA and GRN/no-perturb baselines on single-gene prediction, and >30% on two-gene prediction, with the largest gain (~53%) in the hardest 2/2-unseen regime.
- The paper claims 40% higher precision@10 than CPA on four of five genetic-interaction subtypes and roughly twofold better identification of the strongest interactions.
- The paper claims the model can propose new post-perturbation phenotypes not present in training, exemplified by an erythroid-marker cluster.
- Ablations reported in the paper attribute gains to the cross-gene layer and the autofocus loss more than to either individual knowledge graph.

## Limitations
- Author-stated: reliable predictions require training on the same cell type and condition; combinatorial perturbation data must be in the training set to predict multi-gene outcomes; performance degrades for perturbed genes that are poorly connected in the GO graph; confounders like cell cycle, editing efficiency, and post-perturbation heterogeneity affect accuracy.
- Reviewer: no comparison against scGen or later single-cell foundation models on the same benchmarks, so the "state-of-the-art" framing is against CPA and GRN baselines only. Key implementation hyperparameters (embedding dim, layer count, graph thresholds Hgene/Hpert/δ, loss weights γ/λ, optimizer, learning rate, batch size, compute) are not stated in the paper text; run is not reproducible from the paper alone without reading the code. The new erythroid phenotype claim is not experimentally validated. GI-score evaluation is done on a leave-one-out basis over the 131 Norman combinations, a small denominator; the precision@10 gains are on small top-k slices.

## Take / open questions
Does GEARS still lead when compared against scGen and against current cell foundation models fine-tuned for perturbation prediction on the same Norman and Replogle splits? How much of the gain is the two knowledge graphs and how much is the autofocus + direction-aware loss? The paper's ablation reports both matter but the loss ablation shows a larger swing than either graph ablation.

## relates_to_prior
- target: CPA (Lotfollahi 2023); axis: generalization to unseen perturbed genes and to multi-gene combinations; shown (ran as a baseline across MSE, Pearson, Jaccard, GI precision@10, GI R^2)
- target: GRN-based methods, specifically CellOracle-style linear propagation (Kamimoto 2023) and SCENIC (Aibar 2017); axis: nonlinear/non-additive multi-gene response and scalability; shown (ran as a baseline; also asserts scalability advantage in Supplementary Table 3)
- target: scGen (Lotfollahi 2019); axis: perturbation-response prediction; asserted (cited as prior work, not compared as a baseline in main figures)
- target: Norman et al. 2019 manifold; axis: coverage of genetic-interaction subtypes beyond synergy/buffering; asserted (claims broader interaction taxonomy captured but not benchmarked head-to-head)
