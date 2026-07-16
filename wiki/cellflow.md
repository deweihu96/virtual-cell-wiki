---
model: CellFlow
paper_title: "CellFlow enables generative single-cell phenotype modeling with flow matching"
authors: Klein, Fleck et al. (Theis lab / Helmholtz Munich, Roche)
year: 2025
venue: preprint (bioRxiv)
task: perturbation-prediction
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.1101/2025.04.11.648220
  code: https://github.com/theislab/CellFlow
  weights: not stated
---

# CellFlow

## TL;DR
A conditional flow-matching framework that predicts single-cell phenotypes under arbitrary
combinatorial perturbations by learning a neural vector field that transports a control-cell
distribution to a perturbed distribution, conditioned on an aggregated embedding of the
experimental setup (drugs, gene knockouts, cytokines, dosage, timing, cell line). Applied
across cytokine, drug, gene-knockout, and morphogen screens, including whole-embryo zebrafish
development and organoid protocol design.

## What it is
A generative model operating in a latent cell space (PCA or a scVI-style VAE). Input: a batch
of control cells plus a "condition" (a 3-tuple of perturbations, perturbation covariates, and
sample covariates). A condition encoder aggregates the possibly-combinatorial treatments into
one condition vector via permutation-invariant attention or DeepSet pooling. A time-dependent
feed-forward neural vector field (neural ODE) is trained by flow matching with optimal-transport
pairing to flow control cells (t=0) to perturbed cells (t=1). Output: predicted perturbed
single-cell profiles obtained by integrating the ODE.

## Training data
No single pretraining corpus; trained per screen. Datasets used:
- **Parse-PBMC cytokine screen**: ~10M PBMCs, 12 donors, cytokine treatments (donor-specific
  response prediction).
- **ZSCAPE zebrafish**: whole-embryo perturbation atlas, 23 gene knockouts (F0 editing) across
  developmental stages 18-72 hpf.
- **sciPlex3**: 3 cell lines, many drugs at multiple dosages.
- **combosciplex**: A549 lung cancer line, 31 single or combinatorial drug treatments.
- **Norman 2019** (genetic, K562) and **Srivatsan 2019** (drug) for established benchmarks.
- **iNeuron combinatorial morphogen screen** and an **organoid** morphogen screen (fate
  engineering, virtual protocol screen); some data newly generated.
- Organisms: human and zebrafish. Modality: scRNA (plus a 4i imaging application via CellOT data).

## Core mechanism
The distinguishing idea is flow matching + optimal transport for perturbation, plus a generic
condition abstraction. Rather than reconstructing a mean response, CellFlow learns a continuous
vector field that carries the whole control distribution onto the whole perturbed distribution,
trained simulation-free by regressing the vector field onto predefined interpolation paths.
Optimal-transport (including unbalanced OT, which relaxes mass conservation to allow apoptosis /
proliferation) pairs control and perturbed cells within each condition to straighten the flow.
The condition encoder unifies drugs, genes, cytokines, dosage, timing, and cell state into one
permutation-invariant embedding using pretrained biological embeddings (ESM-2 for genes and
cytokines, molecular fingerprints for drugs, CCLE for cell lines), so it can extrapolate to
unseen individual perturbations and unseen combinations. This is what lets one framework span
cytokines, drugs, knockouts, and morphogens including whole-organism and organoid scales.

## Implementation (paper)
- Input representation: cells embedded into a latent space (PCA, or a VAE adapted from scVI's
  JAX implementation with a normal likelihood); condition = (perturbations, perturbation
  covariates, sample covariates)
- Perturbation/condition encoding: ESM-2 embeddings for genes and cytokines; molecular
  fingerprints for drugs; CCLE embeddings for cancer cell lines; donor control-population
  statistics for patients; one-hot fallback (allows unseen combinations but not unseen singles);
  scalar covariates (dose, timing) log-transformed; aggregation by multi-head attention or
  DeepSet (feature-wise mean)
- Architecture: time-dependent feed-forward neural vector field. Default config: time encoder
  dims [2048, 2048, 2048]; source/cell MLP hidden_dims [4096, 4096, 4096]; decoder_dims
  [4096, 4096, 4096]; dropout 0.0; VAE best config 10 latent dims, 2048 hidden units
- Training objective: conditional flow matching (regress vector field onto OT-interpolated paths
  between control and perturbed latent cells); squared-Euclidean OT cost; unbalanced OT (KL
  divergence) option to break mass conservation
- Optimizer and schedule: Adam (optax); batch_size 1,024 (conditions, not cells, per update);
  num_iterations 500,000 (dataset-tuned); learning rate not stated in the main text
- Inference: sample control cells, integrate the vector field t=0->1 with a numerical ODE solver,
  decode from latent space back to expression
- Software: JAX / flax / optax / diffrax; ott-jax for OT couplings; scverse (anndata, scanpy,
  rapids-singlecell) for data
- Compute: hardware / GPU count / GPU-hours not stated
- Reproducibility: model code at github.com/theislab/CellFlow; analysis-reproduction code in a
  companion repo; several datasets public via pertpy, CellOT repo, and provided sources

