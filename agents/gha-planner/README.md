# gha-planner

User-facing guide for running the `gha-planner` orchestration agent.

## What It Does

`gha-planner` analyzes a target repository and generates a complete set of GitHub Actions workflows tailored to its tech stack:

1. Scans the repo to detect languages, frameworks, package managers, and deployment targets.
2. Plans which workflows are needed (CI, CD, Docker, release, security, etc.).
3. Invokes `gha-create` to produce hardened workflow YAML for each planned workflow.
4. Validates every generated workflow against gha-create's 8-rule checklist.
5. Produces a structured report with per-workflow results.

## Files In This Agent

- `README.md` — human-facing usage guide
- `AGENT.md` — orchestration contract for the runtime
- `config.yaml` — skill dependencies and refs

## Inputs

Required:

- `repo_path`: path to the target repository root

Optional:

- `out`: output directory, defaults to `gha-output/`

`out` must point to a new directory path or an already-empty directory. Do not reuse a non-empty directory from a previous run.

## Outputs

The agent writes a structured directory like:

```text
gha-output/
  |- scan.yaml
  |- plan.yaml
  |- workflows/
  |    |- ci.yml
  |    |- deploy.yml
  |    |- docker-build.yml
  |    |- security.yml
  |- validation/
       |- ci.yml.result
       |- deploy.yml.result
       |- docker-build.yml.result
       |- security.yml.result
```

It also produces a machine-readable final summary with stable top-level fields such as `status`, `total_workflows`, `generated_workflows`, `validated_workflows`, `failed_workflows`, `advisory_warnings`, and per-workflow detail arrays. Its `reason` fields use fixed enums, so downstream consumers should branch on those codes instead of parsing free-form prose.

## Calling It From Claude Code

There is no built-in slash command for this repository today. In Claude Code, the practical way to run this agent is to explicitly tell Claude to use this agent's files as the contract.

Before running it, ensure one of these is true:

- Claude can read a local checkout of `nexus-skills` (recommended during development)
- Claude can access the remote `nexus-skills` repository referenced by `config.yaml`

If you have `nexus-skills` checked out locally next to this repo, use a prompt like this:

```text
Use the `gha-planner` agent in `agents/gha-planner/`.

Read:
- `AGENTS.md`
- `agents/gha-planner/README.md`
- `agents/gha-planner/AGENT.md`
- `agents/gha-planner/config.yaml`
- `../nexus-skills/skills/gha-create/SKILL.md`

Then run the workflow with:
- `repo_path`: path/to/target-repo
- `out`: gha-output

Use the local `../nexus-skills` checkout as the source of truth for skill instructions. Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after all workflows are generated and validated.
```

Use a prompt like this when the skill is resolved remotely:

```text
Use the `gha-planner` agent in `agents/gha-planner/`.

Read:
- `AGENTS.md`
- `agents/gha-planner/README.md`
- `agents/gha-planner/AGENT.md`
- `agents/gha-planner/config.yaml`

Then run the workflow with:
- `repo_path`: path/to/target-repo
- `out`: gha-output

Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after all workflows are generated and validated.
```

Minimal prompt when defaults are acceptable:

```text
Use the `gha-planner` agent in `agents/gha-planner/`.

Read:
- `AGENTS.md`
- `agents/gha-planner/README.md`
- `agents/gha-planner/AGENT.md`
- `agents/gha-planner/config.yaml`

Then run the workflow with:
- `repo_path`: .

Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after all workflows are generated and validated.
```

## Recommended Claude Code Workflow

1. Open the repository that contains both `nexus-agents` and the target repo.
2. Make sure Claude can also access `nexus-skills`, either through a local checkout or through the remote repo.
3. Ask Claude to read `AGENTS.md` first, then this agent's `README.md`, `AGENT.md`, and `config.yaml`.
4. Provide `repo_path` pointing to the target repository.
5. Let Claude execute the steps defined in `AGENT.md`.
6. Review the generated `gha-output/` structure and the final summary.

## Notes For Development

- `config.yaml` currently tracks `nexus-skills` via `ref: "main"` for active development.
- For release or reproducible CI runs, switch `ref` to an immutable tag or commit SHA.
- Required validation checks: S1 (SHA pinning), S2 (permissions), S4 (injection prevention), E2 (caching), E3 (concurrency).
- Advisory checks: S3 (OIDC), E1 (path filtering), E4 (matrix optimization). These produce warnings but do not fail workflows.
- If existing workflows are detected in the target repo, they are noted in the report but never modified.
- Prefer consuming the structured summary fields directly. Human-readable recap text, if present, is secondary.

## When Not To Use This

- You only need a single workflow: use `gha-create` directly.
- You need non-GitHub-Actions CI/CD: this agent is GitHub-Actions-only.
- You want to review or harden existing workflows: use `gha-create` directly.
- The target is not a software repository.
