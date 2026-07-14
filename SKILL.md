---
name: model-wiki-page
description: Extract one machine-learning model or method paper (especially single-cell and virtual-cell foundation models) into a single standardized wiki page. Use this whenever the user wants to add a paper or model to their model wiki, ingest a paper into the knowledge base, turn a method paper into a structured reference page, or "make a page for this model." Trigger even when the user just drops a PDF or arxiv link and says something like "add this to the wiki" or "write up this model." One page per paper. Do not invent cross-links between pages yet.
---

# Model Wiki Page

Turn one paper into one page. The page has three jobs: record what the paper *is* (identity), record what it *does* in reproducible detail (substance), and separate what the authors *claim* from what a reader concludes (assessment). The value of the wiki lives in that last separation. If the page blurs the authors' claims into fact, it becomes a transcription of the abstract and tells you nothing you could not get from the paper's own marketing.

Fill from a single paper in a single pass. Do not pull in outside sources to "correct" the paper on this pass; record what this paper says and flag gaps.

## Where pages are written

Pages are data, not part of the skill. Write every page into the wiki content directory at the repository root, `wiki/`. That directory is the shared, versioned knowledge base: it holds the seed pages shipped with the project and every page users add. Both this skill and Claude Code reach it because it sits under the repository root, which is the Claude Code working directory.

- Resolve the repository root as the directory that contains `CLAUDE.md` and the `.claude/` folder. Write to `<repo-root>/wiki/`.
- If `wiki/` does not exist, create it.
- Filename is the `model` key, lowercased, with spaces and slashes replaced by hyphens. `scGPT` becomes `wiki/scgpt.md`.
- If a page for that model already exists, update it in place. Do not create a second file, and preserve human edits outside the fields you are refreshing.
- Never write pages inside `.claude/skills/`. That folder is the skill itself and may be read-only when installed personally.

## Calibrate before writing

Before generating a page, read the reference example at `${CLAUDE_SKILL_DIR}/references/EXAMPLE.md` and match its structure and grain. `${CLAUDE_SKILL_DIR}` resolves to this skill's own directory whether it is installed at project, personal, or plugin level, so the example is always reachable regardless of how the user installed the skill.

## Core rules

**Record absence as data.** Whenever a field is not reported in the paper, write `not stated`. Do not guess, infer, or leave it blank. A blank looks like the field was never checked; `not stated` says the paper omitted it. For reproducibility fields (implementation, compute, preprocessing) this absence is itself a finding, so make it visible.

**Attribute claims to the authors.** In the Claims section, everything is the authors' assertion, not established fact. Write "the authors report," "the paper claims," not bare statements. Numbers copied from the paper's own tables are the authors' measurements, not neutral truth.

**Do not manufacture criticism.** Extract the substance faithfully. In the reviewer-side fields, only write something you can ground in what the page already contains (an unstated baseline, a missing ablation, an evaluation that only compares against weak baselines). If you cannot ground a critique, leave the scaffold prompt in place for the human. An empty scaffold is better than an invented opinion.

**No em-dashes. Concise sentences. Direct over impressive.**

## Controlled vocabularies

Keep these fields comparable across pages by drawing from fixed vocabularies. Add a new value only when nothing fits, and prefer the closest existing term.

- `task`: embedding, cell-type-annotation, perturbation-prediction, GRN-inference, batch-integration, generation, multimodal-integration, other
- `modality`: scRNA, snRNA, multiome, ATAC, spatial, protein, bulk, multi
- `organism`: human, mouse, multi, other

Comparability only exists within a task. A perturbation predictor and an annotation model cannot be compared on "performance," so the `task` field is what stops the wiki from inviting that comparison later.

## Page template

Use this exact structure. Frontmatter is the machine-readable identity tier. The body carries substance and assessment.

