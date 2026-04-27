---
title: Install · memoryctl
---

# Install

## Platforms

`memoryctl` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Homebrew (recommended on macOS)

```sh
brew tap agent-rt/tap
brew install memoryctl
```

`brew upgrade memoryctl` upgrades when a new tagged release ships.

## Prebuilt binary

```sh
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/memoryctl/releases/download/v${VERSION}/memoryctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "memoryctl-${VERSION}-${TARGET}/memoryctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/memoryctl/releases/download/v${VERSION}/memoryctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "memoryctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.83+.

```sh
git clone https://github.com/agent-rt/memoryctl.git
cd memoryctl
cargo install --path apps/memoryctl-cli --locked --force
```

Installs `memoryctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

## Optional: ripgrep for faster search

`memoryctl search` uses an internal regex engine by default. If `rg` is on your `PATH`, future versions can delegate to it for large topic stores. Either way, plain markdown means standard tools (`grep`, `rg`, `ripgrep-all`, …) work directly on `~/.memoryctl/global/topics/*.md`.

## Verify

```sh
memoryctl --version
memoryctl list --format tsv
```

The first time you run a write command, `~/.memoryctl/global/topics/` is created. List on a fresh install returns just the TSV header.

## Set your agent identity (optional but recommended)

Each entry records `source = <agent-name> @ <project-path>`. Set the agent name explicitly so cross-tool entries are attributable:

```sh
# In Claude Code's wrapper / shell init / agent harness:
export MEMORYCTL_AGENT=claude-code
```

Without this, entries record `unknown @ ...`.

## Uninstall

```sh
brew uninstall memoryctl                 # if installed via Homebrew
cargo uninstall memoryctl-cli            # if installed via cargo
sudo rm /usr/local/bin/memoryctl         # if installed via tarball

# Optional: drop all stored memory (irreversible)
rm -rf ~/.memoryctl
```
