---
title: llmctl
---

# llmctl

`llmctl` is a **fast, pipe-friendly CLI for testing OpenAI-compatible and Anthropic LLM endpoints**. One single Zig binary speaks Chat Completions, OpenAI-compatible servers (llama-server, vLLM, Ollama, …), and Anthropic Messages — through a provider abstraction where `format × timing` (text/json/ndjson × stream/batch) are orthogonal.

```sh
llmctl "explain recursion"
echo "summarize" | llmctl < article.txt
llmctl --base-url http://10.0.0.64:8800 --model gemma "hi"
llmctl --output ndjson -m gpt-4o -m claude-sonnet-4-5 "hi" | jq .
llmctl --provider anthropic --model claude-sonnet-4-5 "hi"
llmctl --render markdown "explain monads"           # ANSI-rendered output
llmctl -i                                           # REPL with slash commands
```

## Why one CLI for every chat API

`llmctl` is opinionated for users who **debug, script, and compare LLM endpoints** from the shell:

- **Provider as a triple of pure functions** — `builder` / `stream_decoder` / `batch_decoder`. Adding a new provider is three functions, not a new code path. `local` (llama-server), `openai`, `openai-compat`, and `anthropic` ship built-in.
- **Concurrent multi-model** — repeat `--model` to fan out the same prompt to multiple models in parallel. Output comes back as tagged NDJSON, ready for `jq`.
- **`format × timing` is orthogonal** — `--output text|json|ndjson` and `--buffer/--no-stream` compose freely. Stream NDJSON to a TUI, buffer JSON for scripting, stream text for humans. `--render markdown` post-renders buffered text with ANSI bold/dim/reverse for headings, code, and emphasis.
- **`--extra` passthrough** — any provider-specific knob (`cache_prompt`, `seed`, `repeat_penalty`, …) goes through type-inferred without a CLI flag for it. `--extra-json '{...}'` for nested objects.
- **Smart error bodies** — 4xx/5xx responses are parsed (OpenAI / Anthropic / FastAPI shapes recognised) and surfaced as a clean one-liner like `Invalid API key [invalid_api_key]` instead of a raw JSON dump.
- **Sessions + REPL** — `llmctl -i` gives a slash-command REPL; conversations persist via `--session path.json`, fork via `--save-session`. `/dry-run` toggles request-printing mid-session for prompt iteration without burning tokens.
- **CLI-managed defaults** — `llmctl config get|set|unset|list|path` writes a `key = value` defaults file (atomic, comment-preserving) so any flag can have a project- or user-level fallback.
- **Single binary** — Zig 0.16, ~1.3 MB, no runtime dependencies.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| **`llmctl`** | Direct chat-completion API client (debugging, scripting) |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |
| [`secretctl`](/secretctl/) | Encrypted secret store + capability injection |

`llmctl` is a leaf — it talks HTTP to a model and gets out of the way. Pair with `secretctl exec --tag ai -- llmctl …` to keep API keys out of your shell environment.

## Get started

Build from source (release tarballs coming):

```sh
git clone https://github.com/agent-rt/llmctl
cd llmctl
zig build -Doptimize=ReleaseSafe
./zig-out/bin/llmctl --version
```

Requires Zig 0.16.

## Defaults

`~/.config/llmctl/defaults` — one `key = value` per line — sets fallback values applied before CLI parsing (any flag overrides). Recognised keys: `provider`, `model`, `base_url`, `system`, `max_tokens`, `temperature`, `top_p`.

```
provider = local
base_url = http://10.0.0.64:8800
model    = unsloth/gemma-4-26B-A4B-it-GGUF:gemma-4-26B-A4B-it-UD-Q4_K_M
max_tokens = 4096
```

Manage the file from the CLI (preserves comments and blank lines, atomic writes):

```sh
llmctl config list                       # print all currently-set keys
llmctl config get model                  # print one value
llmctl config set provider openai        # write/update
llmctl config unset model
llmctl config path                       # print resolved file path
```

Search order: `$LLMCTL_DEFAULTS`, `$XDG_CONFIG_HOME/llmctl/defaults`, `~/.config/llmctl/defaults`.

## REPL slash commands

`llmctl -i` enters an interactive REPL. Inside:

| Command | Effect |
|---|---|
| `/help`, `/?` | List all commands |
| `/exit`, `/quit`, `/q` | Leave the REPL (Ctrl-D also works) |
| `/clear` | Drop conversation history (keep system prompt) |
| `/system [<text>]` | Show or set the system prompt |
| `/model [<name>]` | Show or switch model |
| `/tokens` | Print accumulated input/output token counts |
| `/save <path>` / `/load <path>` | Persist or restore the session JSON |
| `/history` | List current messages |
| `/info` | Provider, model, system, auto-save, dry-run state |
| `/dry-run [on\|off]` | Toggle request-printing — next turn prints the body instead of sending |

## Links

- Source: [github.com/agent-rt/llmctl](https://github.com/agent-rt/llmctl)
- License: Apache-2.0
