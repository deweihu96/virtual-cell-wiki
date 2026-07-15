---
model: TranscriptFormer
paper_title: "TranscriptFormer: A generative cell atlas across 1.5 billion years of evolution"
authors: Pearce et al. (Biohub, CZI)
year: 2026
venue: Science
task: embedding
modality: scRNA
organism: multi
links:
  paper: https://doi.org/10.1126/science.aec8514
  code: https://github.com/czi-ai/transcriptformer
  weights: https://virtualcellmodels.cziscience.com/model/transcriptformer
---

# TranscriptFormer

## TL;DR
Generative autoregressive foundation model for single-cell transcriptomics. Genes
are tokenized by their ESM-2 protein-sequence embeddings, so the model is species-
agnostic without needing an orthology mapping. Trained on up to 112M cells across
12 species spanning 1.53B years of evolution. Used as a frozen embedding model
(linear probing) for cell type classification, zero-shot disease detection, and
cross-species annotation transfer, and as a "virtual instrument" for prompt-based
gene-gene interaction and transcription-factor target prediction.

## What it is
A 12-layer transformer encoder over a cell "sentence": each cell is a bag of
transcripts represented as (ESM-2 gene embedding, raw count) pairs plus a single
assay token. Two output heads: a gene-identity head trained with cross-entropy to
predict the next gene, and a count head trained with zero-truncated-Poisson
negative log-likelihood to predict the count for that gene. Cell embedding =
average of transformer outputs; contextualized gene embedding = per-token output
in the context of a specific cell.

## Training data
Three variants share architecture but differ in training corpus:
- TF-Sapiens: 57M human cells only.
- TF-Exemplar: 110M cells from human + 4 model organisms (mouse, zebrafish,
  fruit fly, roundworm).
- TF-Metazoa: 112M cells across 12 species - human, mouse, rabbit, chicken,
  African clawed frog (X. laevis), zebrafish, green sea urchin (L. variegatus),
  roundworm (C. elegans), fruit fly, freshwater sponge (Spongilla lacustris),
  plus outgroups baker's yeast (S. cerevisiae) and malaria parasite (P.
  falciparum). Spans 1.53B years of evolution.

Human/mouse data primarily from CZ CELLxGENE; other species from public
organism-specific atlases (see paper Data availability). Total 1,889 cell types
in TF-Metazoa training. Species-specific sampling weights used to prevent
high-resource species (human, mouse) from dominating (Table S7). All three
variants trained on a fixed total cell budget so parameter count / active
parameters at inference are identical (302M).

## Core mechanism
Two ideas together: (1) gene tokens are ESM-2 protein embeddings rather than a
per-species vocabulary, so distant species share the same input space without an
orthology table (same trick as UCE, but paired with a generative objective); (2)
generative autoregressive training over cell sentences rather than pure masked
prediction or contrastive learning. Attention is expression-aware: raw counts
enter attention as an additive bias so higher-count genes exert more influence.
Causal masking makes the model a joint distribution over (gene identity, count)
pairs, which is what enables prompt-style querying (condition on a set of marker
genes / a transcription factor, ask for the next-gene distribution).

## Implementation (paper)
- Input representation: cell = unordered set of (ESM-2 gene embedding, raw count),
  prepended with an assay token. Genes tokenized by their protein-coding ESM-2
  embedding, so genes across species live in the same embedding space.
- Vocabulary: ESM-2 embedding per Ensembl gene; TF-Sapiens uses human genes only,
  TF-Exemplar / TF-Metazoa share cross-species vocabulary. "Vocabulary" is
  effectively the union of protein-coding genes with ESM-2 embeddings.
- Architecture: 12-layer transformer encoder with expression-aware multi-head
  self-attention and causal masking. Model dimension and layer count identical
  across the three variants (302M active parameters).
- Number of heads / hidden dim / feedforward dim: not stated in the paper text
  (given in Table S2, not shown here).
- Training objective: composite loss = cross-entropy on next-gene prediction +
  negative log-likelihood on zero-truncated Poisson count prediction.
- Optimizer / schedule: linear warmup + cosine decay; specific LR and optimizer
  choice not stated in the paper text.
- Training scale: ~3.5 trillion tokens total, mixed precision, distributed data
  parallel on a "large-scale GPU cluster" (CZI infrastructure). Specific GPU
  count / GPU-hours not stated.
- Preprocessing: minimal filtering to preserve heterogeneity; genes mapped to
  Ensembl stable IDs; raw counts used (not log1p / library-size normalized).
- Fine-tuning: none. All downstream evaluations use linear probing on frozen cell
  embeddings, or reference mapping via k-nearest-neighbors, or prompt-style
  generative queries. No task-specific fine-tuning is performed.
