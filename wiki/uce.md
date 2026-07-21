---
model: UCE
paper_title: "Universal cell embedding provides a foundation model for cell biology"
authors: Rosen, Roohani et al. (Quake and Leskovec labs, Stanford / CZ Biohub)
year: 2026
venue: Nature
task: embedding
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.1038/s41586-026-10689-z
  code: https://github.com/snap-stanford/uce
  weights: https://figshare.com/articles/dataset/Universal_Cell_Embedding_Model_Files/24320806
---

# UCE

## TL;DR
A 650M-parameter transformer that maps any scRNA-seq cell into a fixed 1,280-dimensional
space with no fine-tuning, no gene selection and no cell type labels, by tokenizing genes
through ESM2 protein embeddings so the model works on any protein-coding gene from any
species, including species absent from training.

## What it is
An encoder-only transformer over a "cell sentence". Input is a raw count vector plus a
lookup of ESM2 protein embeddings for that species' genes; genes are sampled with
replacement in proportion to log-expression, grouped by chromosome and sorted by genomic
position. Output is the final-layer CLS embedding, L2-normalized, used directly as the
cell representation. No decoder to expression space at inference time.

## Training data
The Integrated Mega-scale Atlas (IMA): 36 million cells across more than 300 datasets,
8 species, ~50 tissues and over 1,000 uniquely named cell types.

- 33.9M cells / 285 datasets from CZI CellXGene Census v.2023-07-10 (human and mouse),
  deduplicated by keeping primary cells only.
- 2.3M cells / 28 datasets covering human, mouse, zebrafish, rhesus macaque, crab-eating
  macaque, mouse lemur, western clawed frog and pig.
- No HVG selection at any point. Full protein-coding gene set retained.
- The authors state the corpus is biased toward mammals (especially human and mouse) and
  toward certain tissues such as brain. Per-species and per-tissue cell counts are deferred
  to Supplementary Table 16 and not in the main text.

## Core mechanism
Gene identity is never learned as a per-dataset token. Each gene is replaced by the ESM2
embedding of the protein it encodes (averaged over all proteins coded by that gene), so
the model's input alphabet is protein sequence space rather than a fixed gene vocabulary.
This is what makes the model genome agnostic: a new species needs only its protein-coding
amino acid sequences, with no orthology mapping, no homologue pairing and no shared-gene
intersection. That property is the whole basis for the zero-shot cross-species claim, and
it is the axis on which UCE differs from Geneformer and scGPT, both of which are bound to
a learned gene vocabulary.

The second choice is the "bag of RNA" abstraction: a cell is a weighted multi-set sample of
its expressed genes, not an ordered list of all measured genes. Combined with a binary
"was this gene expressed" objective, this deliberately discards fine-grained quantitative
expression, which the authors acknowledge as a resolution cost.

## Implementation (paper)
- Input representation: multi-set of 1,024 non-unique genes sampled *with replacement* from
  the expressed set G+, with P(g) proportional to log(x_g + 1) normalized over G+. Genes
  grouped by chromosome between chromosome-specific start/end tokens (unique per chromosome
  *and* species), sorted by genomic position within a chromosome; chromosome order is
  randomized. CLS token prepended. The final sequence is the "cell sentence".
- Vocabulary: no gene vocabulary. Genes enter only as 5,120-dim ESM2-15B protein embeddings,
  compressed by a single-layer MLP before the transformer. Special tokens for CLS, padding
  and per-species chromosome boundaries.
- Architecture: 33 transformer layers, over 650M parameters, embedding dim d_emb = 1,280.
  Head count, feedforward width and dropout are not stated in the main text (deferred to
  Supplementary Table 2); see Implementation (code) for the released values.
- Context length: 1,024 sampled genes. The authors note this cap is forced by quadratic
  attention cost and that many cells express more than 1,024 genes.
