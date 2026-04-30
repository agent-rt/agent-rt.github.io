---
title: Cross-Mac sync · secretctl
---

# Cross-Mac sync via git

Want the same vault on your laptop and your home Mac without depending on
iCloud or any Apple ID feature? Push the encrypted vault files to a private
git repo and let each Mac unlock locally with its own keychain protector.

## How it works

```
~/.secretctl/                      ← git working tree on every Mac
  master.key   ← in git (ciphertext + protector list, including N entries
                 — one per Mac)
  vault        ← in git (AES-256-GCM body)
  config.toml  ← in git (non-sensitive)

(Each Mac's own random wrapping_key stays in its local Keychain — never
 in git, never crossing the network.)
```

`master.key` ships a **list of protectors**: passphrase + one keychain
protector per machine. The encrypted master_key inside is identical for
every protector — so any one of them unlocks the vault. Adding a new
machine means appending a new protector entry; removing one means
deleting it. The vault file itself is the same encrypted blob no matter
which Mac wrote it.

## Bootstrap

### 1. On the first Mac (let's call it "macbook")

You already have a vault from `secretctl init`. Make it a git repo and
push to a private remote (GitHub / Codeberg / self-hosted — anything
that takes a git push):

```bash
cd ~/.secretctl
git init -b main
cat > .gitignore <<'EOF'
*.tmp
*.bak
EOF
git add .gitignore master.key vault config.toml
git commit -m "init: vault baseline"

gh repo create yourname/secretctl-vault --private --source=. --remote=origin --push
```

The repo will store ciphertext only — anyone who clones it without your
password (and without your Mac's local keychain wrapping_key) sees
opaque AES-GCM bytes.

### 2. On the second Mac (e.g. "macmini")

```bash
git clone git@github.com:yourname/secretctl-vault.git ~/.secretctl
chmod 700 ~/.secretctl
chmod 600 ~/.secretctl/{master.key,vault,config.toml}
```

The vault is unlockable now via the master password — the passphrase
protector works on every machine. Verify:

```bash
secretctl list --json   # password prompt → vault contents
```

But you don't want to type the password every time on macmini. Add this
machine's own keychain protector:

```bash
secretctl key add-keychain-protector
# Master password required (we use it once to unwrap master_key, then
# encrypt it again with a fresh random wrapping_key stored in macmini's
# Keychain).
```

`master.key` now has one extra protector entry. Push it back so macbook
sees it too:

```bash
secretctl sync
# vault: macmini-name 1777512345
```

## Daily workflow

After any vault edit (`add` / `edit` / `rm`):

```bash
secretctl sync
```

Before any vault edit (if you've used the other Mac recently):

```bash
secretctl sync   # pulls + commits any drift, then pushes
```

`sync` is the entire git ceremony in one step — `git add -A`, commit if
there are local changes, `git pull --ff-only`, then `git push`. Audit
log records `op=sync status=committed|nochange|diverged`.

If you want it automatic, alias it:

```bash
# zsh
alias add='secretctl add'
secretctl_add() { command secretctl add "$@" && secretctl sync; }
```

But manual is safer — diverged history shows up explicitly.

## Conflicts

`vault` is binary; `master.key` is binary; neither will three-way merge.
If both Macs edit and push without syncing first, you'll get:

```
git pull --ff-only failed (history has diverged); resolve manually:
  cd ~/.secretctl && git pull   # then `git checkout --theirs vault` (or --ours) and commit
```

Decide which side wins:

```bash
cd ~/.secretctl
git fetch origin
git checkout --theirs vault         # keep remote (origin) version
# or
git checkout --ours vault           # keep local version
git add vault
git commit -m "resolve conflict: keep <which> side"
git push
```

If both sides have **non-overlapping** changes (you added a token on
macbook, and a different token on macmini), you have to redo the lost
side's edit by hand. There's no automatic three-way merge for binary
ciphertext.

In practice: do `secretctl sync` before every edit on a Mac you haven't
used in a while, and you'll almost never see this.

## Privacy considerations

| Surface | Visibility |
|---|---|
| File contents on remote | AES-256-GCM ciphertext — useless without password / wrapping_key |
| Commit timestamps | Reveal *when* you change secrets; not *what* |
| Commit author | The git config user — your name/email if defaulted |
| Commit message | We use `vault: <hostname> <unix-ts>` — your hostname leaks |
| Push frequency | Reveals "active vault editing periods" |

To minimize hostname / user-name leakage, set per-repo git config:

```bash
cd ~/.secretctl
git config user.email "vault@example.com"
git config user.name "vault"
```

For maximum opacity, host the remote on a server you control rather
than GitHub.

## Removing a machine

If a Mac is lost or compromised, you want to remove its protector
entry. There's no `key remove-protector` command yet (Phase 5 polish).
Manual workaround until then:

```bash
# 1. From any other Mac:
secretctl reinstall-keychain   # creates a fresh wrapping_key on this Mac
# 2. Edit master.key out-of-band? Not yet supported via CLI.
```

For now, if you lose a Mac, your safest move is `secretctl key
change-password` (also Phase 5 polish — coming) followed by
re-creating each Mac's keychain protector. Or, if the lost Mac's
local Keychain is encrypted (FileVault on with strong password) and
the password protector is strong, the risk of an attacker actually
unlocking your vault is low.

## What's next

- `secretctl key remove-protector <id>` (planned)
- `secretctl key change-password` (planned)
- Conflict-aware vault format that allows append-only secret addition
  to merge automatically (research)
