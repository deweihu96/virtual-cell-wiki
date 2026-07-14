Virtual cell wiki

A self-maintaining wiki of ML models and method papers, focused on single-cell and virtual-cell foundation models. One page per model. Ships with a seed set of processed pages, and lets you add your own papers with the bundled Claude Code skill.

## Install

Requires [Claude Code](https://docs.claude.com/en/docs/claude-code/overview).

```bash
git clone <this-repo-url> model-wiki
cd model-wiki
claude            # start Claude Code in the repo root
```

The extraction skill is project-scoped (`.claude/skills/model-wiki-page/`) and activates
automatically. There is nothing else to install: the skill and the content directory travel together in this repository.

## Use

- Browse existing pages in `wiki/`.
- Add a paper: drop the PDF in `sources/` (or have an arxiv link ready) and tell Claude "add this paper to the model wiki". The page is written to `wiki/<model>.md`.
- If the skill does not trigger on its own, invoke it directly with `/model-wiki-page`.

Run Claude Code from the repository root so `wiki/` and the skill are both in reach.

## Sharing pages back

`wiki/` is versioned. Commit new pages and open a pull request to contribute them, or keep
your fork private. Pages are plain Markdown with YAML frontmatter, so `wiki/` also opens as
an Obsidian vault.

## Just want the extractor?

If you already have your own wiki and only want the skill, copy
`.claude/skills/model-wiki-page/` into your own repo's `.claude/skills/`, or into
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
│       └── model-wiki-page/
│           ├── SKILL.md              # the extraction schema and rules
│           └── references/
│               └── EXAMPLE.md        # one filled page, used for format calibration
├── wiki/                             # the knowledge base: seed pages + your pages
│   └── README.md
└── sources/                          # raw papers you drop in to process (optional)
    └── README.md
```

## License

Choose a license before publishing. If you keep this close to the LLM Wiki lineage, note
that project is GPL-3.0.