- Training objective: binary cross-entropy on masked expression *presence*, not value.
  20% of expressed genes are masked out (r_mask = 0.20) and excluded from the input sample.
  The CLS embedding is concatenated with each candidate gene's compressed protein embedding
  and passed to an MLP that predicts expressed vs not. Loss is balanced: N_loss/2 genes drawn
  from the masked-expressed set and N_loss/2 from the unexpressed set. N_loss is not stated
  in the main text.
- Optimizer and schedule: not stated in the main text (deferred to Supplementary Table 2).
  Learning rate, batch size, warmup, and number of steps or epochs are all absent.
- Preprocessing: CellXGene datasets filtered at min 200 genes per cell and min 10 cells per
  gene. No normalization beyond the log-weighting inside the sampler; no HVG selection.
  Preprocessing for the 28 non-CxG datasets is explicitly *not uniform* (Supplementary Note 1).
- Fine-tuning: none, by design. The authors state UCE is meant to be used as-is so that
  embeddings stay universally shareable. Downstream use is a logistic classifier trained on
  frozen embeddings.
- Inference: forward pass, take final-layer CLS after an MLP decoder head, L2-normalize.
  Embeddings are sampling-dependent: different random seeds give slightly different vectors
  for the same cell (Supplementary Table 11); a fixed seed reproduces exactly.
- Compute: 40 days on 24 A100 80GB GPUs.
- Reproducibility: weights and inference code released. No training script is in the public
  repo (see below). Training hyperparameters exist only in supplementary tables.

## Implementation (code)
Repo `snap-stanford/uce`, ~800 LOC of Python across 5 top-level files plus `data_proc/`.
Small enough to read directly.

- Architecture concretes (`evaluate.py:187-196`): `d_model` (emsize) = 1,280 hard-coded,
  `nhead` = 20 hard-coded, `dropout` = 0.05. `d_hid` (feedforward width) and `token_dim`
  both default to 5,120; `nlayers` defaults to **4**, not 33. The 33-layer paper model is a
  separate download and requires `--nlayers 33` explicitly (`README.md:33`). Embeddings from
  the 4-layer and 33-layer models are explicitly not compatible.
- Model definition (`model.py`): standard PyTorch `TransformerEncoder` with sinusoidal
  positional encoding, max_len 1,536. Encoder is Linear(5120 -> 1280) + GELU + LayerNorm.
  The cell-embedding decoder is a 4-block MLP (1280 -> 1024 -> 1280 -> 1280 -> 1280) with
  LayerNorm/GELU/Dropout blocks. The binary decoder takes `[cell_emb || gene_emb]` at width
  `output_dim + 1280` and narrows 2048 -> 512 -> 128 -> 1. CLS is taken as `gene_output[0]`
  then `F.normalize`, confirming embeddings are unit-norm.
- Protein embedding table (`evaluate.py:200`): 145,469 x 5,120, i.e. ~145k gene tokens
  across all supported species. Chromosome token offset 143,574.
- Sampler (`eval_data.py:59,126-127`): weights are `torch.log1p(counts)` normalized, then
  `np.random.choice(..., size=1024, replace=True)`. Matches equation 3 in the paper.
- Padding: sequences padded to 1,536 (`--pad_length 1536`), so the 1,024 sampled genes plus
  chromosome and CLS tokens fit within that budget.
- Inference batch size: 25 per 80GB GPU for the 33-layer model, 100 for the 4-layer
  (`README.md:24`). This is inference only.
- Preprocessing in code (`data_proc/download_proc_czi_cxg.py:185-186`): `filter_cells(min_genes=200)`,
  `filter_genes(min_cells=10)`, matching the paper. But `data_proc/data_utils.py:207-208,215-216`
  uses `min_genes=25` for other paths, so the non-CxG filtering really is looser than the
  headline numbers.
- Normalization delta: `data_proc/data_utils.py:55` computes
  `log1p(expression / rowsum * 1000)`, a CP1K target rather than the CP10K convention used
  by most single-cell pipelines. Not mentioned in the paper text.