## Implementation (code)
Cloned github.com/theislab/CellFlow (~13k Python LOC). Library defaults recovered:
- Default optimizer (`src/cellflow/model/_cellflow.py:270`): `optax.MultiSteps(optax.adam(5e-5),
  20)`, i.e. Adam at lr 5e-5 with 20 accumulation steps for the condition encoder. The main text
  did not state a learning rate; 5e-5 is the code default.
- Velocity field defaults (`src/cellflow/networks/_velocity_field.py`): condition_embedding_dim
  32; time_encoder_dims, hidden_dims, and decoder_dims all (1024, 1024, 1024); pooling
  `attention_token`; activation SiLU (`nn.silu`); time_freqs 1024; conditioning `concatenation`;
  dropout 0.0. These library defaults are **smaller** than the per-experiment config reported in
  the paper methods (hidden/decoder [4096]^3, time [2048]^3), so the paper runs override the
  defaults.
- Solver (`src/cellflow/solvers/_otfm.py`): `OTFlowMatching` with a pluggable `match_fn`
  (e.g. `utils.match_linear`, linear OT via ott-jax); a separate `GENOT` solver
  (`solvers/_genot.py`) provides the stochastic / unbalanced variant.
- Metrics (`src/cellflow/metrics/_metrics.py`): Sinkhorn divergence at epsilon 1 / 10 / 100 plus
  e-distance, matching the paper's latent-space distributional metrics.
- `num_iterations` is a required trainer argument (`training/_trainer.py`), consistent with the
  paper's per-dataset 500,000-iteration setting; batch_size 1,024 conditions per update.

Net: framework and defaults are fully public; exact per-figure configs (which override the small
library defaults) are in the reproduction repo rather than the core package. Compute (GPU
type/count/hours) is not recorded in code.

## Evaluation
- Tasks and baselines:
  - Donor-specific cytokine response (Parse-PBMC): vs identity, a "consistent effect across
    donors" baseline, and a mean baseline. CellFlow beats baselines once a few donors are seen
    per cytokine; with only control-population mean as the donor representation it still wins.
  - Zebrafish development (ZSCAPE): vs identity + two mean/transfer baselines; CellFlow beats all
    across time points; cell-type-proportion accuracy 0.89 vs 0.85 best baseline.
  - sciPlex3 drug response: vs chemCPA and other established methods.
  - combosciplex drug combinations: vs GEARS, biolord, chemCPA; four test splits.
  - iNeuron / organoid morphogen screens: vs baselines and established methods; virtual protocol
    screen proposes untested regimens.
- Metrics: energy distance / Wasserstein / MMD / Sinkhorn divergence in latent space, R^2 of mean
  normalized expression, LFC recovery, cell-type proportion accuracy.
- Reported: state-of-the-art on established gene-knockout and drug benchmarks; the first to model
  whole-embryo and organoid-scale perturbation.

## Claims (authors)
- The authors report CellFlow is a single flexible framework spanning cytokines, drugs, gene
  knockouts, and morphogens, and combinations across those classes.
- They report state-of-the-art on established perturbation benchmarks (sciPlex3, combosciplex,
  gene-knockout tasks) against chemCPA, GEARS, biolord.
- They claim it uniquely scales to whole-organism (zebrafish embryo) and organoid-development
  phenotype prediction.
- They claim OT pairing plus flow matching captures heterogeneous perturbed distributions better
  than mean-response models, and that the virtual organoid protocol screen surfaces novel
  effective treatment regimens.

## Limitations
- Author-stated: the ESM-2 representation of a cytokine was not sufficient for donor-specific
  response prediction on its own (needed a few observed donors); a systematic evaluation of which
  perturbation embeddings are best is left to future work; learning is largely per-dataset.
- Reviewer: the learning rate, compute (GPU type/count/hours), and per-dataset hyperparameters are
  not in the main text, so runs are not reproducible from the article body. Evaluation is in
  latent space (PCA/VAE), so distributional metrics depend on the chosen embedding, and the paper
  itself notes PCA vs VAE trade off reconstruction against distributional fidelity. Baselines
  differ per task (chemCPA for drugs, GEARS/biolord for combinations), and no single strong
  perturbation FM (e.g. scGPT, State, TxPert) is run across all tasks. Not yet peer reviewed.

## Take / open questions
CellFlow and State both frame perturbation as a distribution-to-distribution transport with OT,
but CellFlow makes the generative flow explicit and pushes hardest on breadth (organism and
organoid scale, mixed perturbation classes) rather than leaderboard margins. Open question: how
much of the performance depends on the latent-space choice (PCA vs scVI-VAE), since evaluation
and generation both happen there. The virtual protocol screen is the most novel application and
the clearest test of whether generated phenotypes for untested regimens hold up experimentally.

## relates_to_prior
- target: chemCPA; axis: generalization / drug response; shown - ran as a baseline on sciPlex3
  and combosciplex, reports outperforming it.
- target: GEARS; axis: combinatorial generalization; shown - ran as a baseline on combosciplex.
- target: biolord; axis: generalization; shown - ran as a baseline on combosciplex.
- target: CellOT / OT-based perturbation maps; axis: generalization / scalability; asserted +
  extended - builds on OT pairing but replaces the direct map with a flow-matching vector field
  and adds unbalanced OT to model non-conservation (apoptosis/proliferation).
- target: mean/identity baselines; axis: heterogeneity modeling; shown - beats them once a few
  contexts are observed, argues distributional modeling is the reason.
