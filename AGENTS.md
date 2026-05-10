# AGENTS.md — Contract for humans and AI agents

## What this repo is

A documentation-only canonical reference for the direnv + OS-credential-store GitHub PAT pattern used across Kacific repos and a small number of personal repos. There is no application code, no CI logic, no runtime here. Just templates and prose.

## Hard rules — never violated

1. **No real PAT is ever committed.** Every `.envrc.example*` file references a keychain entry **name**, never a token value. If a PR adds a 40-character GitHub token literal, reject it.
2. **`.envrc` itself is never committed.** The repo's `.gitignore` enforces this. `.envrc.example*` is allow-listed.
3. **Parameterise rather than hardcode** when adding a new variant. The generic `.envrc.example` keeps `<KEYCHAIN_ENTRY>` as a placeholder. Concrete variants (`.envrc.example.kacific`, `.envrc.example.personal`) substitute the real entry name.
4. **The README is the single source of truth** for the pattern. Setup, rotation, and non-mac extensibility live there. Consumer repos point at this README rather than re-document.
5. **No automation.** This repo doesn't run anything. No CI workflows, no scripts, no shell helpers (yet). If a setup helper is added, it must live in a `scripts/` directory and the README must explicitly opt-in to it.

## Adding a new variant

To add support for a new scope (e.g. a separate PAT for a contractor account) or a new vault (e.g. 1Password CLI):

1. Copy `.envrc.example` → `.envrc.example.<variant>` (for a new scope) or `.envrc.example.<vault>` (for a new vault).
2. Substitute the placeholder with the concrete keychain entry name (or replace the read command with the vault-equivalent).
3. Update the README's "Current consumers" or "Extending to non-macOS" table to mention the new variant.
4. Open a PR. Squash-merge.

## What NOT to do here

- Don't add application code, build tooling, or test harnesses.
- Don't vendor copies of consumers' real `.envrc.example` files. Those live in their respective repos and may differ in trivial ways (header comment style, etc.). The variants here are canonical examples, not vendor copies.
- Don't write a setup wizard / shell-installer until at least three consumers ask for it. The current pattern is two commands (`security add-generic-password ... -W` then `direnv allow .`); a wizard adds dependency for marginal ergonomic gain.

## When uncertain, link rather than duplicate

If you find yourself about to copy a paragraph from this repo's README into a consumer repo's documentation, stop. Add a one-line link to this repo instead. Duplication leads to drift; links don't.
