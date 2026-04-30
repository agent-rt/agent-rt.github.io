---
title: secretctl
---

# secretctl

`secretctl` is an **agent-first single-binary secret manager for macOS**. Tokens, API keys, and SSH keys live in a single AES-256-GCM encrypted file under `~/.secretctl/`. AI agents (Claude Code, Codex, Cursor, …) get *capability* access — they run commands with secrets injected as environment variables, but never see the plaintext.

```sh
secretctl init                                # passphrase + macOS Keychain
secretctl add OPENAI_API_KEY --tag ai         # TUI (single line)
secretctl add SSH_KEY --tag ssh --editor      # $EDITOR (multi-line)

secretctl list --json                         # name+tags only, no value
secretctl exec --tag ai -- python main.py     # env-injected, audited
secretctl render .npmrc.tmpl --out ~/.npmrc   # ${NAME} substitution
```

## Why agent-first

`secretctl` is opinionated for users who **share a shell with AI agents** and want capability-based, not credential-based, access:

- **Project-local allowlist** — `.secretctl.toml` declares which `tags` and `commands` are reachable. `exec` enforces the allowlist on both `--tag` and `--only` paths, so an agent cannot bypass restrictions by naming a secret directly.
- **Encrypted metadata** — secret names, tags, and timestamps live inside the AEAD body. `strings(1)` on the vault file shows nothing useful — strictly stronger than SOPS' field-level encryption.
- **Zero ambient state** — no `.env` files, no shell history, no process arguments. CLI rejects `secretctl add NAME value` outright.
- **Capability injection** — `exec --tag ai -- cmd` decrypts only the matching secrets, sets them as env vars on the child, and clears every plaintext buffer in the parent before `waitpid`. Subprocess exit code is transparently passed through.
- **Native Keychain protector** — second-and-later runs unlock through the macOS Keychain, no password prompt. Touch ID hookup planned for a future minor.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| **`secretctl`** | Encrypted secret store + capability-based injection |
| [`skillctl`](/skillctl/) | Curated skill bundles + profile-driven `AGENTS.md` |
| [`memctl`](/memctl/) | Persistent agent memory |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |

`secretctl` is the only tool in the family that holds **plaintext** — every other tool is configuration or orchestration. It deliberately stays a leaf dependency: nothing in the family imports it; it integrates by being on `PATH` and consumed via `exec` and `render`.

## Get started

1. **[Install](/secretctl/install/)** — Homebrew (recommended) or release tarball.
2. **[Quickstart](/secretctl/quickstart/)** — vault to first injected `npm install` in under two minutes.
3. **[Commands](/secretctl/commands/)** — reference for all 8 commands.

## Links

- Source: [github.com/agent-rt/secretctl](https://github.com/agent-rt/secretctl)
- Releases: [github.com/agent-rt/secretctl/releases](https://github.com/agent-rt/secretctl/releases)
- License: Apache-2.0
