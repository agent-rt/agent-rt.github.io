---
title: Nix · secretctl
---

# nix-darwin / Home Manager

`secretctl` ships a Home Manager module that turns vault entries into
files on disk at `home-manager switch` time — same shape as `sops-nix`,
no GPG / age / sops toolchain in your PATH.

## Add to your flake

```nix
{
  inputs.secretctl.url = "github:agent-rt/secretctl";
  # ...

  outputs = { self, nixpkgs, home-manager, secretctl, ... }: {
    homeConfigurations."you@host" = home-manager.lib.homeManagerConfiguration {
      pkgs = import nixpkgs {
        system = "aarch64-darwin";
        overlays = [ secretctl.overlays.default ];   # gives `pkgs.secretctl`
      };
      modules = [
        secretctl.homeManagerModules.default
        ./home.nix
      ];
    };
  };
}
```

## Declare secrets

```nix
# home.nix
{ ... }: {
  programs.secretctl = {
    enable = true;

    # Token-style secrets — get materialized to per-var files (0600) and a
    # generated env.sh that your shell can `source`.
    envSecrets = {
      GITHUB_TOKEN = {};
      OPENAI_API_KEY = {};
      # Different vault key from env-var name:
      GH_PACKAGES_TOKEN = { secretName = "github_packages_token"; };
    };

    # File-style secrets — written to specific paths with custom modes.
    fileSecrets = {
      "SSH_KEY_WORK" = { path = ".ssh/work"; mode = "0600"; };
      "SSH_KEY_HOME"      = { path = ".ssh/home";      mode = "0600"; };
    };
  };

  programs.bash.initExtra = ''
    [ -r ~/.config/secretctl/env.sh ] && source ~/.config/secretctl/env.sh
  '';
  # or fish:
  # programs.fish.interactiveShellInit = "source ~/.config/secretctl/env.sh";
}
```

After `home-manager switch`, `~/.ssh/work` and friends are populated
straight from the vault, and the env-style secrets are exported every
time you open a shell.

## Migrating from sops-nix

The mapping is one-to-one for the common patterns:

```nix
# Before — sops-nix
sops = {
  age.keyFile = "${homeDir}/.config/sops/age/keys.txt";
  defaultSopsFile = ../../secrets/work.yaml;
  secrets.github_token = {};
  secrets.github_publish_token = {};
  secrets."SSH_KEY_WORK" = {
    path = "${homeDir}/.ssh/work";
    mode = "0600";
  };
};
```

```nix
# After — secretctl
programs.secretctl = {
  enable = true;
  envSecrets = {
    GITHUB_TOKEN         = { secretName = "github_token"; };
    GITHUB_PUBLISH_TOKEN = { secretName = "github_publish_token"; };
  };
  fileSecrets = {
    "SSH_KEY_WORK" = { path = ".ssh/work"; mode = "0600"; };
  };
};
```

### One-shot import script

```bash
#!/usr/bin/env bash
# Decrypt your sops file once, walk the keys into secretctl, then
# delete the temp dump.
set -euo pipefail

sops -d ~/dotfiles/secrets/work.yaml > /tmp/sops-dump.yaml
trap 'rm -f /tmp/sops-dump.yaml' EXIT

secretctl init   # if not already

# Token-style — single-line values
for key in github_token github_publish_token github_homebrew_tap_token; do
  value=$(yq -r ".${key}" /tmp/sops-dump.yaml)
  printf "%s\n%s\n" "$MASTER_PASSWORD" "$value" \
    | SECRETCTL_BATCH=1 secretctl add "$key" --tag git
done

# Multi-line — SSH keys
for name in work msn; do
  yq -r ".ssh_keys.${name}" /tmp/sops-dump.yaml > /tmp/ssh-key
  EDITOR="cp /tmp/ssh-key" secretctl add "ssh_keys/${name}" --tag ssh --editor
  rm /tmp/ssh-key
done
```

(The `EDITOR=` trick passes the prepared key file straight through `add
--editor` without you having to paste anything.)

## Activation prerequisites

`home-manager switch` runs after you log in to your macOS GUI session,
which means the login keychain is already unlocked. The default
keychain protector resolves silently in that state — no Touch ID prompt,
no password prompt — and `secretctl materialize` succeeds for each
declared secret.

If the activation runs before login (rare on workstations) the keychain
unwrap will fail and the activation script logs a warning per secret.
The system stays consistent; rerun `home-manager switch` once you're
logged in.

## Why this is simpler than sops-nix

| | sops-nix | secretctl |
|---|---|---|
| External tools to install | sops + age | none beyond `secretctl` |
| Per-host age keys to manage | yes | no — one Mac Keychain protector |
| Decryption point | activation script | activation script (same shape) |
| Secret rotation | re-encrypt YAML, commit | `secretctl edit` — one command |
| Adding a secret | edit `.sops.yaml`, re-encrypt YAML, declare in nix | `secretctl add` + nix entry |
| Browsing what's stored | `sops secrets/work.yaml` reveals values | `secretctl list` (no values) |

If you mainly use sops-nix for the activation/materialize pattern, the
move is a clean swap. Multi-host sharing (`age` recipients across Macs)
is on the Phase 5 roadmap; until then keep using sops-nix when you need
the same encrypted file readable on multiple hosts.
