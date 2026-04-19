---
title: Install · Synap
---

# Install

## Platforms

0.1.x ships **macOS arm64 (Apple Silicon) only**. x86_64 / Linux / Windows
are not supported in this preview.

## Homebrew (recommended)

```sh
brew tap agent-rt/tap
brew install synap
```

Upgrades:

```sh
brew update && brew upgrade synap
```

## One-liner

```sh
curl -fsSL https://raw.githubusercontent.com/agent-rt/homebrew-tap/main/install.sh | sh
```

Installs the latest release to `/usr/local/bin/synap`. `sudo` may be
prompted.

Custom directory:

```sh
SYNAP_INSTALL_DIR="$HOME/.local/bin" \
  curl -fsSL https://raw.githubusercontent.com/agent-rt/homebrew-tap/main/install.sh | sh
```

Pin a version:

```sh
SYNAP_VERSION=v0.1.1 \
  curl -fsSL https://raw.githubusercontent.com/agent-rt/homebrew-tap/main/install.sh | sh
```

## Manual

1. Browse <https://github.com/agent-rt/homebrew-tap/releases> and pick a
   `synap-v<version>` tag.
2. Download `synap-v<version>-aarch64-apple-darwin.tar.gz`.
3. Extract + install:
   ```sh
   tar -xzf synap-*-aarch64-apple-darwin.tar.gz
   cd synap-*-aarch64-apple-darwin
   sudo mv synap /usr/local/bin/
   ```
4. First run downloads the Qwen3 embedding model (~600 MB) from Hugging
   Face into `~/.cache/huggingface/`. Subsequent runs are instant.

## Verify

```sh
synap --version
synap daemon status
```

`daemon status` should print either "running" or "not running".

## Uninstall

Homebrew:

```sh
brew uninstall synap
brew untap agent-rt/tap   # optional
```

Manual:

```sh
synap daemon stop
rm -f /usr/local/bin/synap
rm -rf ~/.synap/   # ⚠️ this erases all stored memory
```

The model cache under `~/.cache/huggingface/` is reusable — only delete it
if you're done with all embedding-model tooling.
