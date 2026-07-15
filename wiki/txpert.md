---
model: TxPert
paper_title: "TxPert: using multiple knowledge graphs for prediction of transcriptomic perturbation effects"
authors: Wenkel, Tu et al. (Valence Labs / Recursion)
year: 2026
venue: Nature Biotechnology
task: perturbation-prediction
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.1038/s41587-026-03113-4
  code: https://github.com/valence-labs/TxPert
  weights: https://github.com/valence-labs/TxPert (public-data checkpoints released)
---

# TxPert

## TL;DR
A latent-transfer deep learning model that predicts post-perturbation single-cell
transcriptomes by combining a basal-state encoder of a control cell with a GNN over
one or more gene-gene knowledge graphs. Targets three OOD tasks: unseen single
perturbations, unseen double perturbations, and perturbations in an unseen cell line.

## What it is
An encoder-decoder with two branches. A basal-state encoder maps a batch-matched
control expression vector `x` to a latent `s`. A GNN over a gene(product)-gene(product)
knowledge graph produces per-gene node embeddings `z_p` for each perturbation token
`p`. The prediction is `y_hat = g_phi(s + sum_p z_p)`, i.e. additive latent shift
decoded back to gene-expression space (log1p, library-size-normalized). Trained with
MSE against observed perturbed cells.

## Training data
Trained separately per task on public CRISPR screens in human immortalized cell lines:
- K562 and RPE1 CRISPRi essential-gene Perturb-seq (Replogle 2022): ~2,000 perturbations each.
- Jurkat and HepG2 CRISPRi (Nadig 2025): ~2,000 perturbations each.
- Norman 2019 CRISPRa in K562: 94 unique singles + 110 unique doubles (doubles task only).

Knowledge graphs used as inductive bias (four sources combined):
- STRING (protein-protein associations, curated + literature).
- Gene Ontology (Jaccard on GO terms, same construction as GEARS).
- PxMap: proprietary Recursion phenomics (Cell Painting) similarity graph from
  genome-wide screens in HUVECs.
- TxMap: proprietary Recursion single-cell transcriptomic perturbation-similarity graph.

Reactome graph was also considered but not in the best configuration. Cell line count
per training run is one (single-cell-line task) or three (cross-cell-line task with
one held out).

## Core mechanism
Two ideas together: (1) latent additive shift, so a single trained decoder handles
singles, doubles, and higher-order perturbations by summing GNN node embeddings; (2)
multi-graph reasoning, where curated database graphs (STRING, GO) are fused with
graphs derived from high-throughput perturbation screens (PxMap phenomics, TxMap
transcriptomics). The authors report that database + screen-derived graphs are
complementary, and the four-graph combination beats any subset. Attention-based GNNs
(GAT, Exphormer graph transformer) are used to downweight noisy edges and cross long
range gaps; multi-graph variants (Exphormer-MG, GAT-Hybrid, GAT-MLG, Hybrid-BMP)
integrate KGs jointly rather than concatenating outputs.

The framework also codifies evaluation practice the authors argue is necessary in
this domain: batch-matched controls for computing delta expression (not a global
control mean), Pearson delta as the primary metric, and retrieval-rank metrics
(cosine / Pearson on delta profiles, unfiltered) to check that predictions preserve
perturbation-specific signal rather than just the mean-baseline response.

## Implementation (paper)
- Input representation: log1p of library-size-normalized counts, target library size
  4,000; delta representation `y - x_{cell_line, batch}` centered on batch-matched
  controls.
- Vocabulary: perturbation tokens = observed perturbations in the task (up to a few
  thousand); graph nodes = union of perturbation targets. Random-init node embeddings
  refined by GNN message passing.
- Basal-state encoder: MLP on raw expression is the default best; scGPT and scVI
  embeddings and a no-encoder pass-through variant also tested (pass-through best for
  cross-cell-line task with no seen perturbations).
- Perturbation GNN: attention-based; best variants per task -
  * unseen singles -> Exphormer-MG (graph transformer) over STRING + GO + PxMap + TxMap.
  * unseen doubles -> GAT-MLG over GO + PxMap + TxMap (supra-adjacency multilayer graph).
  * cross-cell-line -> variant reported per experiment (see Methods; pass-through basal).
