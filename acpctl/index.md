---
title: acpctl
---

# acpctl

`acpctl` is the **ACP control utility for AI agents** — a single-binary CLI
that lets agents discover, inspect, and invoke any
[ACP (Agent Client Protocol)][acp] agent with tagged NDJSON output,
structured errors, and token-lean defaults.

```sh
# Agent perspective: introspect, register, converse
acpctl schema                                      # self-describing contract
acpctl inspect gemini                              # InitializeResponse as JSON
acpctl call gemini --prompt-file - < prompt.txt    # stream events + result
```

[acp]: https://agentclientprotocol.com/

## Why agent-first

`acpctl` is infrastructure for agents, not a developer terminal UI:

- **Stable output contract.** Stream commands emit tagged NDJSON
  (`{"type":"notification"|"result"|"error",...}`); single-result commands
  emit one JSON object on stdout. Errors are always
  `{"error":{"code":"...","message":"...","hint":"..."}}` on stderr with
  stratified exit codes (1 user, 2 protocol, 3 daemon/io, 4 unexpected).

- **Token-lean defaults.** Drops `agent_thought_chunk` and
  `available_commands_update` by default — measured **7.4× byte reduction**
  on real Gemini output (516 B vs 3830 B). Opt back in with
  `--verbose-events`; cap any event with `--max-chunk-bytes N`.

- **Stdin / JSON prompts.** `--prompt-file -` reads from stdin;
  `--prompt-json` accepts a structured `ContentBlock[]` array — no fighting
  the shell over multi-line quoting.

- **TTY-aware formatting.** JSON when piped, pretty-printed on a terminal,
  overridable via `ACPCTL_FORMAT`.

- **Persistent sessions across CLI invocations.** A background daemon keeps
  the agent subprocess alive between calls, so conversations survive even
  when the underlying agent (e.g. `gemini-cli`) doesn't support
  cross-process `session/load`.

- **Machine-readable self-description.** `acpctl schema` emits a compact
  JSON document listing commands, flags, event types, error codes, and
  exit codes — an agent never has to parse `--help`.

## Design principles

- **Process model matches workload.** One-shot `call` uses a short-lived
  subprocess. Persistent conversations go through `session`, backed by a
  long-lived daemon that holds `ConnectionTo<Agent>` alive inside a single
  SDK `connect_with` closure.

- **Stable error surface.** Every error has a stable string code in
  `acpctl schema`; new codes are additive, existing codes never change
  meaning. Agents can branch on code, not on prose.

- **Least privilege by default.** `--approve read-only` is the default
  permission; agents must opt in to filesystem writes or terminal access.

- **No hidden magic.** `~/Library/Application Support/dev.acpctl.acpctl/`
  (or XDG equivalent) holds all state: `config.json`, `sessions/`,
  `runtime/` (UDS socket, PID file, log). `acpctl config path` tells
  you exactly where.

## Get started

1. **[Install](/acpctl/install/)** — cargo, prebuilt tarball, or Homebrew.
2. **[Quickstart](/acpctl/quickstart/)** — first call in 30 seconds; first
   cross-CLI conversation in two minutes.
3. **[Configuration](/acpctl/configuration/)** — agent specs, permissions,
   the daemon, paths.
4. **[Commands](/acpctl/commands/)** — full reference.
5. **[Troubleshooting](/acpctl/troubleshooting/)** — exit codes, common
   errors, protocol gotchas.

## Status

Pre-release — no tagged version yet. The source is stable and regressed
against `gemini --experimental-acp`; first tag will be `v0.1.0`.

## Links

- Source: [github.com/agent-rt/acpctl](https://github.com/agent-rt/acpctl)
- Releases: [github.com/agent-rt/acpctl/releases](https://github.com/agent-rt/acpctl/releases)
- Issues: [github.com/agent-rt/acpctl/issues](https://github.com/agent-rt/acpctl/issues)