- Baseline: ESM2-CE, an in-house ablation that skips the generative transformer
  and just averages ESM-2 protein embeddings across expressed genes. Used to
  isolate the contribution of the generative training.
- Reproducibility: model code and pretrained weights released; embeddings for
  TF-Sapiens and TF-Exemplar available through the CZ CELLxGENE Census.

## Implementation (code)
Values pulled from the released repo (`src/transcriptformer/data/dataclasses.py`
`ModelConfig`, `conf/inference_config.yaml`).

- Architecture defaults: `num_layers` 12, `num_heads` 16, `model_dim` 2048,
  `embed_dim` 2048, dropout 0.1, activation gelu, no attention or feedforward
  bias, softmax mu link with softcap 10.
- Sequence: `seq_len` 2047 genes per cell + `aux_len` 1 auxiliary token (assay
  identifier), Flex-attention block length 128.
- Gene head hidden dim: 2048.
- Inference: `batch_size` 8, `precision` 16-mixed, counts clipped at 30
  (`clip_counts: 30`), genes filtered to vocabulary, no library-size
  normalization (`normalize_to_scale: 0`), no gene sorting or randomization.
- Gene identifier column: `ensembl_id`; auto-detect between `.raw.X` and `.X`.
- Framework: PyTorch, Flex-attention, Hydra configs.
- Preprocessing pipeline: repo ships `preprocess/` scripts; expects h5ad input.

Training-loop hyperparameters (LR, warmup fraction, weight decay, batch size,
total steps) are not present in the released `conf/` - the repo is inference /
fine-tune-oriented; training configs would need to be reconstructed from the
paper's `~3.5T tokens, warmup + cosine` description.

## Evaluation
Tasks and baselines:

1. Cell-type classification on out-of-distribution species (mouse lemur,
   tropical clawed frog, sea lamprey, stony coral) via linear probing. TF-Metazoa
   average macro F1 0.778, TF-Exemplar 0.770, UCE 0.701. Maintains F1 >0.65 even
   on stony coral (685 Myr from humans) where UCE drops to F1 <=0.5.
2. Cross-species cell type annotation transfer on the Murat spermatogenesis atlas
   (9 vertebrates). TF-Exemplar species-averaged F1 0.480 vs UCE 0.377;
   chicken-to-mammal (310 Myr divergence) TF-Exemplar 0.448 vs UCE 0.251.
   Primate dorsolateral prefrontal cortex transfer: TF-Exemplar 0.744, TF-Metazoa
   0.712, UCE 0.55.
3. LPS response transfer in bone-marrow phagocytes across mouse/rat/rabbit/pig:
   TF-Exemplar F1 0.925, TF-Metazoa 0.880, UCE 0.740, ESM2-CE 0.580.
4. Human cell-type classification on Tabula Sapiens 2.0 (new held-out subset,
   26 tissues). TF-Exemplar 0.910, TF-Metazoa 0.907, TF-Sapiens 0.906, tied with
   UCE 0.906. TranscriptFormer beats other models specifically on hard cell
   types (myeloid leukocytes, T cells, ILCs).
5. Zero-shot disease detection - SARS-CoV-2 infection (Wu 2024 lung dataset):
   TF-Sapiens 0.859 vs UCE 0.805, scGPT 0.798, Geneformer 0.797. Glioblastoma
   tumor vs normal: TranscriptFormer variants and scVI 0.88-0.91, ahead of UCE
   and others.
6. Drug perturbation detection on a 95-compound sample of Tahoe-100M: binary
   DMSO-vs-drug classification via linear probing. TF-Sapiens mean AUC 0.879 vs
   UCE 0.779, scGPT 0.774, scVI 0.709. AUC per drug 0.727-0.998; TAK-901,
   Bortezomib, PH-797804 all >0.99.
7. Generative prompting: pointwise conditional mutual information (PMI) between
   selected transcription factors and all other genes, validated against STRING
   v12. E2F8 87 hits vs 2.0 expected by permutation; FOXM1 105 vs 4.1; MYBL2 54
   vs 1.3; SPIB 9 vs 0.1; UHRF1 59 vs 1.4. Cell-type-conditioned TF probability
   heatmap over 112 human TFs recovers the Tabula Sapiens 2.0 diagonal +
   ubiquitous-band structure and known cell-type-specific TFs (IKZF1/3 in T
   cells, CDX1 in stem cells, SOX30/TCFL5 in male germ cells).

Emergent structure (unsupervised): spermatogenic stages ordered correctly by
cosine similarity in embedding space; cross-species embedding similarity
correlates with evolutionary distance (Spearman -0.705, p=0.004) - ESM2-CE
baseline shows no such correlation (-0.059, p=0.801). CGE variance partitioning:
cell type explains >95% of PC1/PC2 variance, tissue and donor <7%.

