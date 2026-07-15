---
model: scPerturBench
paper_title: "Benchmarking algorithms for generalizable single-cell perturbation response prediction"
authors: Wei, Wang, Gao et al. (Liu lab, Tongji)
year: 2026
venue: Nature Methods
task: other
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.1038/s41592-025-02980-0
  code: https://github.com/bm2-lab/scPerturBench
  weights: not applicable (benchmark, no released model)
---

# scPerturBench

## TL;DR
Benchmark of 27 single-cell perturbation response prediction methods across 29
datasets and 6 metrics, split into two task families: cellular-context
generalization (predict a known perturbation in a new cell type / patient / species)
and perturbation generalization (predict an unseen perturbation in a known cell
context). No single method wins across the board; foundation models help only when
fine-tuning data is large; all methods degrade sharply when the test context is
dissimilar to training.

## What it is
A comparative evaluation, not a new predictor. The authors curated datasets, ran 23
published methods plus 4 baselines under a unified preprocessing / split / metric
pipeline, and produced a decision-tree recommendation of which method to use for
each data regime. They also propose a preliminary "cellular context embedding"
strategy as a proof of concept for improving cross-context generalization, but it
is not the main contribution.

## Training data
Not applicable in the usual sense; the paper is a benchmark. Datasets used:

Cellular-context generalization (12 datasets, 477,677 cells, 73 perturbations):
- Cross-cell-line: KangCrossCell, Parekh, Haber, KaggleCrossCell, McFarland,
  Afriat, TCDD, sciPlex3.
- Cross-patient: KangCrossPatient, KaggleCrossPatient, CrossPatient.
- Cross-species: CrossSpecies (mouse, rat, rabbit, pig).
- 7 single-condition datasets, 5 multi-condition (with time-point or dosage covariates).

Perturbation generalization (17 datasets, 1,588,215 cells, 4,874 perturbations):
- 13 genetic: Adamson, Frangieh, TianActivation, TianInhibition, Replogle-exp6/7/8,
  Papalexi, Replogle-RPE1essential, Replogle-K562essential (singles); Norman,
  Wessels, Schmidt, Replogle-exp6 (doubles).
- 4 chemical: sciPlex3-A549/MCF7/K562 (singles), sciPlex3-comb (doubles).
- Perturbation counts per dataset range 25 to 1,618.

## Core mechanism
Not a mechanism paper. The framework has three components:

1. Task taxonomy. Two orthogonal generalization axes (new context vs. new
   perturbation), plus i.i.d. vs. o.o.d. splits within each. o.o.d. uses
   leave-one-out on the generalization axis. Combined perturbations are further
   split into combo-seen0/1/2 by whether zero, one, or both single perturbations
   appear in training.

2. Metric selection. From a survey of 28 tools and 19 candidate metrics, the
   authors keep 6 organized into two families:
   - Population-average: MSE, PCC-delta (Pearson on delta expression, GEARS-style),
     E-distance.
   - Population-distribution: Wasserstein, KL-divergence, Common-DEGs (overlap of
     top-100 predicted vs. true differentially expressed genes).
   Metrics like RMSE, MMD, cosine, PCC, R2 are dropped as redundant with MSE. Final
   rank is average of per-metric ranks across datasets.

3. Baselines. Four non-learned or trivially-learned models (baseReg, baseMLP,
   baseControl, trainMean) are included as required floor comparisons. trainMean
   is the mean training-set delta.

## Methods evaluated

Cellular-context generalization (14 methods): scGen, inVAE, trVAE, scVIDR, scPRAM,
biolord, scDisInFact, CellOT, SCREEN, scPreGAN + baselines (trainMean, baseControl,
baseMLP, baseReg).

Perturbation generalization, genetic (15 methods): CPA, biolord, scouter,
AttentionPert, scELMo, GeneCompass, scGPT, scFoundation, GEARS, GenePert,
linearModel + baselines.

Perturbation generalization, chemical (9 for singles, 6 for doubles): CPA, biolord,
chemCPA, PRnet, cycleCDR + baselines. chemCPA / biolord / cycleCDR are
single-perturbation only.

