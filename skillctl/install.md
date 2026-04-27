---
title: Install · skillctl
---

# Install

## Platforms

`skillctl` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Homebrew (recommended on macOS)

```sh
brew tap agent-rt/tap
brew install skillctl
```

`brew upgrade skillctl` upgrades when a new tagged release ships.

## Prebuilt binary

Pick a release from
[github.com/agent-rt/skillctl/releases](https://github.com/agent-rt/skillctl/releases),
then:

```sh
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/skillctl/releases/download/v${VERSION}/skillctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "skillctl-${VERSION}-${TARGET}/skillctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/skillctl/releases/download/v${VERSION}/skillctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "skillctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.83+.

```sh
git clone https://github.com/agent-rt/skillctl.git
cd skillctl
cargo install --path apps/skillctl-cli --locked --force
```

Installs `skillctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

To build without the `use` TUI (smaller binary, no `ratatui` dependency):

```sh
cargo install --path apps/skillctl-cli --locked --no-default-features --force
```

## Verify

```sh
skillctl --version
skillctl list --global --format tsv
```

The first time you run any command that touches the global store,
`~/.skillctl/skills/` and `~/.skillctl/profiles/` are created automatically.

## Uninstall

```sh
brew uninstall skillctl                # if installed via Homebrew
cargo uninstall skillctl-cli           # if installed via cargo
sudo rm /usr/local/bin/skillctl        # if installed via tarball

# Optional: drop the global store (skills + profiles + lock + trust list)
rm -rf ~/.skillctl
```
