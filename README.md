# GitHub MultiPAT — direnv + OS credential store pattern

Canonical reference for the **direnv + OS-managed credential store** pattern that exports a per-repo `GH_TOKEN` for the `gh` CLI without ever putting a personal access token (PAT) into a tracked file or shell history.

This repository documents the pattern and ships a generic `.envrc.example` template. Each consuming repo vendors its own concrete `.envrc.example` (the keychain entry name is the only thing that varies) and points back here for full setup, rotation, and non-macOS guidance.

## Why this exists

`gh auth login` stores PATs in a system keyring under a single global identity. That works for one user with one account, but breaks down when:

- An engineer has more than one PAT (e.g. an org-scoped fine-grained PAT plus a personal PAT) and needs `gh` to pick the right one **per-repo**, not globally.
- A PAT must rotate (e.g. quarterly fine-grained PAT expiry) and the rotation must not require re-running `gh auth login` in each clone.
- A token must never appear in shell history, transcripts, process lists, or `git log -p` of any tracked file.
- Multiple machines need consistent token sourcing without copy-pasting secrets.

The pattern in this repo solves all four:

- Tokens live **only** in the OS-managed credential store (macOS Keychain, Linux `secret-tool`, Windows Credential Manager, 1Password CLI, etc.).
- `direnv` reads the token from the store at shell-load time and exports it as `GH_TOKEN`. The `gh` CLI prefers `GH_TOKEN` over its own keyring entry, so per-repo `.envrc` files give per-repo identity automatically.
- Rotation is a single `security add-generic-password ... -W` (or vault-equivalent) — no code change, no commit, no redeploy.
- The `.envrc` file itself is gitignored. The committed `.envrc.example` only references the keychain entry **name**, never the token value.

## Repository layout

| File | Purpose |
|---|---|
| `.envrc.example` | Generic placeholder template (`<KEYCHAIN_ENTRY>`). Copy and substitute when adopting in a new repo, or extending to a different vault. |
| `.gitignore` | Standard exclusions: `.envrc`, `.direnv/`, `.DS_Store`. `.envrc.example*` is allow-listed. |
| `AGENTS.md` | Contract for AI agents working in this repo. |
| `README.md` | This file. |

## Setup (macOS)

One-time per machine:

```bash
brew install direnv
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc   # or ~/.bashrc
gh auth setup-git                              # so `git push` over HTTPS uses GH_TOKEN via gh
```

For each scope (one keychain entry per PAT, e.g. one for org-scoped, one for personal):

```bash
security add-generic-password -s 'my_pat_name' -a "$USER" -W
# `-W` reads the PAT from a TTY prompt — never enters shell history, transcripts, or process lists.
```

For each consuming repo:

```bash
cd <repo>
cp .envrc.example .envrc       # then substitute <KEYCHAIN_ENTRY> with the keychain name
direnv allow .
gh auth status                 # should report `Logged in to github.com (GH_TOKEN)`
```

## Required PAT scopes (fine-grained)

When generating a fine-grained PAT for this pattern, grant **at least** the following repository permissions. Fine-grained PATs decouple the PAT's scope from your underlying repo permissions: even a repo admin gets HTTP 403 from any API endpoint whose scope wasn't explicitly granted at PAT-creation time. Missing scopes are silent at `git push` time but bite later when `gh` tries to query the affected endpoint.

| Permission (UI label) | Access | Why |
|---|---|---|
| Contents | Read & Write | `git push`, `git clone` of private repos. Without this, nothing works. |
| Pull requests | Read & Write | `gh pr create`, `gh pr merge`, `gh pr view`. |
| Metadata | Read | Mandatory for any fine-grained PAT (auto-granted). |
| Actions | Read & Write | Two uses bundled: (1) `git push` of any commit touching `.github/workflows/*.yml` is rejected without it (`refusing to allow a Personal Access Token to create or update workflow ...`), even when Contents is granted; (2) `gh run list` and `gh run view` for CI history — the practical substitute for `gh pr checks`, see the gap below. |
| Workflows | Read & Write | Same underlying scope as Actions on some GitHub UI screens; older screenshots show "Workflows" instead. Grant whichever the UI presents. |
| Issues | Read & Write | Optional. Needed for `gh issue create`, automated triage, etc. |