## Implementation (paper)
- Preprocessing (unified across methods): cells with <200 genes or >10% mito
  removed; genes expressed in <3 cells removed; perturbations with <30 cells
  removed; perturbations with >2,000 cells subsampled to 2,000. Retain top-5,000
  HVGs selected on training set only (to prevent leakage), forcing inclusion of any
  perturbed gene not in that set. Global scaling to 10,000 counts then log1p (note
  this differs from the target-4,000 convention used by e.g. TxPert / GEARS).
- Metric implementation: MSE and E-distance via Pertpy; Common-DEGs via Scanpy
  default DE ranking; Wasserstein and KL as standard.
- For CRISPRi datasets, cells with target expression below 70% of control are
  additionally filtered.
- Simulated robustness sweeps: Poisson noise at rate 0.1/0.3/0.5/0.7/0.9 on the
  original expression; sparsity dropout at 0/0.1/0.3/0.5/0.7/0.9. Scalability
  sweeps: 5k/20k/50k/150k cells; 1, 3, 9 perturbations (cellular-context) or 50
  perturbations across cell counts (perturbation-generalization).
- Compute: 96-core CPU, 1TB RAM, 4x NVIDIA 3090 24GB. scFoundation run separately
  on 2x A6000 48GB.
- Software: Python 3.9.16, PyTorch 2.0.1, Scanpy 1.9.3, Pertpy. Podman image with
  all conda environments released.

## Evaluation
Ranking summary (o.o.d. setting, top-3 per task):

- Cellular-context, single-condition datasets: trVAE, CellOT, inVAE.
- Cellular-context, dosage-aware datasets (TCDD, sciPlex3): scVIDR best.
- Cellular-context, best on Common-DEGs specifically: scPRAM.
- Genetic single perturbation, small training data: GenePert (linear regression
  over ChatGPT gene embeddings) and scouter (small MLP over ChatGPT embeddings).
- Genetic single perturbation, large training data (Replogle-RPE1essential /
  K562essential): CPA best on MSE / PCC-delta, scGPT competitive.
- Genetic combined perturbations: linearModel and scouter best overall;
  linearModel/trainMean best on additive gene interactions, CPA best on
  potentiation / epistasis / redundant interactions.
- Chemical single perturbations: chemCPA best.
- Chemical combined perturbations (sciPlex3-comb): baseReg (a baseline)
  outperforms CPA/PRnet on all six metrics.

Cross-cutting findings:
- Performance in o.o.d. is strongly and monotonically correlated with
  inter-heterogeneity (E-distance between test and nearest training context):
  r = 0.71 for MSE, -0.47 for PCC-delta.
- All methods score much worse on population-distribution metrics than on
  population-average metrics. On genetic single perturbations, even the
  best-performing baseMLP recovers <10 of 100 true top-DEGs.
- Ablation on GenePert/scouter: swapping ChatGPT-derived gene embeddings for
  scGPT / geneformer / scBERT / GeneCompass transcriptome-derived embeddings
  degrades performance markedly. LLM-derived embeddings beat transcriptome
  foundation model embeddings under matched downstream architectures.
- scFoundation ran out of memory on the two largest datasets
  (Replogle-RPE1essential, Replogle-K562essential); CellOT the only method with
  scalability issues in the cellular-context scenario (no GPU support).

## Claims (authors)
- No single method generalizes well across all datasets; the choice depends on
  data regime.
- Foundation models beat baselines only when fine-tuning datasets are large;
  simple / linear baselines win on small datasets. This is presented as an
  extension and partial vindication of Ahlmann-Eltze 2025's linear-baseline
  finding, showing it holds mainly in low-data regimes.
- The performance advantage of GenePert / scouter comes from LLM-derived gene
  embeddings, not model architecture (matched-architecture ablation).
- All methods struggle with population-distribution metrics; loss functions
  focused on averages are the likely cause, so future models should include
  distribution terms.
