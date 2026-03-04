# nexus-agents

Orchestration agents for the PSDN-AI nexus ecosystem. Each agent chains multiple [nexus-skills](https://github.com/PSDN-AI/nexus-skills) into end-to-end workflows.

## How Agents Work

An agent is a **knowledge-only** package that defines:

- **`AGENT.md`** — orchestration instructions: pipeline steps, input/output contracts, constraints, and error handling.
- **`config.yaml`** — declares which skills the agent depends on, with pinned versions and source locations.

Agents do not contain scripts or GitHub Actions. They are consumed by an AI runtime that reads the AGENT.md and follows its workflow, invoking the referenced skills at each step.

## Available Agents

| Agent | Description | Skills Used |
|-------|-------------|-------------|
| [prd-planner](agents/prd-planner/) | Decomposes a PRD into domain specs, then generates task graphs per domain | `prd-decompose`, `spec-plan` |

## Cross-Repo Relationship

```
nexus-skills (atomic capabilities)
  |- prd-decompose
  |- spec-plan
  |- agent-launcher
  |- ...

nexus-agents (orchestration)
  |- prd-planner  -->  chains prd-decompose + spec-plan
  |- ...
```

Skills are standalone, single-purpose units. Agents compose skills into multi-step pipelines. An agent's `config.yaml` pins the exact skill versions it depends on.

## Adding a New Agent

1. Create `agents/{agent-name}/AGENT.md` with YAML frontmatter (name, description, license, metadata)
2. Create `agents/{agent-name}/config.yaml` listing skill dependencies
3. Add the agent to the table above

## License

MIT