- Training objective: MSE between predicted and observed log1p expression profile.
- Basal matching: control cell sampled or averaged within same cell line and batch as
  the target perturbed cell (batch-matched control matching + averaging both improved
  performance).
- Data splits: perturbation-grouped 0.5625 / 0.1875 / 0.25 train/val/test; Norman
  doubles uses GEARS's predefined splits; cross-cell-line uses leave-one-cell-line-out
  with only controls from the held-out line in training.
- Optimizer, schedule, batch size, epochs: not stated in the main paper text.
- Fine-tuning: not applicable in the usual foundation-model sense; each task is
  trained end-to-end from scratch. Model selection by validation Pearson delta.
- Inference: decoder outputs the perturbed log1p profile; Pearson delta and retrieval
  computed on unseen test perturbations, averaged (weighted) across replicates then
  across perturbations.
- Compute: not stated (hardware, GPU-hours).
- Reproducibility: training + inference code released; weights for the public-data
  variants released. Proprietary PxMap and TxMap graphs are not released, so the
  full multi-graph configuration in the paper is not fully reproducible.

## Implementation (code)
Values pulled from the released `exphormer-mg` config
(`configs/config-exphormer-mg.yaml`) and `gspp/predictor.py`. The released model is a
public-data-only variant (STRING + GO), not the four-graph configuration.

- Model shapes: overall `hidden_dim` 512, `latent_dim` 64, dropout 0.2, batch norm on.
- Basal-state MLP: rank 16, batch norm on.
- Perturbation GNN (Exphormer-MG default): `num_layers` 4, `hidden_dim` 128,
  `num_heads` 2, dropout 0.1, `expander_degree` 3, `add_reverse_edges` true,
  `add_self_loops` true, layer type `exphormer_w_mpnn`, skip `skip_cat`, union edge
  type `multihot`.
- Graph filtering: STRING and GO both filtered to top-20 incoming edges per node
  (`mode: top_20`). Paper additionally reports top-1% by absolute weight for
  screen-derived graphs (PxMap/TxMap).
- Batch size: 64.
- Optimizer (defaults in `predictor.py`): AdamW, `lr=1e-3`, `weight_decay=0.0`,
  optional cosine schedule with linear warmup (`warmup_epochs`, `total_epochs` set
  per run; specific run values not in the released configs).
- Loss: MSE, `mse_weight=1`.
- Preprocessing (README + `data/onboard_dataset.ipynb`): `sc.pp.normalize_total(target_sum=4000)`
  then `sc.pp.log1p`; HVG selection to top 5,000 genes (deferred to intersection step
  in the cross-cell-line setup).
- Available configs: `gat`, `exphormer`, `exphormer-mg`, `x-cell-gat`, `baseline`,
  `x-cell-baseline`. Public checkpoint filenames follow
  `K562_unseen_pert_<arch>.ckpt`.
- Framework: Python 3.12, PyTorch 2.6, Lightning-style predictor; data auto-downloads
  to `./cache`.

## Evaluation
Tasks and baselines:
1. Unseen singles within cell line (K562, RPE1, HepG2, Jurkat) - compared to GEARS,
   scLAMBDA, a general-additive baseline, a batch-only baseline, split-half
   experimental reproducibility. TxPert beats all learned baselines on every cell
   line and matches split-half reproducibility on K562, Jurkat, HepG2. GEARS falls
   below the non-learned general baseline on this metric setup.
2. Doubles on Norman (both singles seen) - compared to GEARS, scLAMBDA, and an
   additive baseline `x + delta_i + delta_j`. TxPert slightly exceeds the additive
   baseline; both far above GEARS and scLAMBDA.
3. Cross-cell-line (leave-one-line-out, only controls in target line) - compared to
   scLAMBDA (adapted), general baseline, and a nearest-cell-line baseline. TxPert
   beats scLAMBDA and general baseline in all four; beats nearest-cell-line in K562
   and RPE1, ties in HepG2.

Metrics: Pearson delta (primary), plus normalized and fast retrieval rank at
quantile 0.9 for perturbation identifiability. Retrieval on delta with cosine or
Pearson chosen empirically over signed-p-value and top-DEG variants.