- **No training code.** The repo contains `evaluate.py`, `eval_data.py`,
  `eval_single_anndata.py`, `model.py`, `utils.py` and preprocessing only. There is no
  optimizer, no LR schedule and no training loop anywhere in the release, so the
  `not stated` optimizer fields above cannot be recovered from the code either.

## Evaluation
Zero-shot embedding quality, main benchmark: Tabula Sapiens v.2 (581,430 cells, 27 tissues,
167 batches, 162 cell types), unpublished at training time. Scored with scIB metrics
(bio-conservation, batch correction, overall).

- Zero-shot baselines: Geneformer (GF-12L-30M-i2048), scGPT (whole-human), tGPT, PCA.
- Fine-tuned baselines: scVI, scArches. These use cell type labels and dataset-specific
  training; UCE does not.
- Reported: UCE beats next-best Geneformer by 13.9% overall, 16.2% bio-conservation,
  10.1% batch correction. UCE reported as slightly better than scVI and scArches.
- Ovary sub-analysis (droplet vs plate, 45,757 vs 3,610 cells): UCE batch correction close
  to scVI/scArches, bio-conservation higher.

Cross-species zero-shot: green monkey lymph node/lung, naked mole rat spleen, chick retina,
chick heart. None of these species are in training; no bird species at all. Metric is
nearest cell-type centroid match against the IMA.
- Green monkey: 13/17 centroids match at top-1, 17/17 within top-3.
- Naked mole rat: 17/24 at nearest cross-species centroid.
- Chicken heart: 12/15 within nearest two centroids.
- Baselines: SATURN (supervised, label-aware) and SAMap (fine-tuned). UCE wins on 3 of 4
  datasets for label transfer despite being unsupervised and zero-shot.

Structural evaluation: embedding distance increases monotonically with Cell Ontology tree
distance up to 5 hops, then plateaus. Germ-layer classifier on held-out cell types >80%
accuracy. Cell-type centroid alignment between TSv2 and IMA is 41% exact at top-3, which the
authors report as 65% more accurate than the same procedure in log-normalized expression
space over 5,704 shared genes (92% better at top-1).

Application: a logistic classifier for kidney Norn cells, trained on UCE embeddings of mouse
renal cells, applied across a 2.97M-cell representative IMA sample to nominate Norn-like
cells in gonad, heart and lung, validated by marker-gene differential expression.

## Claims (authors)
- The authors report that UCE produces usable embeddings for entirely new datasets with no
  labelling, training or fine-tuning, and that this zero-shot performance matches or exceeds
  methods that retrain per dataset (scVI, scArches).
- The paper claims genome-agnostic tokenization via ESM2 lets UCE embed species never seen
  in training, including a bird, without orthology mapping.
- The authors describe the resulting organization as emergent, arguing that clustering by
  cell type, cross-tissue alignment of macrophages, germ-layer colocalization and agreement
  with the Cell Ontology all arise without any label supervision.
- The paper claims ESM2-15B protein embeddings outperform other protein language models and
  outperform randomly initialized gene embeddings for every species except human
  (Supplementary Fig. 8), i.e. the benefit is concentrated in under-represented species.
- The authors assert that genomic-location ordering has a positive effect on performance
  while simultaneously noting transformers do not require input ordering and that future
  models may drop it.
- The paper positions UCE as a step toward a virtual cell.

## Limitations
- Author-stated: benchmarks measure recovery of expert-annotated cell types, which are
  limited by label granularity and subjectivity, and may not capture perturbation response
  or cross-modality integration. The model is a black box with no interpretability tooling.
  Training data are biased toward mammals and toward tissues such as brain. Model size and
  hyperparameters were set heuristically and are likely suboptimal. UCE does not replace
  batch effect correction for known experimental confounders. Expression-weighted gene
  sampling discards fine-grained quantitative variation. Fine-grained cell-type transfer
  across large evolutionary distances (e.g. Drosophila) remains unreliable, partly because
  ground-truth correspondence between distant cell-type ontologies is itself ambiguous.