```markdown
---
model:            # short stable key, distinct from paper title (e.g. scGPT, Geneformer, State)
paper_title:
authors:          # first author et al., or the group
year:
venue:            # journal / conference / preprint server
task:             # from controlled vocab
modality:         # from controlled vocab
organism:         # from controlled vocab
links:
  paper:
  code:           # or: not stated
  weights:        # or: not stated
---

# {model}

## TL;DR
One or two sentences: what it is and what it is for. No hype.

## What it is
Architecture family, what goes in (input representation), what comes out (output).
Keep to a few lines. The distinguishing idea goes in Core mechanism, not here.

## Training data
The corpus it was trained on and its scale: number of cells, number of datasets,
gene coverage, modalities, organism. For cell foundation models the scale claim is
often the entire pitch, so record the numbers exactly and note what is `not stated`.

## Core mechanism
The one idea that separates this model from its neighbors. Enough detail to tell it
apart from the prior work it builds on. This is conceptual, not the hyperparameters.

## Implementation details
Reproducibility-grade specifics. Use `not stated` liberally here; the omissions are
the point. Cover, where reported:
- Input representation / tokenization: how a cell becomes model input (rank-based
  ordering, value binning, HVG selection, continuous vs discrete, special tokens)
- Vocabulary: gene vocab size, special tokens
- Architecture: transformer type, layers, hidden dim, heads, parameter count,
  context length (genes per cell)
- Training objective: masked expression prediction, MLM, contrastive, autoregressive,
  generative; masking ratio
- Optimizer and schedule: optimizer, learning rate, warmup, batch size, steps/epochs
- Preprocessing: normalization (library size, log1p), HVG count, QC filters
- Fine-tuning / adaptation: frozen embeddings + linear probe vs full fine-tune, and
  per-downstream-task heads
- Inference: how an embedding or prediction is produced at use time
- Compute: hardware, GPU count, GPU-hours or wall-clock
- Reproducibility: is training code released, are configs released

## Evaluation
Benchmarks, metrics, and the baselines it was compared against. Record the baselines
explicitly, not just the scores. A model that only benchmarks against weak baselines
is telling you something. Note anything that looks like an unfair or missing comparison.

## Claims (authors)
The paper's headline claims, attributed to the authors. Bullet form. These are
assertions, not facts.

## Limitations
- Author-stated: limitations the paper itself acknowledges.
- Reviewer: gaps you can ground in this page (unstated baseline, missing ablation,
  leakage risk, evaluation scope). Leave this scaffold in place if you cannot ground one.

## Take / open questions
Optional. Short. Only grounded observations or genuine open questions. Leave blank
rather than filling with generic praise.

## relates_to_prior
Free text seed for the future critique layer. For each prior work this paper positions
against, note: target (which model/paper), axis (scale, biological fidelity, evaluation
rigor, generalization, batch handling, leakage), and whether the paper *shows* the
prior work fails (ran it as a baseline) or merely *asserts* it. Capture this now even
though pages are not linked yet; backfilling it later means re-reading every paper.
```

## Example: implementation section (fragment)

Shows the intended grain, including honest use of `not stated`.

Input: a masked-language-model cell foundation paper that reports architecture and objective but omits optimizer and compute.

Output:
```markdown
## Implementation details
- Input representation: expression values binned into 51 tokens; genes ordered by
  input; [CLS] token prepended
- Vocabulary: ~60k gene tokens plus special tokens
- Architecture: 12-layer transformer, hidden dim 512, 8 heads, ~50M params,
  context length 1200 genes
- Training objective: masked expression-value prediction, masking ratio not stated
- Optimizer and schedule: not stated
- Preprocessing: library-size normalization then log1p; 1200 HVGs
- Fine-tuning: frozen embeddings + linear probe for annotation; full fine-tune for
  perturbation
- Inference: [CLS] embedding used as the cell representation
- Compute: not stated
- Reproducibility: inference code released; training code and configs not stated
```

The three `not stated` lines are not filler. Together they say the training run is not reproducible from the paper, which is exactly the kind of fact the wiki exists to surface.

## What to skip for now

Pages are standalone. Do not create `[[wikilinks]]`, do not build a graph, do not
cross-reference other pages. The `relates_to_prior` field captures the raw material
for that layer without committing to it. Keep each page fillable and readable on its own.
