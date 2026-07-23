---
model: Stable-Shift
paper_title: "Stable-Shift: Biologically Structured Prediction of Transcriptional Responses to Unseen Gene Perturbations"
authors: Dip and Zhang (Virginia Tech)
year: 2026
venue: ACM BCB 2026
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://arxiv.org/abs/2606.24940
  code: https://github.com/Sajib-006/PerturbGraph
  weights: not stated
---

# Stable-Shift

## TL;DR
A perturbation-level method for predicting the transcriptional response of a gene
never perturbed in training: fit a low-rank response basis on training
perturbations only, then map per-gene biological context (STRING graph, Node2Vec,
control-cell stats, GO) into coordinates in that basis via graph convolution and
decode to a genome-wide expression shift.

## What it is
A structured regressor for unseen-gene perturbation prediction. Input is per-gene
biological context available for every gene (not a learned perturbation identity):
STRING interaction graph, Node2Vec structural embeddings, control-cell expression
statistics and graph summaries, and a GO membership embedding. Output is a predicted
latent response program that decodes (via a fixed training-derived SVD basis) into a
genome-wide expression-shift vector. Graph convolution is the encoder in the
evaluated implementation, but the paper frames the contribution as the whole
context→latent-program→decode design, not the GCN operator.

## Training data
K562 essential-gene Perturb-seq (Replogle et al.): ~3.1x10^5 cells, 8,563 measured
genes, 1,832 targeted genes, plus non-targeting controls. A second dataset (Norman)
is used as an additional evaluation context. Perturbation genes are split
~70/15/15 into disjoint train/val/test gene sets, with identical partitions across
all compared models. Biological priors (STRING, GO, pathways, control-cell stats)
are treated as external knowledge available before the experiment and may include
test genes.

## Core mechanism
The defining choice is a leakage-controlled response target: the low-rank basis (a
rank-K truncated SVD of stacked training treatment-effect vectors) and all
preprocessing statistics are fitted on the training partition only, never seeing
held-out perturbation responses. Because an unseen test gene has no measured
response and no learnable identity, prediction must come entirely from attributes
available for every gene. Stable-Shift therefore predicts the test gene's
coordinates in the training-derived basis from its biological context, then decodes.
This is what makes strict unseen-gene extrapolation possible where identity-based
latent models (scGen, CPA) cannot transfer.

## Implementation (paper)
- Prediction target: perturbation-level pseudobulk shift Δ_i = perturbed mean minus
  control mean, in R^d. Stack training shifts, take rank-K truncated SVD X_train ≈ HV;
  rows of H are latent programs, V is the decoder. Latent dimension K chosen by
  validation.
- Node features: concatenation of (1) Node2Vec embeddings of the STRING graph,
  (2) control-cell statistics + graph summaries (baseline mean expression, variance,
  detection frequency, degree, centrality, local neighborhood), (3) low-dimensional
  GO membership embedding. All response-derived transforms fitted on train partition.
- Graph: weighted gene graph from STRING protein-protein associations; nodes = genes,
  edge weights = interaction-confidence scores; self-loops added, adjacency
  symmetrically normalized.
- Architecture: graph convolution layers (GCN) over normalized adjacency, then an
  MLP projection to the predicted latent program. Depth/width not stated in the paper.
- Training objective: MSE between predicted and observed latent programs, over
  training genes only.
- Optimizer and schedule: Adam with validation-based early stopping; learning rate,
  batch size, epochs not stated in the paper.
- Preprocessing: normalized single-cell expression averaged per perturbation into
  pseudobulk shifts; cell-level baseline outputs (scGen/CPA/GEARS) averaged and
  converted to shifts using the same control reference.
- Inference: predict latent program ĥ_i from the gene's context, decode Δ̂_i = ĥ_i V.
- Compute: not stated.
- Reproducibility: source code, pipeline, configs, and supplement released.

## Implementation (code)
From `stable_shift_bench.py` in the PerturbGraph repo (values are code defaults):
- SVD response basis: `n_components = 128` (rank-K), TruncatedSVD, seed 42, fit on
  train rows only via `build_svd_targets_train_only`.
- Encoder `WeightedGCN_MLP`: two `GCNConv` layers (in_dim→hidden→hidden), then MLP
  hidden→512→out; `hidden_dim = 256`, `dropout = 0.15`.
