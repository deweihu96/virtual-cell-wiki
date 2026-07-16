---
model: scFoundation
paper_title: "Large-scale foundation model on single-cell transcriptomics"
authors: Hao, Gong, Zeng et al. (Tsinghua / BioMap)
year: 2024
venue: Nature Methods
task: embedding
modality: scRNA
organism: human
links:
  paper: https://doi.org/10.1038/s41592-024-02305-7
  code: https://github.com/biomap-research/scFoundation
  weights: https://github.com/biomap-research/scFoundation
---

# scFoundation

## TL;DR
A 100M-parameter transformer foundation model (backbone: xTrimoGene, also called
xTrimoscFoundationα) pretrained on over 50M human single cells across all ~19,264
protein-coding genes, using a read-depth-aware masked regression objective. Cell and
gene context embeddings are then fed, often without fine-tuning, into downstream models
for read-depth enhancement, drug response prediction, perturbation prediction, cell type
annotation, and gene module inference.

## What it is
Asymmetric encoder-decoder transformer, masked-autoencoder style. Input is a cell's full
gene expression vector (19,264 genes) plus two total-count indicator tokens. The encoder
runs vanilla self-attention over only the non-zero, non-masked genes (~10% of the vector);
the decoder runs a Performer (kernel-approximated attention) over the full gene length to
reconstruct expression at masked positions. Outputs: pooled encoder embedding = cell
embedding; per-gene decoder outputs = cell-specific gene context embeddings.

## Training data
- Over 50 million human scRNA-seq profiles collected from all major public resources
  (GEO, Single Cell Portal, HCA, hECA, DISCO, EMBL-EBI, and others), spanning 100+ tissue
  types across normal, disease, and tumor states.
- Aligned to a fixed list of 19,264 human protein-coding + common mitochondrial genes
  (HGNC). Missing symbols padded with zeros.
- QC: cells with >200 expressed genes retained. Validation set = 100,000 randomly sampled
  cells, held constant across model-size comparisons.
- Organism: human only. Modality: scRNA-seq (raw counts; normalized datasets converted
  back to pseudo-counts where possible).

## Core mechanism
Two distinguishing choices. First, continuous value embedding: each scalar expression value
is mapped to a learnable embedding as a weighted sum over a set of basis embeddings (weights
learned from the scalar), with no discretization/binning, unlike scGPT/Geneformer. Second,
read-depth-aware (RDA) pretraining: the model is given a downsampled ("source", total count
S) version of a cell and must predict the raw ("target", total count T) expression at masked
positions, with T and S passed in as explicit indicator tokens. Setting T > S at inference
turns the model into a read-depth enhancer. The asymmetric design (heavy vanilla-attention
encoder on non-zero genes, light Performer decoder on all genes) is what makes training over
the full ~19k-gene vector tractable given scRNA-seq sparsity.

## Implementation (paper)
- Input representation: full 19,264-gene expression vector; continuous value embedding (no
  binning) summed with a learnable gene-name embedding; two extra total-count indicator
  tokens T (target/raw) and S (source/input)
- Vocabulary: 19,264 human protein-coding + mitochondrial genes; T and S indicator tokens
- Architecture: xTrimoGene asymmetric encoder-decoder, ~100M parameters. Encoder = vanilla
  transformer blocks over non-zero/non-masked genes only (~10% of the vector), larger param
  share; decoder = Performer (kernelized attention) over the full 19,266-length input
  (all genes + T,S), smaller param share; shared MLP head projects decoder outputs to scalar
  expression predictions. Gene embedding dim 512. Layer counts, head counts, and per-module
  param split not stated in the main text (deferred to Supplementary Table 7)
- Training objective: RDA masked regression; masking ratio 30% applied to both zero and
  non-zero genes; regression (MSE) loss computed only at masked positions between predicted
  and raw expression
- Optimizer and schedule: not stated in the main text (Supplementary Table 7)
- Preprocessing: raw counts; normalize + log-transform raw and input vectors; total count
  recorded as T/S indicators; hierarchical Bayesian downsampling generates the low-read-depth
  input variant
- Scaling: three sizes trained (3M, 10M, 100M params); validation loss follows a power-law
  (scaling-law) decline with parameters and FLOPs; 100M is the released model
- Fine-tuning / adaptation: mostly frozen. Read-depth enhancement and drug-response tasks use
  non-fine-tuned embeddings; cell type annotation fine-tunes only a single encoder layer plus
  a 2-layer MLP head; perturbation uses frozen gene context embeddings as GEARS graph nodes
- Inference: cell embedding = pooled encoder output; to enhance read depth, set T to a high
  target (e.g. 10,000) and S to the cell's own total count; gene context embeddings taken
  from decoder (last MLP dropped)
- Compute: not stated in the main text (Supplementary Table 7)
- Reproducibility: model code, weights, online-API code, and embedding-inference demo
  released on GitHub / Zenodo; full pretraining configs and compute deferred to supplement

## Implementation (code)
Cloned github.com/biomap-research/scFoundation (~9.2k Python LOC). Key finding: the repo
releases inference, embedding-extraction, and downstream fine-tuning code, but **no pretraining
script**, and the encoder/decoder layer counts, hidden dim, and head counts are **not hardcoded
anywhere** in the repo. They are read at runtime from the released checkpoint's config
(`model/load.py:76` `convertconfig` pulls `ckpt['config']['model_config'][model_type]`), so the
paper's `not stated` architecture numbers remain unavailable without the weight file. What the
code does fix:
- Fixed 19,264-gene vocabulary via `OS_scRNA_gene_index.19264.tsv` (`model/load.py:174`,
  `model/get_embedding.py`).
