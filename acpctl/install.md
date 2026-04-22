---
title: Install · acpctl
---

# Install

## Platforms

`acpctl` 0.3.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported (UDS session daemon requires Unix). Install from
source works on any target Rust builds.

## Homebrew (macOS)

```sh
brew tap agent-rt/tap
brew install acpctl
```

## Prebuilt binary

Pick a release from
[github.com/agent-rt/acpctl/releases](https://github.com/agent-rt/acpctl/releases),
then:

```sh
VERSION=0.3.2
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/acpctl/releases/download/v${VERSION}/acpctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "acpctl-${VERSION}-${TARGET}/acpctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/acpctl/releases/download/v${VERSION}/acpctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "acpctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.85+ (edition 2024).

```sh
cargo install --locked --git https://github.com/agent-rt/acpctl
# or
git clone https://github.com/agent-rt/acpctl.git
cd acpctl
cargo install --path . --locked --force
```

Installs `acpctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

## Verify

```sh
acpctl --version
acpctl schema | jq '.commands | keys'
```

The second command should print the full list of subcommands.

## Uninstall

```sh
# cargo
cargo uninstall acpctl

# Homebrew
brew uninstall acpctl && brew untap agent-rt/tap

# prebuilt
sudo rm /usr/local/bin/acpctl

# state (safe to keep; daemon socket + JSON configs)
rm -rf "$HOME/Library/Application Support/dev.acpctl.acpctl/"   # macOS
rm -rf "$HOME/.local/share/acpctl/"                             # Linux XDG
```
