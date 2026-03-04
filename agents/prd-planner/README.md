# prd-planner

User-facing guide for running the `prd-planner` orchestration agent.

## What It Does

`prd-planner` takes a Product Requirements Document and runs a two-stage workflow:

1. `prd-decompose` splits the PRD into domain folders.
2. `spec-plan` generates a `tasks.yaml` for each plannable domain folder.

This agent is useful when you want to go from a single PRD to implementation-ready task graphs without manually invoking both skills yourself.

## Files In This Agent

- `README.md` — human-facing usage guide
- `AGENT.md` — orchestration contract for the runtime
- `config.yaml` — skill dependencies and refs

## Inputs

Provide exactly one of:

- `prd_path`: path to a PRD markdown file
- `prd_markdown`: inline PRD markdown content

Optional:

- `out`: output directory, defaults to `prd-output/`

## Outputs

The agent writes a structured directory like:

```text
prd-output/
  |- meta.yaml
  |- contracts/
  |- {domain}/
  |    |- spec.md
  |    |- boundary.yaml
  |    |- config.yaml
  |    |- tasks.yaml
  |- uncategorized/
       |- spec.md
```

## Calling It From Claude Code

There is no built-in slash command for this repository today. In Claude Code, the practical way to run this agent is to explicitly tell Claude to use this agent's files as the contract.

Before running it, ensure one of these is true:

- Claude can read a local checkout of `nexus-skills` (recommended during development)
- Claude can access the remote `nexus-skills` repository referenced by `config.yaml`

If you have `nexus-skills` checked out locally next to this repo, use a prompt like this:

```text
Use the `prd-planner` agent in `agents/prd-planner/`.

Read:
- `agents/prd-planner/README.md`
- `agents/prd-planner/AGENT.md`
- `agents/prd-planner/config.yaml`
- `../nexus-skills/skills/prd-decompose/SKILL.md`
- `../nexus-skills/skills/spec-plan/SKILL.md`

Then run the workflow with:
- `prd_path`: path/to/my-prd.md
- `out`: prd-output

Use the local `../nexus-skills` checkout as the source of truth for skill instructions. Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after planning.
```

Use a prompt like this when your PRD already exists on disk:

```text
Use the `prd-planner` agent in `agents/prd-planner/`.

Read:
- `agents/prd-planner/README.md`
- `agents/prd-planner/AGENT.md`
- `agents/prd-planner/config.yaml`

Then run the workflow with:
- `prd_path`: path/to/my-prd.md
- `out`: prd-output

Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after planning.
```

Use a prompt like this when you want to paste the PRD inline:

```text
Use the `prd-planner` agent in `agents/prd-planner/`.

Read:
- `agents/prd-planner/README.md`
- `agents/prd-planner/AGENT.md`
- `agents/prd-planner/config.yaml`

Then run the workflow with:
- `prd_markdown`: <paste the PRD markdown here>
- `out`: prd-output

Follow the agent contract exactly. Do not skip validation gates. Produce the final summary after planning.
```

## Recommended Claude Code Workflow

1. Open the repository that contains both `nexus-agents` and the target PRD.
2. Make sure Claude can also access `nexus-skills`, either through a local checkout or through the remote repo.
3. Ask Claude to read this agent's `README.md`, `AGENT.md`, and `config.yaml`.
4. Provide either `prd_path` or `prd_markdown`.
5. Let Claude execute the steps defined in `AGENT.md`.
6. Review the generated `prd-output/` structure and the final summary.

## Notes For Development

- `config.yaml` currently tracks `nexus-skills` via `ref: "main"` for active development.
- For release or reproducible CI runs, switch `ref` to an immutable tag or commit SHA.
- `contracts/` and `uncategorized/` are intentionally skipped during the planning loop.
- If the final summary reports `unverified_domains > 0`, treat the run as partial success and review those domains manually.

## When Not To Use This

- You only need decomposition: use `prd-decompose`.
- You already have a single domain folder and only need tasks: use `spec-plan`.
- You want code generation or execution: use a downstream implementation agent.