Ablations: (a) rewiring STRING edges 0-100% degrades performance monotonically;
(b) edge downsampling robust until >60% removed; (c) among single graphs STRING >
PxMap > GO > TxMap on K562-unseen; (d) incremental graph fusion (STRING -> +PxMap ->
+GO/TxMap -> all four) monotonically improves, p<0.027 for four vs best three.
Per-target breakdown by Pharos knowledge rank shows performance rises with target
knowledge for all models including the general baseline, but STRING+PxMap fusion
outscores STRING alone at every knowledge level.

## Claims (authors)
- TxPert reaches state-of-the-art on unseen single-perturbation prediction across
  four CRISPRi cell lines and is competitive with split-half experimental
  reproducibility on three of them.
- The authors report an 8-25% improvement in Pearson delta over existing methods
  for double unseen and cross-cell-line single-perturbation tasks.
- Combining curated (STRING, GO) and screen-derived (PxMap, TxMap) knowledge graphs
  gives complementary signal; using all four is strictly better than any subset.
- Attention-based GNNs (GAT, Exphormer) outperform the simple graph convolution
  used in GEARS in this task.
- Batch-matched controls, retrieval metrics, and inclusion of strong non-learned
  baselines are needed to correctly rank models in this domain; without them,
  earlier deep models were credited with performance they did not have.
- One explicit failure mode acknowledged: the model does not learn the standard
  downregulation of the perturbed target gene itself (an issue several prior
  methods hide by copying ground truth for the target row).

## Limitations
- Author-stated: does not learn the target-gene downregulation for unseen
  perturbations; trained only on (largely cancer-derived) immortalized cell lines
  so generalization to primary tissue is not shown; zero-shot cross-cell-line only,
  no few-shot or active-learning extension yet; metrics do not explicitly evaluate
  the specificity/conditionality of perturbation effects across novel contexts.
- Reviewer: the strongest reported configuration relies on two proprietary graphs
  (PxMap, TxMap) that are not released, so the headline four-graph results cannot
  be reproduced by the community; the released `exphormer-mg` checkpoint is
  STRING + GO only and is not the paper's best variant. Training compute and
  wall-clock are not stated. Task-by-task the "best" architecture differs
  (Exphormer-MG, GAT-MLG, Hybrid-BMP), so what "TxPert" is depends on the task;
  the framework is compared against split-half reproducibility that the paper itself
  notes is a lower bound (0.07-0.11 Pearson delta below the sample-based
  full-size estimate). scLAMBDA and GEARS are the only external learned baselines
  on unseen singles; more recent perturbation models (State, CellFlow, others cited
  in the discussion) are not benchmarked against.

## Take / open questions
Does the four-graph configuration keep its edge when the proprietary maps are
replaced by open perturbation-similarity graphs derivable from Perturb-seq atlases
(Replogle, Nadig, X-Atlas/Orion, Tahoe-100M)? The paper's own ablation shows PxMap
is the second-strongest single graph, so open replacements are the natural next
experiment.

## relates_to_prior
- target: GEARS (Roohani 2024); axis: architecture (simple graph conv vs attention),
  evaluation rigor (global vs batch-matched controls); shown - ran as baseline on
  unseen singles and doubles, TxPert substantially higher Pearson delta and GEARS
  below the general baseline on unseen singles.
- target: scLAMBDA (Wang 2024); axis: prior source (GPT-derived gene text embeddings
  vs biological KGs); shown - ran as baseline on singles, doubles, and adapted for
  cross-cell-line.
- target: mean/additive baselines (Wong 2025, Ahlmann-Eltze 2025, Kernfeld 2025,
  PerturBench); axis: evaluation methodology; asserted and adopted - explicitly
  includes non-learned baselines as required benchmarks and criticizes prior work
  for omitting them.
- target: single-cell foundation models (scGPT, scVI, GenePT); axis: whether pretrained
  cell embeddings help as basal-state encoders; shown - tested as basal encoders, MLP
  on raw expression wins for singles/doubles, pass-through wins for cross-cell-line.
- target: Exphormer / graph transformers (Shirzad 2023, 2025); axis: applicability to
  biological KGs; adopted directly and extended to multi-graph (Exphormer-MG).