- Architecture (`model/pretrainmodels/mae_autobin.py`): MAE-style autobin encoder-decoder.
  `AutoDiscretizationEmbedding2` turns each continuous scalar into an embedding via a softmax
  over learnable bins ("autobin", not fixed discretization); learned positional embedding
  `nn.Embedding(max_seq_len+1)`; `decoder_embed = Linear(embed_dim -> decoder_embed_dim)`,
  LayerNorm, Performer decoder (`performer.py`), final `to_final = Linear(decoder_embed_dim -> 1)`
  scalar head.
- GEARS input mode `A` (`model/README.md`): normalized+log1p expression with total count
  appended, shape N x 19,265, used for the perturbation task.
- Fine-tuning is largely frozen: README example unfreezes only the last ~2 encoder blocks
  (`transformer_encoder[-2]`).
- Downstream GEARS variant defaults (`GEARS/train.py`): epochs 20, batch_size 32,
  test_batch_size 128, hidden_size 64, train_gene_set_size 0.75, gradient accumulation 1. Note
  the main text described the GEARS *baseline* run as 15 epochs / batch 30, so the released
  scFoundation-GEARS defaults differ from that baseline setting.

Net: the paper-vs-code delta confirms the pretraining run is not reproducible from the public
repo; only the architecture *skeleton* and downstream recipes are recoverable.

## Evaluation
Used as an embedding/representation provider plugged into task-specific downstream models,
not fine-tuned end-to-end. Tasks and comparisons:
- Read-depth enhancement / imputation: vs MAGIC, SAVER, scImpute, scVI on downsampled
  pancreatic islet data; scFoundation best once T/S fold > ~1, plateaus above 3.5x. Weaker
  than SAVER at fold = 1 (unchanged read depth).
- Model-size scaling: validation MSE vs scBERT-, Geneformer-, scGPT-equivalent
  architecture sizes and scVI; 100M scFoundation lowest loss.
- Bulk drug response (IC50): scFoundation embeddings swapped into DeepCDR, beat the raw
  gene-expression MLP on most drugs/cancer types; leave-one-drug-out blind test improved.
- Single-cell drug response classification: embeddings into SCAD, higher AUC on all four
  hard drugs (sorafenib, NVP-TAE684, PLX4720, etoposide).
- Perturbation prediction: gene context embeddings as GEARS co-expression graph nodes;
  lower top-20-DE MSE than vanilla GEARS and CPA on Norman/Dixit/Adamson, including the
  0/2-unseen two-gene case, plus better genetic-interaction (synergy/suppressor) recovery.
- Cell type annotation: Zheng68K and Segerstolpe; fine-tuning one encoder layer + MLP head.

## Claims (authors)
- The authors report scFoundation is the largest single-cell transcriptomics foundation
  model at publication in parameters (100M), gene coverage (~20k), and data (50M+ cells).
- They report a scaling law: validation loss declines as a power law in parameters and FLOPs.
- They claim the continuous (non-discretized) value embedding outperforms binning-based
  schemes (ablation, Supplementary Fig. 14).
- They claim the RDA task and asymmetric architecture are each beneficial (ablations,
  Supplementary Notes/Tables).
- They report that frozen or lightly-adapted scFoundation embeddings improve state-of-the-art
  downstream models across six task families, i.e. the value is as a plug-in representation
  without task-specific pretraining.

## Limitations
- Author-stated: pretraining data, though near-complete public human scRNA-seq at curation
  time, may still be insufficient; the model covers only protein-coding + mito genes;
  planned future work is larger models.
- Reviewer: the entire training recipe that matters for reproducibility (layer/head counts,
  optimizer, learning rate, warmup, batch size, epochs, GPU count, GPU-hours) is pushed to
  supplementary tables and not in the main text, so the run is not reproducible from the
  article body alone. Nearly all downstream wins are reported as embeddings-into-someone-
  else's-model comparisons rather than end-to-end, which conflates representation quality
  with the downstream model; the paper does not isolate how much of each gain is scFoundation
  vs the host model. Perturbation and annotation baselines are the host methods' own defaults
  (GEARS, CPA, SCAD, DeepCDR), so no stronger contemporary foundation-model baseline is run
  head-to-head on those tasks.

## Take / open questions
scFoundation is one of the reference "big frozen embedding" single-cell FMs, and later
benchmarks (see scPerturBench) run it directly: notably scFoundation ran out of memory on
the two largest Replogle datasets there, so the full-gene-vector design has a real
scalability cost at evaluation time. The continuous-value-embedding vs binning claim is the
most transferable idea and is worth checking against scGPT's binning under matched settings.

## relates_to_prior
- target: scGPT / Geneformer / scBERT; axis: scale and value encoding; asserted + partial
  shown - claims larger scale and that continuous value embedding beats their discretization
  (ablation), and plots their architecture-equivalent sizes on the scaling curve, but does
  not run them as full downstream baselines.
- target: GEARS; axis: perturbation generalization; shown - plugs scFoundation gene context
  embeddings into GEARS as graph nodes and reports lower top-20-DE MSE than vanilla GEARS.
- target: CPA; axis: perturbation generalization; shown - reports lower MSE than CPA on the
  two-gene perturbation task.
- target: scVI / SAVER / MAGIC / scImpute; axis: imputation / read-depth enhancement; shown -
  ran as baselines, scFoundation wins once T/S fold > 1.
