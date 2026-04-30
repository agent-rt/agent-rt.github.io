---
title: Install · secretctl
---

# Install

## Platforms

`secretctl` 0.1.x ships a single prebuilt binary:

- `aarch64-apple-darwin` (Apple Silicon)

Intel Mac, Linux, and Windows are **not supported**. The vault format and FFI bindings target macOS Security.framework specifically.

## Homebrew (recommended)

```sh
brew tap agent-rt/tap
brew install secretctl
```

`brew upgrade secretctl` upgrades when a new tagged release ships. The formula declares `depends_on arch: :arm64`, so an Intel Mac install fails fast with a clear error rather than at runtime.

## Prebuilt binary

Pick a release from
[github.com/agent-rt/secretctl/releases](https://github.com/agent-rt/secretctl/releases),
then:

```sh
VERSION=0.1.0
TARGET=aarch64-apple-darwin

curl -L "https://github.com/agent-rt/secretctl/releases/download/v${VERSION}/secretctl-${VERSION}-${TARGET}.tar.gz" \
  | tar -xz
sudo mv "secretctl-${VERSION}-${TARGET}/secretctl" /usr/local/bin/
```

Verify checksum:

```sh
curl -LO "https://github.com/agent-rt/secretctl/releases/download/v${VERSION}/secretctl-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "secretctl-${VERSION}-${TARGET}.tar.gz.sha256"
```

## From source

Requires [Zig 0.16.0](https://ziglang.org/download/).

```sh
git clone https://github.com/agent-rt/secretctl.git
cd secretctl
zig build -Doptimize=ReleaseSafe
sudo cp zig-out/bin/secretctl /usr/local/bin/
```

The release-safe binary is ~600 KB. There are no runtime dependencies — `Security.framework`, `CoreFoundation`, and `libc` are part of macOS itself.

## Verify

```sh
secretctl --version
secretctl --help
```

The first time you run `secretctl init`, the directory `~/.secretctl/` is created with mode `0700` and three files:

- `vault` — encrypted secrets (mode `0600`)
- `master.key` — KDF parameters and protector container (mode `0600`)
- `config.toml` — non-sensitive config (mode `0600`)

Audit log lives at `~/Library/Logs/secretctl.log`.

## Uninstall

```sh
brew uninstall secretctl                # if installed via Homebrew
sudo rm /usr/local/bin/secretctl        # if installed via tarball

# Optional: drop the vault, master key, and audit log
rm -rf ~/.secretctl
rm -f ~/Library/Logs/secretctl.log
```

If you used the macOS Keychain protector and want to wipe the wrapping key from the keychain too:

```sh
security delete-generic-password -s secretctl     # repeat per init'd vault
```
