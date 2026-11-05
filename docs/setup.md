# Setup

How to wire a repository up for the cloud-agent pull-request workflow described in [AGENTS.md](../AGENTS.md). The reasoning behind each choice is in [design-rationale.md](design-rationale.md).

## 1. Give cloud agents access to the repository

A cloud agent needs to clone the repository, push a branch, and open a pull request. There are two ways to grant that.

### Option A: a GitHub App (recommended)

Install the agent vendor's GitHub App (for Claude, the Claude GitHub App) on the repository.

- **Install it on selected repositories only, not "All repositories".** Each repository you add is readable by every cloud session that can reach it. Start with one.
- Permissions are scoped to the installation, show up in the repository's **Settings → Integrations → GitHub Apps**, and can be revoked in one place.
- Commits and PRs are attributed to the app or to you-via-the-app, which makes agent work easy to tell apart in history and in rulesets.
- Some agent surfaces require the App specifically (for example, project-style threads that clone every attached repository). Check your vendor's docs.

### Option B: a personal access token

Some harnesses can instead use a token you provide (for example, by forwarding your local `gh` login).

- Prefer a **fine-grained** token scoped to the single repository, with only *Contents: read and write* and *Pull requests: read and write*.
- Set an expiry. Calendar a renewal; an expired token fails silently from the agent's point of view.
- The token acts as you. Commits and PRs look like yours, and a ruleset bypass that applies to you applies to the agent too. That is the main reason to prefer the App.
- Never commit the token or paste it into a prompt. Store it in the harness's secret store.

### Cloud environment setup

If your harness supports a setup script for the cloud environment, install the check tooling there so agents can run the checks before opening a PR:

```bash
#!/usr/bin/env bash
set -euo pipefail
python3 -m pip install --quiet pre-commit==4.7.0
pre-commit install-hooks   # pre-fetch hook environments so the first run is fast
```

Make sure the environment's network policy allows whatever the hooks download (PyPI and GitHub for the default config).

## 2. Protect `main`

The AGENTS.md rules ("never commit to main", "never merge your own PR") are instructions. Branch protection is what makes them true when an instruction is missed.

Use a **repository ruleset** (Settings → Rules → Rulesets) targeting the default branch. Rulesets supersede classic branch protection rules, can be layered, and support bypass lists. Recommended rules:

| Rule | Setting | Why |
|------|---------|-----|
| Restrict deletions | On | `main` can't be deleted. |
| Block force pushes | On | History on `main` can't be rewritten. |
| Require a pull request before merging | On | Nobody, agent or human, pushes to `main` directly. |
| Required approvals | 1 for teams; 0 for a solo repo (see below) | A human signs off. |
| Dismiss stale approvals when new commits are pushed | On | An approval covers the diff that was reviewed, not later pushes. |
| Require approval of the most recent reviewable push | On (teams) | The person who pushed last can't be the one who approves. |
| Require conversation resolution | On | Review comments get answered. |
| Require status checks to pass | On, select `pre-commit` | CI is a gate, not a suggestion. |
| Require branches to be up to date before merging | Optional | Stricter, but every merge forces a rebase on busy repos. |
| Require linear history | Optional | Pick squash or rebase merges if you want it. |

**Bypass list:** keep it empty, or limit it to repository admins with "for pull requests only". Never add the agent's GitHub App to the bypass list.

**Solo repositories:** GitHub doesn't let you approve your own PR, so "required approvals: 1" blocks a solo owner from merging agent PRs that were opened under their own identity (Option B). Either set approvals to 0 and rely on "require a pull request" plus required status checks, or, if your harness opens PRs as the app rather than as you, use the GitHub App (Option A) so you are not the PR author and can approve it. Check which identity your harness's PRs actually show before relying on this.

A second, optional ruleset can target `claude/**` branches to, for example, block force pushes from anyone other than the agent, or restrict who can create branches matching that pattern.

## 3. Review ownership

Decide who reviews agent PRs before the first one arrives.

