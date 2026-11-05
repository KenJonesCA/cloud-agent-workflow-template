# cloud-agent-workflow-template

A starter kit for repositories where cloud coding agents deliver their work as pull requests that a human reviews and merges.

Cloud agents run in ephemeral containers on a fresh clone of your repository. Anything they don't push is lost. This template gives them one well-defined way to get work back to you — a `claude/<topic>` branch and a single pull request per change — plus the instructions, checks, and repository settings that keep that safe.

## What's in it

| File | Purpose |
|------|---------|
| [`AGENTS.md`](AGENTS.md) | The agent instructions: local vs. cloud write paths, branch naming, one PR per change, never merge your own PR, run checks first, clone freshness, write zones. Project-specific sections are marked **PROJECT-SPECIFIC**. |
| [`CLAUDE.md`](CLAUDE.md) | One-line pointer to `AGENTS.md` for harnesses that don't read it yet. |
| [`.pre-commit-config.yaml`](.pre-commit-config.yaml) | General-purpose hygiene hooks (whitespace, line endings, YAML/JSON/TOML, conflict markers, large files, case conflicts, private keys). |
| [`.github/workflows/pre-commit.yml`](.github/workflows/pre-commit.yml) | Runs every hook on pull requests to `main` and on pushes to `main`. |
| [`.github/pull_request_template.md`](.github/pull_request_template.md) | PR description that mirrors the `AGENTS.md` rules. |
| [`docs/setup.md`](docs/setup.md) | GitHub App vs. token access, rulesets for `main`, review ownership, and adapting for a team or work environment. |
| [`docs/design-rationale.md`](docs/design-rationale.md) | Why each rule exists and when to revisit it. |

## Adopting it in an existing repository

1. **Copy the files in.** From the root of your repository:

   ```bash
   git switch -c adopt-agent-workflow
   ```

   ```bash
   TEMPLATE=https://github.com/<owner>/cloud-agent-workflow-template.git
   git clone --depth 1 "$TEMPLATE" /tmp/agent-template
   cp /tmp/agent-template/{AGENTS.md,CLAUDE.md,.pre-commit-config.yaml} .
   mkdir -p .github/workflows docs
   cp /tmp/agent-template/.github/pull_request_template.md .github/
   cp /tmp/agent-template/.github/workflows/pre-commit.yml .github/workflows/
   cp /tmp/agent-template/docs/{setup.md,design-rationale.md} docs/
   ```

   Check for collisions first: if you already have an `AGENTS.md`, `CLAUDE.md`, PR template, or pre-commit config, merge by hand instead of overwriting.

2. **Fill in `AGENTS.md`.** Search for `PROJECT-SPECIFIC` and replace or delete each block: what the repository is, the data boundary, the write-zone table (map it to your real directories), and your build/test commands. Keep the generic rules unless you have a reason to change them — and if you do, update `docs/design-rationale.md` to match.

3. **Tune the hooks.** Add `exclude:` patterns for generated or vendored paths, and add your language's linters. Then run them once over the whole repository and fix (or exclude) what they flag before turning CI on, so the first agent PR isn't blamed for old problems:

   ```bash
   pip install pre-commit
   ```

   ```bash
   pre-commit run --all-files
   ```

   If something in your workflow auto-commits and must never be blocked, use the manual-stage variant in [docs/setup.md](docs/setup.md#ci-vs-local-hooks).

4. **Set up access and protection.** Follow [docs/setup.md](docs/setup.md): install the GitHub App on this repository only, add a cloud environment setup script that installs `pre-commit`, and create a ruleset on `main` that requires a PR and the `pre-commit` check.

5. **Open the adoption PR yourself**, merge it, and then run one small trial task end to end with an agent: it should open a PR from a `claude/` branch, CI should pass, and you should be the one who merges.

## Starting a new repository from it

Use GitHub's **Use this template** button (enable it under Settings → General → Template repository), then do steps 2–5 above.

## Scope

This template is deliberately generic. It encodes a workflow, not a project: no language tooling, no merge strategy, no opinions about your directory layout beyond the write-zone table you fill in. The agent named in branch prefixes and the pointer file is Claude; rename `claude/` throughout if you use a different agent.