- Reviewer: the training run is not reproducible. Optimizer, learning rate, batch size and
  step count are absent from the main text, and the released repo contains no training code
  at all, so neither source closes the gap. The 40-GPU-day compute figure is therefore not
  actionable.
- Reviewer: the headline scIB comparison is against zero-shot Geneformer and scGPT, both of
  which are known to underperform when used without fine-tuning. A simple PCA baseline is
  described in Methods but its scIB numbers are not surfaced in the main text, and the
  centroid-alignment analysis is the only place raw expression space is used as a floor.
  For an embedding paper, PCA and scVI-with-HVGs are the floors that matter.
- Reviewer: embeddings are stochastic. A different seed changes the sampled 1,024 genes and
  therefore the vector. The paper reports this in a supplementary table but does not quantify
  how much downstream metrics move under reseeding, which matters for a representation
  advertised as universally shareable.
- Reviewer: the released default is a 4-layer model whose embeddings are explicitly
  incompatible with the 33-layer model in the paper. Anyone running the repo as documented
  reproduces neither the numbers nor the embedding space.
- Reviewer: preprocessing for the 28 non-CxG datasets is explicitly non-uniform, and the code
  shows a min_genes threshold of 25 on those paths versus 200 for CxG. Those datasets carry
  most of the non-human, non-mouse signal, i.e. exactly the species where the cross-species
  claim lives.
- Reviewer: the Norn cell analysis is validated by marker-gene differential expression on the
  model's own predicted cells, with the canonical marker (Epo) explicitly excused as
  undetectable. No orthogonal validation.

## Take / open questions
The genome-agnostic tokenization is the durable idea and it is genuinely separable from the
rest: the ablation showing protein embeddings beat random init on every species *except*
human says the mechanism does what it claims, and says the benefit is a data-scarcity fix,
not a general one. On human data the protein prior appears to buy little.

Open: does the reported zero-shot advantage over scVI survive on a modern scIB run with
HVG-tuned baselines, and how much of the cross-species result is the protein embeddings
versus simply the 36M-cell corpus? The second is answerable with the released weights,
the first only partly, since no training code means the ablations cannot be rerun.

Worth noting for this wiki's purposes: UCE is an embedding model, not a perturbation
predictor. The authors say so directly in their limitations. Its relevance to cross-context
prediction is as a frozen context encoder, which is the same role State's SE plays.

## relates_to_prior
- target: Geneformer (Theodoris 2023); axis: generalization / zero-shot transfer; shown -
  run as a zero-shot baseline on Tabula Sapiens v.2, UCE reports +13.9% overall scIB.
- target: scGPT (Cui 2024); axis: input representation and generalization; shown - run as a
  zero-shot baseline. Also asserted: the paper argues modelling expression "as text in the
  form of a sequence of genes" is inefficient and rests on inaccurate biological assumptions,
  a direct criticism of the Geneformer/scGPT tokenization family, not tested by ablation
  against their tokenizers.
- target: tGPT (autoregressive gene-list model); axis: input representation; described as a
  baseline in Methods but not surfaced in the main-text comparison.
- target: scVI (Lopez 2018) and scArches (Lotfollahi 2022); axis: whether per-dataset
  retraining is necessary; shown - run as fine-tuned baselines, UCE reported slightly better
  despite using no labels. This is the paper's strongest comparison because the baselines are
  advantaged.
- target: SATURN (Rosen 2024, same group) and SAMap (Tarashansky 2021); axis: cross-species
  alignment without orthology; shown - run as supervised/fine-tuned baselines on 4
  cross-species datasets, UCE wins 3 of 4.
- target: orthology-based cross-species integration generally; axis: species-specific
  constraints; asserted - the protein-embedding design is presented as removing the need for
  homologue pairing, with the novel-species results as indirect evidence.
- target: PCA on log-normalized expression; axis: whether a learned embedding is needed at
  all; partially shown - used as the floor for the centroid-alignment analysis (UCE 65%
  better at top-3) but not reported in the main scIB table.
