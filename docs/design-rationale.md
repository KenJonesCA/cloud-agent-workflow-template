# Design rationale

Why each rule in [AGENTS.md](../AGENTS.md) and each piece of repository machinery exists. Read AGENTS.md for what the rules *are*; this document is only about why, what each rule rejected, and when it would be worth revisiting.

If you change a rule, update the matching entry here in the same PR.

## The core idea: cloud agents deliver pull requests

A cloud agent works in an ephemeral container on a clone of the repository. Nothing it writes survives the container unless it is pushed. That leaves three ways to get its work back: attach files to a chat reply and copy them in by hand, have a local session redo the write, or push a branch and open a pull request.

The pull request wins because it turns every agent edit into a reviewable diff in the place you already review code, it works while your own machine is off, and it reuses machinery (CI, rulesets, code owners) that already exists. The cost is one more review queue and granting the agent access to the repository. *Revisit if* the review queue becomes the bottleneck and most PRs are merged unread — at that point the review gate is theatre and the workflow needs either smaller PRs or less agent autonomy.

## Two write paths

**Local and cloud sessions write differently because they see different things.** A local session sees the owner's working tree, including uncommitted and gitignored files, and the owner is present to see what it does; writing in place is natural. A cloud session sees only what has been pushed, and nobody is watching; the PR is its only safe output. Pretending the two are the same leads to cloud agents trying to read files that don't exist, or local agents committing and pushing when the owner only wanted an edit.

**When unsure, assume cloud.** A PR from a local session costs a review; a direct write from a cloud session goes nowhere or lands on `main`. The asymmetry favours the PR.

**Local sessions don't commit unless asked.** The owner may have their own uncommitted work in the tree, or their own commit rhythm. An agent commit can sweep that up.

## Branch naming: `claude/<short-topic>`

A fixed prefix makes agent branches easy to find, filter, clean up, and target with a ruleset. Some harnesses also only allow pushes to prefixed branches. The topic part should describe the change so the branch list is readable without opening each PR. *Rename the prefix* if you use a different agent, or several; the point is one predictable prefix per agent identity.

## One PR per change

Reviewers approve or reject a PR as a unit. If one PR mixes a good change with a doubtful one, the reviewer has to either accept both or send both back. Small, single-purpose PRs are also easier to revert. The rule says "one logical change", not "one file", because a fix plus its test, or a set of tightly related files, is one decision.

## Never merge your own PR

The review is the whole point of the workflow. An agent that merges its own PR has turned the PR into a slower direct push, with a misleading audit trail that suggests someone looked. This applies to approving and to enabling auto-merge as well, because both are ways of merging without a human decision.

This rule is also enforced by branch protection (see [setup.md](setup.md)) rather than trusted to instructions alone, because instructions get missed and the cost of one unreviewed merge to `main` is high.

## Run the checks before opening the PR

CI will run the same checks anyway, so this might look redundant. It isn't: a PR that fails CI costs a reviewer notification, a red X, and a round trip in which the agent's session may already have ended. Running the checks first makes "done" checkable from inside the session, and the agent can fix what it broke while it still has the context. The PR template asks the agent to say it ran them, which makes skipping visible.

**Never weaken a check to pass it.** A check that fails is either right (fix the change) or wrong (fix the check, deliberately, in its own PR). An agent quietly disabling a hook hides both cases.

## "The clone is only as current as the last push"

The clone is a snapshot of GitHub, and GitHub is only as current as the last time someone pushed. An owner who works locally and pushes occasionally will often have a local copy ahead of the remote. A cloud agent that concludes "this file doesn't exist, I'll create it" may be recreating something that exists one unpushed commit away, producing a duplicate or a conflict.

Checking `git log -1` is cheap and tells the agent how old its view is. Saying so in the PR, rather than guessing, lets the reviewer catch it. The same fact constrains scheduled or recurring cloud jobs: they only ever see what has been pushed, so they are only worth running once pushes are reliable.

## Don't rename, move, or delete without being asked