## Claims (authors)
- ESM-2 gene tokenization + generative pretraining beats existing single-cell
  foundation models (UCE, scGPT, Geneformer, GeneCompass, scPRINT, AIDO.Cell,
  scVI) on cross-species cell-type classification and zero-shot disease
  detection, with the gap widening at larger evolutionary distances.
- Multispecies pretraining does not hurt within-species performance:
  TF-Metazoa/TF-Exemplar match or slightly beat TF-Sapiens on Tabula Sapiens 2.0.
- Species-specific pretraining still wins for tasks inside its training domain:
  TF-Sapiens beats TF-Metazoa on SARS-CoV-2 detection and on Tahoe-100M drug
  perturbation classification.
- Phylogenetic structure, developmental order, tissue-vs-donor hierarchy emerge
  in embeddings without any supervision on those attributes.
- The generative interface can be prompted to predict TF-target associations that
  match STRING annotations far above chance, positioning the model as a
  "virtual instrument" for hypothesis generation.

## Limitations
- Author-stated: no zero-shot perturbation prediction (Tahoe-100M is used as a
  classification/detection benchmark, not a forward prediction task); batch
  effects still confound cross-species comparisons (human spermatogenesis cells
  cluster apart from other primates due to technical batch, not biology);
  multi-modal extension not attempted; cross-kingdom mapping (yeast to animal
  progenitors) noted as speculative.
- Reviewer: absolute training compute and GPU-hours are not reported, and the
  released repo ships inference configs but no training config, so exact
  reproduction of the pretraining run is not possible from the paper alone.
  TF-Sapiens vs TF-Metazoa differences on within-species tasks are small and
  differences on disease tasks are attributed to species-specific pretraining
  advantage; a matched-cell-count TF-Sapiens vs TF-Sapiens-trained-longer
  ablation would help separate that from raw scale. Perturbation evaluations are
  detection tasks, not the standard perturbation-prediction benchmark (unseen
  perturbation, delta-expression regression); scPerturBench-style comparison is
  absent. STRING is used as ground truth for TF-target validation despite being
  a compilation that itself includes coexpression and text-mined signals, so the
  PMI-vs-STRING enrichment is somewhat circular.

## Take / open questions
The strongest scientific claim - that a pure autoregressive generative objective
on ESM-2 gene tokens beats masked / contrastive alternatives - is inseparable
from the ESM-2 gene-tokenization choice (also used by UCE and scPRINT). The
ESM2-CE ablation shows generative training adds value on top of ESM-2 tokens,
but the paper does not ablate the reverse (same generative objective with a
per-species learned gene vocabulary), so how much of the cross-species gain is
"generative training" vs "shared protein-embedding gene space" is not cleanly
attributed. This matters because the ESM-2 embedding is essentially a
phylogenetic prior baked into the input.

For downstream users the practical question is which variant to pick: paper
recommends TF-Metazoa / TF-Exemplar for evolutionary / cross-species work and
TF-Sapiens for human disease modelling; all three share inference architecture
so switching is cheap.

## relates_to_prior
- target: UCE (Rosen 2023); axis: gene tokenization (both use ESM-2), objective
  (masked vs generative); shown - ran as baseline across all cross-species and
  human tasks, TranscriptFormer wins.
- target: scGPT (Cui 2024), Geneformer (Theodoris 2023), scFoundation (Hao 2024);
  axis: species coverage (human/mouse only), objective (masked-value or ranked
  prediction); shown - ran as baselines on human tasks, TranscriptFormer wins on
  SARS-CoV-2 and Tahoe-100M perturbation detection, ties on Tabula Sapiens 2.0
  classification.
- target: GeneCompass (Yang 2024), scPRINT (Kalfon 2025), Nicheformer
  (Tejada-Lapuerta 2025), AIDO.Cell (Ho 2024), scMulan (Chen 2024), scVI (Lopez
  2018); axis: architectural choice, training corpus; shown - included as
  baselines in relevant subsets.
- target: ESM2-CE (in-house baseline); axis: whether transformer training on
  transcriptomes adds value beyond averaged ESM-2 protein embeddings; shown -
  substantial gap on all downstream tasks (91% relative improvement on
  cross-species spermatogenesis transfer, no phylogenetic-distance correlation in
  ESM2-CE), used to isolate the contribution of generative pretraining.
- target: Tabula Sapiens 2.0 (Quake 2024); axis: TF-target discovery
  methodology; shown - generative prompting recovers the same diagonal +
  ubiquitous-band TF-vs-cell-type structure reported empirically in TS2.0.
- target: STRING (Szklarczyk 2023); axis: TF-target ground truth; adopted -
  STRING treated as gold standard for validation of PMI predictions.
