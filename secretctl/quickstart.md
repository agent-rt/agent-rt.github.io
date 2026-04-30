---
title: Quickstart · secretctl
---

# Quickstart

Goal: vault from zero to first agent-safe `npm install` in under two minutes.

Prereq: `secretctl` installed (see [Install](/secretctl/install/)).

## 1. Initialize the vault

```sh
secretctl init
```

You will be asked for a master password (twice) and whether to enable the macOS Keychain protector. Answer **y** to the Keychain prompt — every subsequent command unlocks transparently without re-entering the password.

```
Master password: ********
Confirm password: ********
Use macOS Keychain to skip password on subsequent runs? [Y/n] y
vault created at /Users/you/.secretctl
```

The vault is **empty**. `master.key` carries two protectors: passphrase (Argon2id-derived wrapping key) and Keychain (random wrapping key stored in your login keychain). Either one alone unlocks the vault.

## 2. Add a few secrets

`secretctl add` enters a TUI for the value. It deliberately rejects any positional argument that could put the plaintext on the shell command line:

```sh
secretctl add NPM_TOKEN --tag npm
# Name:        NPM_TOKEN
# Value:       ____________________
# Tags:        (you provided npm)
```

For multi-line secrets like SSH keys, use `--editor` to open `$VISUAL` / `$EDITOR` (default `vi`):

```sh
secretctl add SSH_KEY --tag ssh --editor
```

The temp file lives in `$TMPDIR` with mode `0600`, is `ftruncate(0)`'d on read, and `unlink()`'d before the editor sees a SIGPIPE.

## 3. List, agent-style

```sh
secretctl list --json
```

```json
[
  {"name":"NPM_TOKEN","tags":["npm"],"created_at":1761859200,"updated_at":1761859200},
  {"name":"SSH_KEY","tags":["ssh"],"created_at":1761859260,"updated_at":1761859260}
]
```

Note what is **not** in the output: any value, ever. `--json` is the agent-friendly format; the bare `secretctl list` produces a colored table for humans.

## 4. Lock down the project

In the project root, declare an allowlist:

```sh
cat > .secretctl.toml <<'EOF'
[allow]
tags     = ["npm"]
commands = ["npm", "yarn", "pnpm", "node"]
EOF
```

Now `exec` only injects secrets whose tag is in `allow.tags` *and* runs commands whose basename is in `allow.commands`:

```sh
secretctl exec --tag npm -- npm install      # OK
secretctl exec --tag ssh -- ssh server       # rejected: tag not allowed (exit 2)
secretctl exec --tag npm -- sh -c '…'        # rejected: command not allowed (exit 2)
```

Without a `.secretctl.toml`, `exec` still requires explicit `--tag` or `--only` (no implicit injection), but does not gate which commands you can run.

## 5. Render a config file

For tools that read from a file rather than env vars (`.npmrc`, `.env`, custom configs), use `render` with `${NAME}` placeholders:

```sh
cat > .npmrc.tmpl <<'EOF'
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
EOF

secretctl render .npmrc.tmpl --out ~/.npmrc
```

The output file gets mode `0600`. Use `$$` in the template to emit a literal `$`.

## 6. Edit / rotate

```sh
secretctl edit NPM_TOKEN     # opens $EDITOR with current value
secretctl rm STALE_TOKEN
secretctl reveal NPM_TOKEN   # prints to TTY only; refuses if stdout is a pipe
```

`edit` preserves tags and metadata. If you save without changing the bytes, it prints `unchanged` and never re-encrypts (skipping a pointless `body_nonce` roll).

## 7. What an agent sees

In a typical Claude Code session the agent runs the following itself:

```sh
secretctl list --json                              # discover capabilities
secretctl exec --tag npm -- npm install            # use them
```

It cannot:

- read plaintext (`reveal` requires a TTY; `--allow-all` is rejected outright)
- inject secrets into commands not in the allowlist
- bypass the allowlist by naming secrets directly via `--only`

Every `exec`, `render`, `edit`, `reveal` writes a single line to `~/Library/Logs/secretctl.log` — useful when reviewing what an agent did during a long session.

## Next

- **[Commands](/secretctl/commands/)** — every flag, every exit code.
- Source & issues: [github.com/agent-rt/secretctl](https://github.com/agent-rt/secretctl).
