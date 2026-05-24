# CLAUDE.md — Contract for humans and AI agents

## What this repo is

A documentation-only canonical reference for the direnv + OS-credential-store GitHub PAT pattern. There is no application code, no CI logic, no runtime here. Just a generic template and prose.

## Hard rules — never violated

1. **No real PAT is ever committed.** The `.envrc.example` references a keychain entry **name**, never a token value. If a PR adds a 40-character GitHub token literal, reject it.
2. **`.envrc` itself is never committed.** The repo's `.gitignore` enforces this. `.envrc.example*` is allow-listed.
3. **The generic template stays generic.** `.envrc.example` keeps `<KEYCHAIN_ENTRY>` as a placeholder. Concrete entry names belong in **consumer** repos' own `.envrc.example`, not here.
4. **The README is the single source of truth** for the pattern. Setup, rotation, and non-mac extensibility live there. Consumer repos point at this README rather than re-document.
5. **No automation.** This repo doesn't run anything. No CI workflows, no scripts, no shell helpers (yet). If a setup helper is added later, it must live in a `scripts/` directory and the README must explicitly opt-in to it.

## Adding a new vault example to the README

To add support for a new vault (e.g. KeePassXC CLI, AWS Secrets Manager, HashiCorp Vault):

1. Add a row to the README's "Extending to non-macOS / non-Keychain vaults" table.
2. Add a commented example block to `.envrc.example` showing the read-line replacement.
3. Open a PR. Squash-merge.

Adding a new vault here does **not** add a `.envrc.example.<vault>` file — the canonical only ships the generic placeholder. Consumer repos that adopt the new vault will copy the relevant snippet from the README into their own `.envrc.example`.

## What NOT to do here

- Don't add application code, build tooling, or test harnesses.
- Don't ship organisation-specific or repo-specific concrete `.envrc.example` variants. Those live in their respective consumer repos.
- Don't mention specific consumer repos by name in this canonical README. The pattern is generic; the consumer list is private to whoever runs it.
- Don't write a setup wizard / shell-installer until at least three independent operators ask for it. The current pattern is two commands (`security add-generic-password ... -W` then `direnv allow .`); a wizard adds dependency for marginal ergonomic gain.

## When uncertain, link rather than duplicate

If you find yourself about to copy a paragraph from this repo's README into a consumer repo's documentation, stop. Add a one-line link to this repo instead. Duplication leads to drift; links don't.
