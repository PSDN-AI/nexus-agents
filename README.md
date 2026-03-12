# nexus-agents

Orchestration agents for the PSDN-AI nexus ecosystem. Each agent chains multiple [nexus-skills](https://github.com/PSDN-AI/nexus-skills) into end-to-end workflows.

## How Agents Work

This repository defines a shared `AGENTS.md` contract plus per-agent **knowledge-only** packages that define:

- **`AGENTS.md`** — repo-level best practices and shared invariants for all agents.
- **`README.md`** — human-facing usage and invocation guide.
- **`AGENT.md`** — orchestration instructions: pipeline steps, input/output contracts, constraints, and error handling.
- **`config.yaml`** — declares which skills the agent depends on, with version metadata, refs, and source locations.

Agents do not contain scripts or GitHub Actions. They are consumed by an AI runtime that reads the AGENT.md and follows its workflow, invoking the referenced skills at each step.

## Available Agents

| Agent | Description | Skills Used |
|-------|-------------|-------------|
| [prd-planner](agents/prd-planner/) | Decomposes a PRD into domain specs, then generates task graphs per domain | `prd-decompose`, `spec-plan` |
| [gha-planner](agents/gha-planner/) | Scans a repo's tech stack and generates hardened GitHub Actions workflows | `gha-create` |

## Cross-Repo Relationship

```
nexus-skills (atomic capabilities)
  |- prd-decompose
  |- spec-plan
  |- agent-launcher
  |- ...

nexus-agents (orchestration)
  |- prd-planner  -->  chains prd-decompose + spec-plan
  |- gha-planner  -->  wraps gha-create
  |- ...
```

Skills are standalone, single-purpose units. Agents compose skills into multi-step pipelines. During active development, an agent's `config.yaml` may track a branch ref such as `main`; for releases, pin each dependency to an immutable tag or commit SHA.

## Adding a New Agent

1. Read `AGENTS.md` and follow the shared authoring conventions
2. Create `agents/{agent-name}/README.md` with human-facing usage and invocation instructions
3. Create `agents/{agent-name}/AGENT.md` with YAML frontmatter (name, description, license, metadata)
4. Create `agents/{agent-name}/config.yaml` listing skill dependencies, including a `ref` for each skill (prefer immutable refs for releases)
5. Add the agent to the table above

## License

MIT