Renames break inbound references (links, imports, docs, bookmarks, other people's branches), and in many diffs a rename shows up as a delete plus an add, which hides any edit made in the same change. They are also the kind of change a reviewer is least likely to want from an agent unprompted. Proposing it in the PR description keeps the decision with a human.

## Write zones

A write zone table says, per directory, how freely an agent may change things. It replaces a long list of "don't touch X" rules with one place to look, and it applies equally to local and cloud sessions, because the risk of an inappropriate edit doesn't depend on where the agent runs.

The typical gradient:

- **Free-write zones** for low-stakes additions, so an agent always has somewhere safe to put something when it is unsure. "When in doubt, write to the most permissive zone that fits" exists so that uncertainty produces a draft, not a guess in the wrong place.
- **Normal zones** where changes are fine but reviewed.
- **Tool-owned zones** (generated output, editor configuration, files written by another tool) where hand edits are overwritten or cause churn.
- **Machinery** (CI, hook config, AGENTS.md itself), which changes how every future session behaves and so is changed deliberately.
- **Local-only material**, which never reaches the repository at all.

## Data boundary

**Treat the repository as one leaked token away from public.** Private repositories leak through tokens, forks, misconfigured apps, and future visibility changes. Granting a cloud agent access widens the set of things that can read the repository. Writing rules as if the repository were public means none of those events is a disaster.

**Keep sensitive raw material outside git, not just outside the rules.** A gitignored local-only directory is never cloned, so a cloud agent cannot read it no matter what its instructions say. A rule that says "don't read this" is weaker than the file not being there. The cost is that anything needing that material has to run locally, and that the directory needs its own backup, because git no longer covers it.

**Commit conclusions, not raw exports.** The reasoning and results built on sensitive data are usually fine to commit; the raw data with identifiers is not. This keeps the useful part reviewable without putting the sensitive part in history, where it is very hard to remove.

## AGENTS.md as the single source, CLAUDE.md as a pointer

Different harnesses look for different instruction files. Keeping the rules in one cross-tool file and making every other file a one-line pointer means the rules can't drift apart between tools. The pointer holds no rules of its own so there is never a question of which file wins. *Delete the pointer* once every harness in use reads `AGENTS.md` directly.

## CI on pull requests and on pushes to main

The pull-request trigger is the gate before merge. The push trigger exists because things reach `main` without a PR: an admin bypass, an automated sync, or a direct push before protection was set up. Without it, a broken file pushed straight to `main` would surface later as a failure on some unrelated PR, and the author of that PR would be left debugging someone else's change. The push run puts the red X on the commit that caused it. It reports; it cannot block. Blocking is the ruleset's job.

## Hooks: general-purpose first, pinned, enforced in CI

**General-purpose hooks only in the starter config.** Whitespace, line endings, parseable YAML/JSON/TOML, conflict markers, large files, case conflicts, and private keys are problems in any repository and cost nothing to check. Language linters are project decisions and are left as commented examples.

**`--enforce-all` on the large-file check.** Without it the hook only inspects newly staged files, which makes it a no-op under `pre-commit run --all-files` in CI — a check that silently does nothing is worse than no check.

**Pin to commit SHAs, not tags.** A tag can be moved; a SHA can't. Pinning hooks and actions means CI behaves the same tomorrow as today, and updates arrive as reviewable PRs.

**CI is the enforcement point; local hooks are optional.** Local hooks depend on each contributor (or agent) having run `pre-commit install`, and can be skipped with `--no-verify`. CI can't be skipped. Making CI authoritative means the local setup is a convenience, not a correctness requirement. Where local commits must never be blocked — a tool that auto-commits on a timer, for example — the hooks can move to the manual stage entirely; [setup.md](setup.md) covers that variant and its main trap (forgetting `--hook-stage manual` makes the run check nothing).

## GitHub App over personal token, installed on one repository

An App installation is scoped, visible in repository settings, revocable in one place, and gives agent work its own identity, which lets rulesets and reviewers treat it differently from a human. A personal token acts as you, inherits your bypasses, and expires on its own schedule. Installing on selected repositories rather than all of them keeps the exposure to what you actually decided to share; every repository an agent can reach is also one it reads, and in some harnesses one every session clones. Start with one repository as a trial and widen deliberately.

## What this template deliberately does not decide

- **Merge strategy** (squash, rebase, merge commit). Teams have strong existing preferences; nothing in the workflow depends on it.
- **Which agent harness.** The rules are harness-agnostic; only the branch prefix and the CLAUDE.md pointer name one.
- **Project-specific conventions.** They are placeholders because rules invented up front tend to describe how you imagine the work will go rather than how it does. Add a rule when you see the same correction made twice.