- Training: `lr = 1e-3`, `weight_decay = 1e-4`, `max_epochs = 300`, `patience = 12`,
  Adam, seed 42 (the paper's "not stated" schedule values).
- Node2Vec: `dim = 128`, `walk_length = 30`, `num_walks = 100`.
- GO embedding SVD dim = 64; pathway embedding SVD dim = 32.
- Pseudobulk: `min_cells_per_pert = 50`; STRING edges weighted by `combined_score`
  (confidence), self-loops added, no single hard confidence cutoff shown in the graph
  builder.

## Evaluation
- Primary K562 unified unseen-gene benchmark (Table 1, per-perturbation averaged):
  Stable-Shift cosine 0.592 vs GEARS 0.569 and feature-only MLP 0.565; also higher
  Spearman (0.340) and Prec@50 (0.277) and lower MSE. Baselines: Random Forest,
  scGen, CPA, GEARS, MLP, GraphSAGE (also Lasso/ElasticNet/GAT in secondary tables).
- Robustness: 5 unseen-gene splits, mean cosine 0.589 ± 0.008 vs CPA 0.566 ± 0.010,
  RF 0.555 ± 0.012; paired two-sided test vs CPA p < 0.01 (only 5 partitions, treated
  as supporting evidence).
- Stress tests preserve ordering: graph-aware split 0.558 vs GraphSAGE 0.535 / CPA
  0.528; residualized (dominant components removed) 0.285 vs CPA 0.232; gene-space
  reconstruction 0.392 vs CPA 0.375 (much lower than latent space); Norman dataset
  cosine 0.940 / Spearman 0.815 vs MLP 0.922 / 0.801.
- DE recovery (Table 2, controlled run): directional accuracy 0.619, DE AUROC 0.784,
  DE AUPRC 0.253, Prec@50 0.292 — small gains over GraphSAGE on direction/AUROC,
  larger on AUPRC/Prec@50.

## Claims (authors)
- The authors report Stable-Shift has the highest values among compared feature,
  graph, and perturbation baselines on the supplied K562 and Norman benchmarks across
  cosine, Spearman, Prec@50, and MSE.
- The authors claim the training-only low-rank target avoids leakage from held-out
  perturbation measurements, making the unseen-gene setting honest.
- The authors claim the combined context (interaction + expression + topology + GO)
  design, not a new graph operator, is the contribution; ablations show adding
  expression/topology and then GO features each improve the graph-only variant.

## Limitations
- Author-stated: evidence is strongest for perturbation-level pseudobulk in K562;
  the method does not model cell-to-cell heterogeneity; the graph prior inherits
  missing interactions and annotation bias from STRING/GO; performance degrades in
  sparse graph neighborhoods and after full gene-space reconstruction (0.392 vs 0.592
  latent). Norman is only a second context, not proof of cross-cell-type transfer.
  Protein interactions are not transcriptional regulation.
- Reviewer: absolute gains over the strongest baseline are small (+0.023 cosine over
  GEARS on the primary benchmark; +0.005–0.014 over GraphSAGE on DE metrics), and the
  paper repeatedly cautions these are single-benchmark differences, not a ranking.
  Statistical support rests on 5 partitions. The largest reported number (Norman
  cosine 0.940) reflects a dataset dominated by a shared response direction that even
  the plain MLP nearly matches (0.922), so global cosine likely overstates biological
  fidelity — the latent-to-gene-space gap the authors flag is the real signal.

## Take / open questions
The honest framing is the strength here: the paper foregrounds the latent-vs-gene-
space gap and treats its own margins as provisional. Open question: does the graph
context actually cause the gain, or is the training-only low-rank target plus good
node features enough? The MLP with the same features (0.565 cosine) sits close to
Stable-Shift (0.592), so the marginal value of graph propagation over the shared
feature set is the key unresolved question.

## relates_to_prior
- target: GEARS; axis: unseen-gene generalization; shown (ran as baseline, cosine
  0.592 vs 0.569). GEARS also uses a gene interaction graph; Stable-Shift differs in
  target (training-derived low-rank program, not identity propagation).
- target: CPA; axis: identity-based interpolation vs attribute-based extrapolation;
  shown (ran as baseline across splits and stress tests). Paper argues CPA's learned
  perturbation identity gives no evidence for an excluded test gene.
- target: scGen; axis: latent-arithmetic transfer; shown (ran as baseline).
- target: GraphSAGE / GAT (graph-only encoders); axis: graph aggregation vs combined
  context design; shown (ran as baselines, Stable-Shift higher). Argues a graph-only
  model inherits noise/incompleteness from one resource.
- target: feature-only MLP; axis: neighborhood propagation value; shown (ran as
  baseline, close margin 0.592 vs 0.565) — the comparison the paper leans on but that
  most weakly separates the method.
