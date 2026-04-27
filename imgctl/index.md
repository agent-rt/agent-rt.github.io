---
title: imgctl
---

# imgctl

`imgctl` is the **agent-first image processing CLI**. A single static Rust binary with 16 commands for editing, visual analysis, and diagram generation — built for programmatic consumers, not GUIs. TSV output by default, JSON with `--json`, stable error codes everywhere.

```sh
# Convert + resize + crop
imgctl convert -i in.png -o out.jpg --quality 80
imgctl resize  -i in.png -o out.png --width 400 --fit contain
imgctl crop    -i in.png -o out.png --x 10 --y 20 --w 200 --h 150

# Annotate (text, arrows, boxes, blur)
imgctl annotate text -i in.png -o out.png --x 50 --y 100 --text "Hello"

# Mermaid → PNG (headless Chrome under the hood)
imgctl mermaid -i diagram.mmd -o diagram.png
```

## Why agent-first

Existing image tools (ImageMagick, ffmpeg, …) weren't built for AI agents — complex flags, prose error messages, no structured output. `imgctl` fixes those:

- **Structured output by default** — TSV (compact, agent-optimized), JSON with `--json`. No string-parsing required.
- **Stable error codes** — agents can pattern-match on `error.code` (`UNSUPPORTED_FORMAT`, `IO_ERROR`, `CHROME_TIMEOUT`, …) rather than decoding human prose.
- **Single static binary** — no Node, no Python, no ImageMagick. `cargo install` or `brew install` and you're done.
- **Dual-channel I/O** — when output is `-` (stdout), binary data goes to stdout and TSV/JSON metadata goes to stderr automatically. Agents can pipe images while still capturing structured metadata.
- **Fast cold start** — `--help` returns in < 10ms.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| **`imgctl`** | Image processing & visual diagramming |
| [`skillctl`](/skillctl/) | Curated capability bundles |
| [`memctl`](/memctl/) | Persistent agent memory |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |

`imgctl` is invocation infrastructure — it doesn't manage agent state. It's used *by* agents that need to inspect or transform images during their workflow.

## What's inside

- **Edit** — `convert`, `resize`, `crop`, `compose`, `info`, `metadata`
- **Annotate** — `annotate text`, `annotate arrow`, `annotate box`, `annotate blur`, `annotate composite`
- **Analyze** — `compare`, `diff`, `histogram`, `palette` (via `imageproc`)
- **Diagram** — `mermaid` (headless Chrome rendering)
- **Vision** — pluggable visual diff for screenshot regression

## Get started

1. **[Install](/imgctl/install/)** — Homebrew or `cargo install`.
2. **[Quickstart](/imgctl/quickstart/)** — convert, annotate, render a Mermaid diagram in two minutes.
3. **[Commands](/imgctl/commands/)** — all 16 subcommands with stable I/O contracts.

## Links

- Source: [github.com/agent-rt/imgctl](https://github.com/agent-rt/imgctl)
- Releases: [github.com/agent-rt/imgctl/releases](https://github.com/agent-rt/imgctl/releases)
- Issues: [github.com/agent-rt/imgctl/issues](https://github.com/agent-rt/imgctl/issues)
