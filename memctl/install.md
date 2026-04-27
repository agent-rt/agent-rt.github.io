---
title: Install · memctl
---

# Install

## Platforms

`memctl` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Homebrew (recommended on macOS)

```sh
brew tap agent-rt/tap
brew install memctl
```

`brew upgrade memctl` upgrades when a new tagged release ships.

## Prebuilt binary

```sh
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/memctl/releases/download/v${VERSION}/memctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "memctl-${VERSION}-${TARGET}/memctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/memctl/releases/download/v${VERSION}/memctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "memctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.83+.

```sh
git clone https://github.com/agent-rt/memctl.git
cd memctl
cargo install --path apps/memctl-cli --locked --force
```

Installs `memctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

## Optional: ripgrep for faster search

`memctl search` uses an internal regex engine by default. If `rg` is on your `PATH`, future versions can delegate to it for large topic stores. Either way, plain markdown means standard tools (`grep`, `rg`, `ripgrep-all`, …) work directly on `~/.memctl/global/topics/*.md`.

## Verify

```sh
memctl --version
memctl list --format tsv
```

The first time you run a write command, `~/.memctl/global/topics/` is created. List on a fresh install returns just the TSV header.

## Set your agent identity (optional but recommended)

Each entry records `source = <agent-name> @ <project-path>`. Set the agent name explicitly so cross-tool entries are attributable:

```sh
# In Claude Code's wrapper / shell init / agent harness:
export MEMORYCTL_AGENT=claude-code
```

Without this, entries record `unknown @ ...`.

## Uninstall

```sh
brew uninstall memctl                 # if installed via Homebrew
cargo uninstall memctl-cli            # if installed via cargo
sudo rm /usr/local/bin/memctl         # if installed via tarball

# Optional: drop all stored memory (irreversible)
rm -rf ~/.memctl
```
