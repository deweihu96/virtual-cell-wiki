# Virtual cell wiki

A self-maintaining wiki of ML models and method papers, focused on single-cell and virtual-cell foundation models. One page per model. Ships with a seed set of processed pages, and lets you add your own papers with the bundled Claude Code skill.

## Install

Requires [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). For large PDF files, install tools like [poppler](https://formulae.brew.sh/formula/poppler) 

```bash
git clone https://github.com/deweihu96/virtual-cell-wiki
cd virtual-cell-wiki
claude            # start Claude Code in the repo root
```

The extraction skill is project-scoped (`.claude/skills/virtual-cell-wiki/`) and activates
automatically. There is nothing else to install: the skill and the content directory travel together in this repository.

## Use

- Browse existing pages in `wiki/`.
- Add a paper — any of:
  - drop a PDF into `sources/` and say "add this to the wiki";
  - paste an arxiv link (`https://arxiv.org/abs/...`);
  - paste a bioRxiv link (`https://www.biorxiv.org/content/...`) — Claude fetches the PDF into `sources/` for you.
- If the skill does not trigger on its own, invoke it directly with `/virtual-cell-wiki`.

Run Claude Code from the repository root so `wiki/` and the skill are both in reach.

## Deepening a page from the reference code

Papers routinely omit hyperparameters and layer specifics. When the paper links a code
repo, ask Claude to "pull implementation details from the code". Claude clones the repo
into `code/<model>/` (gitignored), reads it, and appends an `## Implementation (code)`
section next to the existing `## Implementation (paper)` section so the paper-vs-code
delta stays visible. For small repos this is a grep pass; for large repos, Claude uses
the optional [graphify](https://github.com/Graphify-Labs/graphify) Claude Code skill (if installed) and discards the graph afterward.

## Sharing pages back

`wiki/` is versioned. Commit new pages and open a pull request to contribute them, or keep
your fork private. Pages are plain Markdown with YAML frontmatter, so `wiki/` also opens as
an Obsidian vault.

## Just want the extractor?

If you already have your own wiki and only want the skill, copy
`.claude/skills/virtual-cell-wiki/` into your own repo's `.claude/skills/`, or into
`~/.claude/skills/` for personal use across all projects. When installed outside this repo,
point it at your own content directory (the skill writes to a `wiki/` folder at the working
tree root by default).

## Layout

```
model-wiki/
├── CLAUDE.md                         # project context Claude Code reads automatically
├── README.md
├── .claude/
│   └── skills/
│       └── virtual-cell-wiki/
│           ├── SKILL.md              # the extraction schema and rules
│           └── references/
│               └── EXAMPLE.md        # one filled page, used for format calibration
├── wiki/                             # the knowledge base: seed pages + your pages
│   └── README.md
├── sources/                          # raw papers you drop in to process (optional)
│   └── README.md
└── code/                             # gitignored scratch space for cloned reference repos
```

## License

Choose a license before publishing. If you keep this close to the LLM Wiki lineage, note
that project is GPL-3.0.
