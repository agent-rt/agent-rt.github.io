---
title: llmctl
---

# llmctl

`llmctl` is a **fast, pipe-friendly CLI for testing OpenAI-compatible and Anthropic LLM endpoints**. One single Zig binary speaks Chat Completions, OpenAI-compatible servers (llama-server, vLLM, Ollama, …), and Anthropic Messages — through a provider abstraction where `format × timing` (text/json/ndjson × stream/batch) are orthogonal.

```sh
llmctl "explain recursion"
echo "summarize" | llmctl < article.txt
llmctl --base-url http://10.0.0.64:8800 --model gemma "hi"
llmctl --output ndjson "hi" | jq .
llmctl --provider anthropic --model claude-sonnet-4-5 "hi"
llmctl -i                                          # REPL with slash commands
```

## Why one CLI for every chat API

`llmctl` is opinionated for users who **debug, script, and compare LLM endpoints** from the shell:

- **Provider as a triple of pure functions** — `builder` / `stream_decoder` / `batch_decoder`. Adding a new provider is three functions, not a new code path. `local` (llama-server), `openai`, `openai-compat`, and `anthropic` ship built-in.
- **Concurrent multi-model** — repeat `--model` to fan out the same prompt to multiple models in parallel. Output comes back as tagged NDJSON, ready for `jq`.
- **`format × timing` is orthogonal** — `--output text|json|ndjson` and `--buffer/--no-stream` compose freely. Stream NDJSON to a TUI, buffer JSON for scripting, stream text for humans.
- **`--extra` passthrough** — any provider-specific knob (`cache_prompt`, `seed`, `repeat_penalty`, …) goes through type-inferred without a CLI flag for it. `--extra-json '{...}'` for nested objects.
- **Sessions + REPL** — `llmctl -i` gives a 10-command slash REPL; conversations persist via `--session path.json` and fork via `--save-session`.
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

`~/.config/llmctl/defaults` — one `key=value` per line — sets fallback values for any flag:

```
provider=local
base-url=http://10.0.0.64:8800
model=unsloth/gemma-4-26B-A4B-it-GGUF:gemma-4-26B-A4B-it-UD-Q4_K_M
max-tokens=4096
```

## Links

- Source: [github.com/agent-rt/llmctl](https://github.com/agent-rt/llmctl)
- License: Apache-2.0
