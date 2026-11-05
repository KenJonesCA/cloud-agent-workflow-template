# AGENTS.md — Agent Conventions for This Repository

<!--
TEMPLATE FILE. Sections marked "PROJECT-SPECIFIC" are placeholders: replace them
with your own rules, or delete them if they don't apply. Everything else is the
generic cloud-agent pull-request workflow and is meant to be kept as-is unless
you have a reason to change it. See docs/design-rationale.md for why each rule exists.
-->

This file is the single source of truth for how coding agents work in this repository. It is named `AGENTS.md` after the cross-tool convention, so every agent and harness reads the same rules from one place. `CLAUDE.md` is only a pointer for harnesses that don't yet read `AGENTS.md`; it holds no rules of its own.

Read this file at the start of every session.

## About this repository

> **PROJECT-SPECIFIC — replace this block.** One or two paragraphs: what the repository holds, who owns it, and what a good change looks like here. If there is a strong opinion about what does *not* belong in the repo, say so here.

## Data boundary

> **PROJECT-SPECIFIC — replace or delete this block.** State what must never be committed, and what is fine to commit. A useful default stance: treat the repository as one leaked token away from public, even if it is private.

Never commit credentials, API keys, private keys, tokens, or personal identifiers. If you come across one while working, do not copy it anywhere; refer to the thing it identifies by a neutral nickname instead.

If this repository has a gitignored local-only directory (for raw exports, fixtures with real data, or other material that must not leave the machine), name it here: `<local-only-dir>/`. It is never committed, and anything taken from it into a tracked file has identifiers removed first. It does not exist in cloud sessions.

## Where you are running decides how you write

There are two write paths. Work out which one you are on before changing anything.

| | Local session | Cloud session |
|---|---|---|
| Sees | The owner's working tree, including uncommitted and gitignored files | A fresh clone of the GitHub repository, as of the last push |
| Writes to | The working tree directly | A `claude/<short-topic>` branch, delivered as a pull request |
| Commits | Only when the owner asks | Yes — commit and push to your branch |
| Merges | Never | Never |

You are in a **cloud session** if the working directory is a fresh clone in an ephemeral container, the owner's local-only files are absent, or your harness says so. When unsure, assume cloud and use the pull-request path; it is the safe one.

### Cloud session rules

- **Work on a new branch named `claude/<short-topic>`.** Lowercase, kebab-case, describing the change (`claude/fix-broken-links`, not `claude/update`). Never commit to `main` directly.
- **One pull request per change.** A change is one logical unit: one fix, one feature, or one tightly related set of files. Unrelated changes go in separate PRs, even if you made them in the same session.
- **Never merge your own pull request**, and never approve it. A human reviews and merges. Do not enable auto-merge.
- **Run the checks before opening the PR, and fix what they report:** `pre-commit run --all-files`. The same checks run in CI on every pull request to `main`; a PR that fails them locally will fail there too. Say in the PR description that you ran them and what they reported.
- **The PR description says what changed and why.** Fill in the pull request template; don't leave its sections empty or delete them.
- **The clone is only as current as the last push.** Something missing from your clone may simply not have been pushed yet. Check `git log -1` and the date of the last commit before concluding a file is missing, a feature doesn't exist, or something needs to be recreated. If it looks like the owner's local copy is ahead, say so in the PR rather than guessing.
- **Don't rename, move, or delete existing files unless the task asks for it.** Renames break inbound references and make review harder. If one seems necessary, propose it in the PR description instead of doing it.
- **The write zones below still apply.** A cloud session has no extra permissions.
- **Anything needing local-only material runs locally.** If the task needs files that only exist on the owner's machine, stop and say so; don't fabricate or approximate them.

### Local session rules

- Edit the working tree directly. Don't commit, push, or create branches unless the owner asks.
- If the owner asks you to commit, the same branch rules apply when the change will go through a PR.
- The write zones below apply here too.

## Write zones

Which parts of the repository an agent may change, and how freely.

> **PROJECT-SPECIFIC — replace the example rows with your own layout.** Keep the table shape: one row per directory or file pattern, and a plain-English permission. Delete rows you don't need.

| Zone | Permission |
|------|-----------|
| `<drafts-dir>/` | Free write. Low-stakes additions; no special review needed beyond the PR. |
| `<source-dir>/` | Write when the task calls for it. Changes are reviewed in the PR. |
| `docs/` | Write when the task calls for it, or when your change makes existing docs wrong. |
| `<generated-dir>/` | Written by tooling only. Don't hand-edit; regenerate instead. |
| `<tool-config-dir>/` | Editor or tool configuration. Don't touch unless asked. |
| `<local-only-dir>/` | Gitignored. Read only when the owner allows it; never commit. |
| `.github/`, `.pre-commit-config.yaml`, `AGENTS.md` | Repository machinery. Change deliberately, only when asked, and explain why in the PR. |

When in doubt, write to the most permissive zone that fits, or ask.

## Checks

The checks are defined in `.pre-commit-config.yaml` and enforced by `.github/workflows/pre-commit.yml`.

```bash
pre-commit run --all-files                  # everything
pre-commit run <hook-id> --files <path>     # one hook, one file
```

If a hook modifies files (for example, fixing trailing whitespace), review the change, stage it, and run the checks again until they pass. Never skip, disable, or weaken a check to get a PR green; if a check is wrong, say so in the PR and let a human decide.

> **PROJECT-SPECIFIC — add your build, test, and lint commands here** if they are not already pre-commit hooks, and say which ones must pass before a PR is opened.

## Project-specific conventions

> **PROJECT-SPECIFIC — replace this block.** Style rules, naming conventions, file layout, commit-message format, anything an agent would otherwise guess at. Rules that are easy to forget under pressure belong near the top of this section. Prefer short, checkable rules over aspirations.

## Session protocol

**Start:** read this file. Decide whether you are in a local or cloud session. In a cloud session, check `git log -1` so you know how current the clone is.

**End (cloud):** checks pass, work is committed and pushed to your `claude/` branch, one PR is open with the template filled in, and you have not merged it.

**End (local):** summarize what changed in the working tree; don't commit unless asked.

## Rules (quick reference)

- Cloud: `claude/<short-topic>` branch, never commit to `main`.
- One PR per change; the description says what and why.
- Never merge or approve your own PR.
- Run `pre-commit run --all-files` before opening a PR; fix what it reports.
- The clone is only as current as the last push — check `git log -1` before assuming something is missing.
- Respect the write zones, in both cloud and local sessions.
- Don't rename, move, or delete existing files unless asked.
- Never commit credentials or identifiers.
