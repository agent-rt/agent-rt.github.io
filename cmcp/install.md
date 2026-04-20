---
title: Install · cmcp
---

# Install

## Platforms

`cmcp` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Prebuilt binary

Pick a release from
[github.com/agent-rt/cmcp/releases](https://github.com/agent-rt/cmcp/releases),
then:

```sh
# replace <VERSION> and <TARGET>
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/cmcp/releases/download/v${VERSION}/cmcp-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "cmcp-${VERSION}-${TARGET}/cmcp" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/cmcp/releases/download/v${VERSION}/cmcp-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "cmcp-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.75+.

```sh
git clone https://github.com/agent-rt/cmcp.git
cd cmcp
just install      # → cargo install --path . --locked --force
```

Installs `cmcp` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

Or directly from crates.io (once published):

```sh
cargo install cmcp
```

## Verify

```sh
cmcp --version
cmcp config sources
```

`config sources` prints the discovered config files — it should list at
least `cmcp` (your own), plus any Agent config you have installed.

## Uninstall

```sh
# cargo
cargo uninstall cmcp

# prebuilt
sudo rm /usr/local/bin/cmcp

# own config (safe to keep; it's just JSON)
rm -rf ~/.config/cmcp/
```
