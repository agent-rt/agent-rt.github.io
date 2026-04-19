# agent-rt.github.io

Unified documentation site for [agent-rt](https://github.com/agent-rt)
products. Built with Jekyll, served via GitHub Pages at
<https://agent-rt.github.io>.

## Layout

```
.
├── index.md              # org landing
├── synap/                # Synap product docs
│   ├── index.md
│   ├── install.md
│   ├── quickstart.md
│   ├── concepts.md
│   ├── config.md
│   ├── mcp-clients.md
│   ├── troubleshooting.md
│   ├── reference.md
│   └── recipes/
│       ├── index.md
│       ├── personal-memory.md
│       ├── knowledge-base.md
│       ├── task-board.md
│       └── accounting.md
└── _config.yml
```

New products go under their own top-level directory
(`cortex/`, `<other>/`, …) and get a card on `index.md`.

## Local preview

```sh
bundle exec jekyll serve
```

## Contributing

PRs welcome. The site is agent-first — write clear, concise Markdown,
avoid JS-heavy rendering, and prefer explicit tool-call examples over
screenshots.
