# CLAUDE.md — Contract for humans and AI agents

<!-- BEGIN kacific:concurrency-coordination (managed, do not edit by hand) -->
## Concurrency and git coordination

This repo may be worked by more than one agent or person at once. Assume a peer may be editing shared
files or moving branches at any moment. The rules below exist because a shared checkout was seen to
shuffle a branch ref mid-commit.

- **Worktree-per-session (mandatory).** Never edit in a shared main checkout. Create your own linked
  worktree **under the repo**, at a gitignored path, and work there:
  ```
  git -C <repo> fetch --prune origin main
  git -C <repo> worktree add .claude/worktrees/<task> -b <area>/<task> origin/main
  # edit in that worktree, commit, push, open the PR, then:
  git -C <repo> worktree remove .claude/worktrees/<task>
  ```
  A worktree has its own HEAD and index, so a peer's checkout cannot move your branch under you.
  **Place it under the repo, never beside it as `../wt-<task>`.** A sibling inherits the PARENT
  directory's treatment rather than the repo's, so it silently escapes every protection scoped to the
  repo path at once: backup roots and sync excludes that name the repo, ignore rules, and anything else
  keyed on that path. This is not hypothetical, it cost a live incident. Under the repo it inherits all
  of them, and being gitignored keeps it out of the index. Full reasoning in the `using-git-worktrees`
  and `multi-agent-repo-coordination` vault skills.
  **This repo's tracked `.gitignore` must COVER `.claude/worktrees/`, which is not the same as carrying
  that literal line.** A local `.git/info/exclude` entry, or a line in your global ignore file, is not
  enough: neither travels, so on a fresh clone the worktree shows as untracked and a nested working copy
  can be committed by accident. The global-ignore case is the worse of the two, because it fails toward
  compliant. **Coverage is the test, and the way to ask is
  `git -c core.excludesFile=/dev/null check-ignore -q .claude/worktrees/`**, where the pin is what makes
  the answer a property of the repo rather than of the machine you happen to be on. A repo that ignores
  `.claude/` wholesale, or whose `.gitignore` is a whitelist, already covers the path and needs nothing
  added. Add the rule only where that probe says the path is not covered.
- **Pull before dev.** `fetch --prune` then `pull --ff-only` (or branch straight off `origin/main`)
  before the first edit. Fast-forward only, never force. If `--ff-only` refuses (diverged) or a dirty
  tree would conflict, stop and surface it.
- **Branch per task off `main`; commit immediately; verify the pushed ref.** After pushing, confirm
  `git rev-parse --short origin/<branch>` equals your commit before relying on it. If a commit lands on
  the wrong branch (ref shuffle), recover by pushing the SHA explicitly:
  `git push origin <sha>:refs/heads/<branch>`.
- **Append, do not rewrite** shared docs where a peer may be mid-edit. Prefer small targeted edits over
  wholesale rewrites; on a conflict, reconcile rather than clobber.
- **Clone, do not assume.** If this repo, or a referenced repo, is not present locally, clone it from
  the `Kacific` GitHub org and keep it synced. Do not assume a stale local copy is current.
- **Canonical working copy.** Edit in your dedicated checkout. An incidental copy produced by an
  all-org-repos clone (for example a `~/Documents/Programming/<repo>` mirror) is **read-only**; do not
  edit it, it drifts.
<!-- END kacific:concurrency-coordination -->

<!-- BEGIN kacific:agent-practices (managed, do not edit by hand) -->
## Mandatory practices for agents

These apply to every session in this repo, on top of the concurrency rules above. They are the estate
default, not optional, and hold even where a task prompt does not restate them.

- **Use the skills vault.** Before non-trivial work, check which `~/.claude/skills/` vault skills fire
  for the task (per `plan-time-tooling`) and use them; do not re-derive from memory what a skill already
  encodes. If a relevant skill is missing, propose one (`author-skill`) rather than working around it.
- **Run the boundary-check at every boundary.** Invoke the `boundary-check` skill before proposing a
  compact, before a park / standdown / restart, and at every chunk close or shift (before offering the
  next chunk). It does the fresh-disk re-read of standing instructions and memory, reconciles the
  session, and emits the visible stamp.
- **Plan at every kickoff.** Enter plan mode at the start of each task or chunk (per
  `reread-memory-before-planning`): re-read memory and this `AGENTS.md` from disk, enumerate the tooling
  that fires, and surface scope decisions before acting. Plan mode is the default cadence; action is the
  exception that needs alignment first.
- **Land changes by PR, and merge your own.** Branch off the default branch, push, open the PR,
  squash-merge it yourself, delete the branch. **No review is required and none should be waited for**,
  unless this repo's own `AGENTS.md` says otherwise below, which overrides this block. Stated because it
  was nowhere written down, which is not the same as unsettled: measured across the org on 2026-09-07,
  **236 of 249** recent merged PRs were merged by their own author with zero reviews. The gap cut both
  ways, since a session can read a PR requirement and infer that a reviewer is coming, which parks the
  work indefinitely, or conclude the PR is skippable ceremony when it is the record of why a change was
  made. Only the GitHub PR API can establish any of this: every squash-merge records GitHub's bot as the
  committer, so `git log` can never distinguish a self-merge from a reviewed one.
<!-- END kacific:agent-practices -->

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
