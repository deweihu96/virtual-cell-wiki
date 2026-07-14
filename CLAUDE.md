
# virtual-cell-wiki

Knowledge database for virtual cell models that can be easily read by AI models. One page per model.

## Layout

- `wiki/` is the knowledge base. Seed pages ship here and new pages are written here. One
  Markdown file per model, named `<model>.md`, with YAML frontmatter and a Markdown body.
- `sources/` holds raw papers you drop in to process (PDF or notes). Inputs only.
- `code/` (gitignored) is scratch space for cloned reference implementations used to enrich pages.
- `.claude/skills/virtual-cell-wiki/` is the extraction skill that turns a paper into a page.

## Processing a paper

Point Claude at a paper (a PDF in `sources/`, an arxiv link, or pasted text) and ask to add
it to the wiki. The `virtual-cell-wiki` skill extracts it into a standardized page and writes
it to `wiki/<model>.md`. Run Claude Code from this repository root so the skill and the
content directory are both in reach.

## Enriching a page with code-derived implementation details

Papers routinely omit hyperparameters, optimizer settings, and layer specifics. When the
paper links a code repo, clone it into `code/<model>/` and pull concrete values back into
the wiki page under a distinct "Implementation (code)" section, keeping the
"Implementation (paper)" section untouched so the paper-vs-code delta stays visible.

Choose the read tool by repo size:
- Small repo (roughly <10k LOC or <20MB): `git clone --depth 1` then grep/Read directly.
- Big repo: use the `graphify` skill to build a queryable knowledge graph, then discard
  `graphify-out/` once the wiki page is updated. One-off per paper; do not keep the graph.

## Rules for Claude

- Write new model pages to `wiki/`, one file per model. Never put data files inside `.claude/`.
- Follow the `virtual-cell-wiki` skill schema exactly, including the `not stated` convention.
- Do not fabricate hyperparameters, benchmark numbers, or baselines. If a paper omits a
  detail, record `not stated`.
