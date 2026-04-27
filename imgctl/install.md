---
title: Install · imgctl
---

# Install

## Platforms

`imgctl` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Homebrew (recommended on macOS)

```sh
brew tap agent-rt/tap
brew install imgctl
```

`brew upgrade imgctl` upgrades when a new tagged release ships.

## Prebuilt binary

```sh
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/imgctl/releases/download/v${VERSION}/imgctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "imgctl-${VERSION}-${TARGET}/imgctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/imgctl/releases/download/v${VERSION}/imgctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "imgctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.85+ (uses 2024 edition).

```sh
git clone https://github.com/agent-rt/imgctl.git
cd imgctl
cargo install --path apps/imgctl-cli --locked --force
```

Installs `imgctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

## Optional: Chrome / Chromium for `mermaid`

The `imgctl mermaid` subcommand renders Mermaid diagrams via headless Chrome (through the `chromiumoxide` crate). Install one of:

- **macOS**: Chrome / Chromium / Microsoft Edge — auto-detected.
- **Linux**: `apt install chromium-browser` or `pacman -S chromium`.

Other commands (convert, resize, crop, annotate, analyze) work without Chrome — they're pure Rust on top of `image` + `imageproc`.

## Verify

```sh
imgctl --version
imgctl --help

# Sanity test
echo "graph TD; A-->B" > /tmp/d.mmd
imgctl mermaid -i /tmp/d.mmd -o /tmp/d.png
file /tmp/d.png   # → PNG image data
```

## Uninstall

```sh
brew uninstall imgctl                    # if installed via Homebrew
cargo uninstall imgctl-cli               # if installed via cargo
sudo rm /usr/local/bin/imgctl            # if installed via tarball
```
