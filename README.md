# Kacific GitHub MultiPAT

Canonical reference for the **direnv + OS-managed credential store** pattern that exports a per-repo `GH_TOKEN` for the `gh` CLI without ever putting a personal access token (PAT) into a tracked file or shell history.

This repository documents the pattern, ships a generic `.envrc.example` template, and provides concrete pre-filled variants for the two scopes currently in use: Kacific org repos and personal (geekazoid80) repos.

## Why this exists

`gh auth login` stores PATs in a system keyring under a single global identity. That works for one user with one account, but breaks down when:

- An engineer has both an org-scoped fine-grained PAT (Kacific) and a personal PAT (geekazoid80), and needs `gh` to pick the right one **per-repo**, not globally.
- A PAT must rotate (e.g. quarterly fine-grained PAT expiry) and the rotation must not require re-running `gh auth login` in each clone.
- A token must never appear in shell history, transcripts, process lists, or `git log -p` of any tracked file.
- Multiple machines need consistent token sourcing without copy-pasting secrets.

The pattern in this repo solves all four:

- Tokens live **only** in the OS-managed credential store (macOS Keychain, Linux `secret-tool`, Windows Credential Manager, 1Password CLI, etc.).
- `direnv` reads the token from the store at shell-load time and exports it as `GH_TOKEN`. The `gh` CLI prefers `GH_TOKEN` over its own keyring entry, so per-repo `.envrc` files give you per-repo identity automatically.
- Rotation is a single `security add-generic-password ... -W` (or vault-equivalent) — no code change, no commit, no redeploy.
- The `.envrc` file itself is gitignored. The committed `.envrc.example` only references the keychain entry **name**, never the token value.

## Repository layout

| File | Purpose |
|---|---|
| `.envrc.example` | Generic placeholder template (`<KEYCHAIN_ENTRY>`). Copy and substitute when introducing a new scope or non-mac vault. |
| `.envrc.example.kacific` | Concrete variant for Kacific-org repos. Maps `GH_TOKEN` → keychain entry `gh_kacific_pat`. |
| `.envrc.example.personal` | Concrete variant for personal (geekazoid80) repos. Maps `GH_TOKEN` → `gh_personal_pat`. |
| `.gitignore` | Standard exclusions: `.envrc`, `.direnv/`, `.DS_Store`. `.envrc.example*` is allow-listed. |
| `AGENTS.md` | Contract for AI agents working in this repo. |

## Setup (macOS)

One-time per machine:

```bash
brew install direnv
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc   # or ~/.bashrc
```

For each scope (Kacific PAT, personal PAT, etc.):

```bash
security add-generic-password -s 'gh_kacific_pat' -a "$USER" -W
# `-W` reads the PAT from a TTY prompt — never enters shell history, transcripts, or process lists.
```

For each consuming repo:

```bash
cd <repo>
cp .envrc.example .envrc       # or copy .envrc.example.kacific / .envrc.example.personal
direnv allow .
gh auth status                 # should report `Logged in to github.com (GH_TOKEN)`
```

## Rotation (macOS)

When a PAT expires or you want to roll it:

```bash
security delete-generic-password -s 'gh_kacific_pat'
security add-generic-password    -s 'gh_kacific_pat' -a "$USER" -W
# Same `-W` as above. Open a new shell or run `direnv reload` for the change to take effect.
```

No commit. No push. No `.envrc` change.

## Extending to non-macOS / non-Keychain vaults

The template assumes macOS `security` CLI, but the pattern is generic: any command that prints a secret to stdout works. The full `.envrc.example` includes commented examples; the table below summarises:

| OS / Vault | Read command (replaces `security find-generic-password`) | Write command (replaces `security add-generic-password`) |
|---|---|---|
| **macOS Keychain** | `security find-generic-password -s NAME -w` | `security add-generic-password -s NAME -a "$USER" -W` |
| **Linux** (libsecret / GNOME Keyring) | `secret-tool lookup service NAME account "$USER"` | `secret-tool store --label='NAME' service NAME account "$USER"` |
| **Linux** (`pass`) | `pass NAME` | `pass insert NAME` |
| **Windows** (PowerShell + Credential Manager) | `(Get-StoredCredential -Target NAME).Password \| ConvertFrom-SecureString -AsPlainText` | `New-StoredCredential -Target NAME -UserName "$env:USERNAME" -Password (Read-Host -AsSecureString)` |
| **1Password CLI** (cross-platform) | `op read "op://Private/<vault-item>/credential"` | `op item edit ...` |
| **Bitwarden CLI** (cross-platform) | `bw get password NAME` (after `bw unlock`) | `bw create item ...` |

When introducing a new vault, copy `.envrc.example` to `.envrc.example.<vault>` and substitute the read/write commands. Keep the canonical `.envrc.example` as the cross-vault placeholder reference.

## Current consumers

Repositories using this pattern as of the most recent rollout:

**Kacific org** (use `gh_kacific_pat`):
- `Kacific/Kacific-Brand-Guide`
- `Kacific/Kacific-HR-Process`
- `Kacific/Kacific-NUC-Cron`
- `Kacific/Kacific-RozeeGPT-Adapter`

**Personal — geekazoid80** (use `gh_personal_pat`):
- `geekazoid80/Living-Networked-Compendium`
- `geekazoid80/claude-infrabot`

Each consumer ships a slim per-repo `.envrc.example` that references this canonical repo for the full setup / rotation / extensibility documentation. The export line in their `.envrc.example` matches the canonical pattern, parameterised by the appropriate keychain entry.

## License

Internal Kacific reference. Visibility is private. Make public only after scrubbing the `.envrc.example.kacific` variant if open-sourcing the pattern externally.
