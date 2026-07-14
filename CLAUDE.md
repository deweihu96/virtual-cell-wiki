
# virtual-cell-wiki

Knowledge database for virtual cell models that can be easily read by AI models. One page per model.

## Layout

- `wiki/` is the knowledge base. Seed pages ship here and new pages are written here. One
  Markdown file per model, named `<model>.md`, with YAML frontmatter and a Markdown body.
- `sources/` holds raw papers you drop in to process (PDF or notes). Inputs only.
- `.claude/skills/model-wiki-page/` is the extraction skill that turns a paper into a page.

## Processing a paper

Point Claude at a paper (a PDF in `sources/`, an arxiv link, or pasted text) and ask to add
it to the wiki. The `model-wiki-page` skill extracts it into a standardized page and writes
it to `wiki/<model>.md`. Run Claude Code from this repository root so the skill and the
content directory are both in reach.

## Rules for Claude

- Write new model pages to `wiki/`, one file per model. Never put data files inside `.claude/`.
- Follow the `model-wiki-page` skill schema exactly, including the `not stated` convention.
- Do not fabricate hyperparameters, benchmark numbers, or baselines. If a paper omits a
  detail, record `not stated`.
