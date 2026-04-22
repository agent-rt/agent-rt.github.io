---
title: Install · mcpctl
---

# Install

## Platforms

`mcpctl` 0.1.x ships prebuilt binaries for:

- `x86_64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Windows is not supported. Install from source works on any target Rust builds.

## Prebuilt binary

Pick a release from
[github.com/agent-rt/mcpctl/releases](https://github.com/agent-rt/mcpctl/releases),
then:

```sh
# replace <VERSION> and <TARGET>
VERSION=0.1.0
TARGET=aarch64-apple-darwin  # or x86_64-apple-darwin, x86_64-unknown-linux-gnu

curl -L "https://github.com/agent-rt/mcpctl/releases/download/v${VERSION}/mcpctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "mcpctl-${VERSION}-${TARGET}/mcpctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/mcpctl/releases/download/v${VERSION}/mcpctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "mcpctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires Rust 1.75+.

```sh
git clone https://github.com/agent-rt/mcpctl.git
cd mcpctl
cargo install --path . --locked --force
```

Installs `mcpctl` to `~/.cargo/bin/`. Make sure that directory is on `PATH`.

## Verify

```sh
mcpctl --version
mcpctl config sources
```

`config sources` prints the discovered config files — it should list at
least `mcpctl` (your own), plus any agent config you have installed.

## Migration from cmcp

If you previously used `cmcp`, your config at `~/.config/cmcp/mcp.json` is
automatically migrated to `~/.config/mcpctl/mcp.json` on first run. No
manual action required.

## Uninstall

```sh
# cargo
cargo uninstall mcpctl

# prebuilt
sudo rm /usr/local/bin/mcpctl

# own config (safe to keep; it's just JSON)
rm -rf ~/.config/mcpctl/
```
