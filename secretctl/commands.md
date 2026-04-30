---
title: Commands · secretctl
---

# Commands

`secretctl` ships 11 commands. Use `secretctl --help` for the inline summary.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | success |
| `1` | internal failure (oom, fs, decrypt) |
| `2` | usage error / policy reject / not found |
| `126` | `exec` child found but not executable |
| `127` | `exec` child not found |
| *N* | `exec` child's exit code (transparent passthrough) |

## `init`

```sh
secretctl init
```

Creates `~/.secretctl/{vault,master.key,config.toml}` (all mode `0600`). Prompts for a master password (Argon2id-derived wrapping key) and offers to enable the macOS Keychain protector.

Refuses to overwrite an existing `master.key`. Requires a TTY (set `SECRETCTL_BATCH=1` to bypass for scripted setup).

## `add`

```sh
secretctl add NAME [--tag X,Y] [--editor]
```

Adds a new secret. The value is **never** taken from a positional argument or stdin — only from a non-echoing TTY prompt or `$EDITOR`.

- `--tag X,Y` — comma-separated tags, used by `exec --tag` and `.secretctl.toml` allowlist.
- `--editor` — open `$VISUAL` / `$EDITOR` (default `vi`) on a `0600` temp file in `$TMPDIR`. Empty file aborts the add.

A trailing newline is stripped (matches editor save convention). The temp file is `ftruncate(0)` + `unlink`'d on read-back.

## `edit`

```sh
secretctl edit NAME
```

Open `$EDITOR` with the current plaintext. On save:

- bytes unchanged → prints `unchanged`, no re-encrypt
- non-empty new bytes → atomically replaces the secret (preserving `tags`)
- empty file → aborts with exit 1 (use `rm` to delete)

The decrypted plaintext lives in the temp file only for the editor's lifetime. Same `0600` + `ftruncate` + `unlink` discipline as `add --editor`.

## `rm`

```sh
secretctl rm NAME
```

Delete a secret by name (case-insensitive). Exits 2 if not found.

## `list`

```sh
secretctl list [--json] [--tag X]
```

Without flags: colored table (`NAME  TAGS  UPDATED`).

With `--json`: stable schema for agents.

```json
[
  {"name":"NPM_TOKEN","tags":["npm"],"created_at":1761859200,"updated_at":1761859200}
]
```

`--tag X` filters to secrets carrying tag `X`. **No flag, ever, surfaces values.**

## `exec`

```sh
secretctl exec [--tag X] [--only N1,N2] -- COMMAND ARGS...
```

Decrypts the selected secrets, sets them as environment variables on the child, and forks `COMMAND`. The parent process zeroes every plaintext buffer before `waitpid` returns; the child's exit code is passed through transparently.

Selection rules:

- at least one of `--tag` or `--only` is required (no implicit injection)
- `--tag X[,Y]` — every secret carrying any listed tag
- `--only N1[,N2]` — exact name match; missing names exit 2
- `--allow-all` is **rejected** (no escape hatch)

Policy enforcement (when `.secretctl.toml` is present in cwd or any ancestor):

- child's `argv[0]` basename must appear in `allow.commands`
- every `--tag` value must appear in `allow.tags`
- every selected secret (regardless of selection method) must have at least one tag in `allow.tags` — closes the `--only` bypass

```toml
# .secretctl.toml
[allow]
tags     = ["npm", "github"]
commands = ["npm", "yarn", "pnpm", "gh", "node"]
```

A line is appended to `~/Library/Logs/secretctl.log`:

```
2026-04-30T03:35:41Z exec cmd=npm tags=npm cwd=/path/to/proj exit=0
```

Argv beyond `argv[0]` is **never** logged.

## `materialize`

```sh
secretctl materialize NAME --out PATH [--mode MODE] [--mkdir]
```

Decrypt a single secret and write its value byte-for-byte to `PATH`. No
trailing newline is appended (so SSH private keys round-trip cleanly).
Default mode is `0600`; `--mkdir` creates parent directories with mode
`0700` if they don't exist.

This is the primitive used by the Home Manager module to populate
`~/.ssh/*` and `~/.config/secretctl/env/*` at activation time. See
[Nix integration](/secretctl/nix/) for the declarative wrapper.

## `render`

```sh
secretctl render TEMPLATE --out PATH
```

Reads `TEMPLATE`, replaces `${NAME}` with the value of secret `NAME`, writes the result to `PATH` with mode `0600`. Use `$$` to emit a literal `$`. Refuses to overwrite an existing `PATH`.

Useful for `.npmrc`, `.env`, and any config tool that doesn't read env vars at runtime.

## `mcp`

```sh
secretctl mcp [--cwd PATH] [--allow-secret-read]
```

Start an MCP (Model Context Protocol) server on stdio. Three tools by
default — `list_secrets`, `check_secret_available`, `run_with_secrets`.
`--allow-secret-read` adds a fourth tool, `get_secret`, gated by
per-call Touch ID; the server refuses to start without biometry
hardware.

The same `.secretctl.toml` allowlist gates `run_with_secrets` as it
does the CLI `exec`, so an agent can only execute commands you've
explicitly trusted with the matching tags.

`--cwd PATH` overrides the project root used when looking up
`.secretctl.toml`; defaults to the server's current working directory.

## `reinstall-keychain`

```sh
secretctl reinstall-keychain [--no-touch-id]
```

Rebuild the macOS Keychain protector. Existing keychain ACLs cannot be
modified in place, so use this command after upgrading from a version
of `secretctl` whose ACL doesn't suit you (e.g. enabling Touch ID on a
vault that was originally created with the trusted-app ACL, or vice
versa).

## `reveal`

```sh
secretctl reveal NAME
```

Print plaintext on the TTY. Refuses if stdout is not a TTY (so it cannot be silently captured to a file or piped to a logger). Logs `op=reveal` to the audit trail.

There is no `--clipboard` option in 0.1.x. Use `secretctl reveal NAME | pbcopy` if you really mean it (the TTY check rejects this; bypass via `SECRETCTL_BATCH=1` only for trusted scripted setups).

## Environment

| Variable | Effect |
|---|---|
| `$VISUAL`, `$EDITOR` | Editor for `add --editor` and `edit`. Default `vi`. |
| `$SECRETCTL_HOME` | Override `~/.secretctl` (testing & multi-vault setups). |
| `$SECRETCTL_BATCH` | `=1` skips TTY enforcement on `init`, `reveal`, and password prompts (reads from stdin). For scripted setup; do **not** use on workstations. |
| `$TMPDIR` | Where editor temp files live. Inherited from macOS shell. |

## Files

| Path | Mode | Purpose |
|---|---|---|
| `~/.secretctl/vault` | `0600` | Encrypted secrets (single-file AEAD body). |
| `~/.secretctl/master.key` | `0600` | KDF parameters + protector container + HMAC. |
| `~/.secretctl/config.toml` | `0600` | Non-sensitive config (reserved for future). |
| `~/Library/Logs/secretctl.log` | `0600` | Append-only audit log. |
| `$TMPDIR/secretctl-edit-*` | `0600` | Editor temp file; lifetime = editor process. |
| `.secretctl.toml` (project) | any | `[allow]` policy block. |