- Add a `.github/CODEOWNERS` file so reviewers are requested automatically:

  ```text
  # Default owner for everything
  *                   @your-org/your-team
  # Repository machinery needs a second pair of eyes
  /.github/           @your-org/maintainers
  /AGENTS.md          @your-org/maintainers
  /.pre-commit-config.yaml @your-org/maintainers
  ```

  Turn on **Require review from Code Owners** in the ruleset if owners must sign off, not just be notified.
- Whoever *started* the agent session is a natural reviewer, but on a team they should not be the only one: they wrote the prompt, so they share the agent's blind spots.
- Changes to `AGENTS.md`, CI, and hook config change how every future agent behaves. Route them to maintainers.
- If the agent harness supports auto-fixing review comments or CI failures on PRs it opened, that's fine; merging is still a human step.

## 4. Adapting for a team or work environment

The template defaults suit a single owner. A team or employer setting changes a few things.

### Data classification

- Classify the repository before granting cloud access. Cloud sessions send whatever they read to the model provider. Check that the repository's classification allows that provider under your organization's policy, and that the vendor's data-retention terms meet it.
- Replace the "Data boundary" section of `AGENTS.md` with your organization's actual rules: what classes of data may be committed, what may be read by an agent, what may leave the network.
- If some material must never reach a cloud model, keep it out of the repository entirely (a gitignored local-only directory, or a separate repository the App is not installed on). A rule in `AGENTS.md` that says "don't read X" is weaker than X not being in the clone.
- Add secret scanning and push protection (Settings → Code security) in addition to the `detect-private-key` hook; the hook only catches PEM keys.

### Approval of the GitHub App

- In an organization, installing a GitHub App usually needs an org owner's approval. Expect to file a request and to explain the permissions it asks for and which repositories it will be installed on.
- Ask for selected-repository installation, and start with one low-risk repository as a trial.
- Your security team may require the token route to be disabled, or may require that agents use a dedicated machine user rather than a person's identity. Either works with this template; what matters is that the agent's identity is distinct and cannot bypass the ruleset.
- Record who approved the installation and when, and review the list of repositories it covers periodically.

### CI vs. local hooks

The template runs every hook in CI on each PR and on each push to `main`. Locally, hooks are optional.

- **Default (hooks at the `pre-commit` stage):** developers who run `pre-commit install` get the checks on every `git commit`; agents run `pre-commit run --all-files` before opening a PR; CI enforces. This suits most teams.
- **Manual-stage variant:** if something commits automatically and must never be blocked (a sync tool, a notebook autosave, a bot), move every hook to the `manual` stage so `git commit` runs nothing even with hooks installed, and CI is the only enforcement point:

  ```yaml
  default_stages: [manual]
  # ...and on each hook from an external repo, which declares its own stages:
  #   stages: [manual]
  ```

  Then add `--hook-stage manual` to the CI command and to the agent instructions in `AGENTS.md`. Without that flag, `pre-commit run` silently checks nothing, which is the main trap of this variant.
- **Heavy checks** (full test suites, builds) usually belong in CI only, as separate jobs, with the fast hygiene hooks in pre-commit. List the commands an agent must run before opening a PR in the project-specific section of `AGENTS.md`.
- **Anything that pushes to `main` automatically** must pull before it pushes, or its commits will conflict with merged PRs. The push trigger in the CI workflow catches what it breaks, but can't prevent it.
- Pin hook revisions and action versions to commit SHAs (`pre-commit autoupdate --freeze`; Dependabot or Renovate for actions). A team's CI should not change behavior because an upstream tag moved.

## 5. Checklist

- [ ] GitHub App (or fine-grained token) installed on this repository only
- [ ] Cloud environment setup script installs `pre-commit`
- [ ] Ruleset on `main`: PR required, `pre-commit` status check required, force pushes and deletions blocked, agent not on the bypass list
- [ ] `CODEOWNERS` in place (teams)
- [ ] `AGENTS.md` placeholder sections filled in or deleted
- [ ] One trial task run end to end: agent opens a PR, CI passes, a human reviews and merges