- Cross-context generalization is fundamentally bottlenecked by
  inter-heterogeneity: adding a training context similar to the test context
  helps a lot, adding a dissimilar one does not help.
- Multi-condition (dosage / time) covariates carry real signal, but current
  methods that use them (biolord, scDisInFact) are still weak; scDisInFact was
  designed for batch integration, not perturbation prediction, and does not beat
  its own no-covariate baseline.
- A prototype cellular-context-embedding strategy improves prediction on
  cell-line-derived contexts (PCC-delta and Common-DEGs), but not consistently
  across all metrics; presented as proof-of-concept only.

## Limitations
- Author-stated: some test scenarios have too few datasets to be conclusive;
  the proposed context-embedding strategy is limited to cell-line contexts and
  gives inconsistent gains; population-distribution performance not addressed by
  the paper's own proposal; robustness only sweeps technical noise/sparsity, not
  biological variability.
- Reviewer: metrics computed on top-100 differentially expressed genes make the
  numbers task-specific; results on all genes are only on the companion website.
  All datasets are downsampled to 2,000 cells per perturbation and 5,000 HVGs,
  which is fine for comparability but caps what any method can learn from the
  larger perturbation atlases. The 10,000-count normalization differs from
  target-4,000 conventions used by some of the methods being benchmarked (GEARS,
  TxPert, CPA) - each method is then re-trained under this pipeline, so results
  do not always reproduce the numbers from the original method papers (the
  authors note the discrepancy with Ahlmann-Eltze 2025 as such an artifact). More
  recent perturbation methods (State, CellFlow, TxPert) are not included.
  Combined-perturbation coverage is thin: 4 genetic combo datasets, 1 chemical
  combo dataset. LLM-embedding ablation compares different pretrained embeddings
  but not the same LLM with vs. without biomedical fine-tuning, so the "LLM
  priors" conclusion is under-attributed.

## Take / open questions
The most interesting finding is the LLM-embedding ablation: matched downstream
architectures, only the input embedding varies, and ChatGPT-derived gene text
embeddings beat scGPT / geneformer transcriptome embeddings. If reproducible on
newer transcriptome foundation models (State, C2S-Scale), it argues that current
scRNA foundation pretraining is not extracting perturbation-relevant priors that
LLMs pick up from gene text. Worth testing directly.

The paper is also a useful reference index for which datasets are used by which
methods and where to obtain them (PerturBase, scPerturb, Figshare, Zenodo).

## relates_to_prior
- target: Ahlmann-Eltze 2025 (Deep learning does not outperform linear baselines);
  axis: evaluation rigor; shown - extends the finding, agrees for small-data
  regimes and reverses it for large-data regimes.
- target: PerturBench (Wu 2024), Bendidi 2024 ("one PCA still rules them all");
  axis: evaluation rigor; adopted - cites both as motivation and includes their
  baseline-inclusion practice.
- target: GEARS (Roohani 2024); axis: evaluation methodology; shown - re-benchmarks
  GEARS under unified preprocessing, reports GEARS competitive on genetic singles
  but not top; adopts PCC-delta from GEARS as a primary metric.
- target: scGPT (Cui 2024), scFoundation (Hao 2024); axis: whether foundation
  models help perturbation prediction; shown - foundation models help only when
  fine-tuning data is large; scGPT wins on combo-seen0 in Schmidt but scFoundation
  hits OOM on the largest datasets.
- target: GenePert / scouter / scELMo (LLM-embedding methods); axis: source of
  gains; shown - matched-architecture ablation attributes gain to LLM-derived
  embeddings rather than model design.
- target: biolord, scDisInFact, scVIDR (covariate-aware methods); axis: whether
  including dosage/time covariates helps; shown - covariates help when the
  underlying model is already strong (biolord, scVIDR), do not rescue a
  weaker-for-this-task model (scDisInFact).
- target: Ji et al. 2023 (metric benchmarking); axis: evaluation methodology;
  adopted - metric selection follows their ranking, extended with a redundancy
  filter.