**Commit statuses is deliberately not on this list.** An earlier version of the table carried a
`Commit statuses: Read` row for the legacy Status API (`/repos/.../commits/.../statuses`). The scope was
removed on 2026-09-01 rather than granted, and its absence is a decision rather than an oversight. Probed
twice on separate occasions against a private repo, `GET /repos/{owner}/{repo}/commits/HEAD/statuses`
returns **HTTP 403** with the response header `x-accepted-github-permissions: statuses=read`. The service
names the missing scope itself, so this is not inferred from a bare failure, and a deliberately malformed
token returned 401 in the same run, so the probe does discriminate. It stays ungranted because nothing
using this pattern calls the legacy Status API, and `gh run list` under `Actions: Read` already covers CI
history. **Grant it only if you adopt a legacy CI integration that writes commit statuses instead of
check-runs**, and correct this note in the same change. Do not re-add the row as an obvious omission.

### Gap: the Checks API is not callable from fine-grained PATs

GitHub explicitly documents this as a known limitation of fine-grained PATs. From [Managing your personal access tokens — Fine-grained personal access tokens limitations](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens):

> The major gaps in fine-grained personal access tokens are:
> - ...
> - **Using fine-grained personal access token to call the Checks API.**
> - ...

Consequence: `gh pr checks` and `gh pr view --json statusCheckRollup` (both backed by `/repos/.../commits/.../check-runs`) return HTTP 403 with `Resource not accessible by personal access token` and the header `x-accepted-github-permissions: checks=read`. The header names an internal permission that the current PAT creation UI does not expose as a selectable scope. **Older PATs created before this limitation took effect may show `Checks` granted (grandfathered from a prior UI), but the option is not selectable on new tokens.**

**Workaround that works with what fine-grained PATs do support today:**

```bash
# Instead of: gh pr checks NN
gh run list --branch <branch> --limit 20 \
  --json status,conclusion,name,headSha,databaseId

# Instead of: gh pr view NN --json statusCheckRollup
gh run view <run-id> --json jobs \
  --jq '.jobs[] | {name, conclusion}'

# Failed-step log for a run
gh run view <run-id> --log-failed
```

Both work with `Actions: Read` (which the table above already includes). They return per-workflow conclusions, which is usually what you actually want when checking "did CI pass". This same workaround is also documented inline at the consumer-repo level — see `Kacific/Kacific-HR-Process` `AGENTS.md` § "Tooling quirks" for an example.

For tooling that absolutely must call the check-runs API directly: use a classic PAT with `repo` scope, or a GitHub App. Reserve that fallback for the one piece of tooling that needs it; keep fine-grained PATs for everything else.

For **org-scoped** PATs (targeting repos under an organisation), the org owner must additionally approve the new fine-grained PAT before it can read private org repos. The approval step is org-side, not user-side; check your org admin if the PAT seems to work on public repos but fails on private ones.

## Rotation (macOS)

When a PAT expires or you want to roll it:

```bash
security delete-generic-password -s 'my_pat_name'
security add-generic-password    -s 'my_pat_name' -a "$USER" -W
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

When adopting in a new repo on a non-macOS machine, copy `.envrc.example` to `.envrc` and substitute the read command on the export line with the vault-equivalent. The pattern (read at shell-load, export as `GH_TOKEN`, gitignored) stays the same.

## Adopting in a new consuming repo

In the consuming repo:

1. Copy this repo's `.envrc.example` to your repo's `.envrc.example`.
2. Replace `<KEYCHAIN_ENTRY>` with your concrete keychain entry name (e.g. `my_org_pat`).
3. Slim the comment block to a header pointer back to this canonical repo (no need to duplicate the full setup / rotation / non-mac docs in every consumer). Suggested template:

   ```
   # .envrc.example — direnv + OS Keychain GH_TOKEN.
   # Standard: https://github.com/<owner>/<this-repo>
   # See the canonical README for full setup, rotation, and non-macOS variants.
   #
   # Keychain entry: my_org_pat
   #
   # Setup (macOS):
   #   security add-generic-password -s 'my_org_pat' -a "$USER" -W
   #   cp .envrc.example .envrc && direnv allow .

   _gh_tok="$(security find-generic-password -s 'my_org_pat' -a "$USER" -w 2>/dev/null)"
   if [ -n "$_gh_tok" ]; then
     export GH_TOKEN="$_gh_tok"
     export GITHUB_TOKEN="$_gh_tok"
   fi
   unset _gh_tok
   ```

   The guard (`if [ -n "$_gh_tok" ]`) ensures a missing or wrong Keychain entry leaves `GH_TOKEN` unset rather than setting it to an empty string. `gh` treats both the same way (reports "not logged in"), but other tools that distinguish `[ -z "$x" ]` from "unset" can behave inconsistently when the var is exported empty. See the rationale block at the bottom of `.envrc.example`.

4. Add `.envrc` (and optionally `.direnv/`) to your repo's `.gitignore`. Allow-list `.envrc.example` if your gitignore uses globs that would catch it.

## License

[MIT](./LICENSE).
