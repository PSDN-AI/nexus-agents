# AGENTS.md

Shared authoring and operating conventions for all agents in `nexus-agents`.

This file is the repo-level contract and the single source of truth for repository guidelines. Individual agents still define their own workflow in `agents/{agent-name}/AGENT.md`, but reusable rules should live here first.

## What This Repo Is

`nexus-agents` is a **knowledge-only** repository — no scripts, no builds, no tests, no CI. There is nothing to compile or run. Each agent is a set of markdown and YAML files consumed by an AI runtime.

Agents orchestrate multi-step workflows by chaining skills from the companion repo [nexus-skills](https://github.com/PSDN-AI/nexus-skills). Skills are atomic, single-purpose units; agents compose them into pipelines.

## Repository Structure

```
AGENTS.md            # Repo-level conventions shared by all agents (this file)
CLAUDE.md            # Claude Code entry point — imports this file
agents/{agent-name}/
  ├── README.md        # Human-facing usage and invocation guide
  ├── AGENT.md         # Orchestration contract: pipeline steps, I/O contracts, constraints, error handling
  └── config.yaml      # Skill dependencies with version, ref, and source location
reference.md           # Curated reference material (AI agent engineering practices)
```

## Why This File Exists

- Keep cross-agent conventions in one place instead of duplicating them in every agent
- Make the repository legible to an AI runtime before it opens any specific agent
- Separate stable architectural rules from agent-specific workflow steps

## Agent Anatomy

Each agent directory must contain:

- **README.md**: human-facing usage and invocation guidance, including example prompts for invoking the agent from Claude Code
- **AGENT.md**: the primary orchestration contract. Uses YAML frontmatter (name, description, license, metadata) followed by a detailed step-by-step workflow with explicit gates, I/O contracts, constitutional constraints, and error handling
- **config.yaml**: declares skill dependencies. Each skill entry has `name`, `version`, `ref`, `source`, and `path`. During development, `ref` tracks `main`; for releases, pin to an immutable tag or commit SHA

## Core Best Practices

### 1. Treat Agents As Orchestrators

- An agent coordinates skills. It does not replace them.
- Reusable logic belongs in `nexus-skills`; sequencing, gates, and aggregation belong here.
- Agents must not fabricate outputs that should be produced by a referenced skill.
- Agents invoke only skills listed in their `config.yaml` — no inline task fabrication.

### 2. Make Inputs, Outputs, And Gates Explicit

- Every agent must define accepted inputs, produced outputs, and stop conditions.
- Validation gates should happen before expensive or irreversible steps.
- If partial success is allowed, state exactly what can fail without aborting the whole run.
- The input PRD is always read-only. Output goes to a separate directory by default; agents that write into the target repository (e.g., to a well-known path like `.github/workflows/`) must declare this in their `AGENT.md`, must never modify or delete existing files, and must use conflict detection to avoid overwrites.

### 3. Keep Shared Invariants Centralized

- Repo-wide rules should be documented here, not re-invented per agent.
- Agent-specific exceptions should be documented locally in that agent's `AGENT.md`.
- If a new rule is likely to apply to future agents, add it here before or alongside local edits.

### 4. Prefer Deterministic, Inspectable Workflows

- Reference only skills listed in `config.yaml`.
- During development, `ref: main` is acceptable; for release or CI, pin to an immutable tag or commit SHA.
- The same inputs plus the same resolved skill refs should produce the same output structure.

### 5. Optimize For Runtime Legibility

- The runtime should be able to understand an agent by reading a small, obvious set of files.
- Avoid scattering required behavior across unrelated docs.
- Put the human-facing invocation guide in `README.md`, the machine-oriented workflow in `AGENT.md`, and dependency resolution in `config.yaml`.

### 6. Design For Failure Containment

- A failure in one sub-step should only abort the full run if the contract says it is a hard gate.
- Where possible, prefer per-unit failure reporting over all-or-nothing failure.
- Summaries should clearly distinguish success, partial success, skipped work, and unverified output.

### 7. Prefer Structured Summaries

- Each agent should define a stable machine-readable final summary shape.
- Status should use a small explicit enum rather than prose-only labels.
- Per-unit outcomes should be reported in deterministic arrays that downstream agents can parse reliably.
- Any machine-readable `reason` field should use a fixed documented enum, not ad hoc strings.
- If an agent emits per-unit statuses, define them as an explicit state machine with illegal combinations called out.

## Key Conventions

- Agents **never contain executable code** — no scripts, no GitHub Actions.
- Skills come from `nexus-skills` repo; agents from this repo. Keep the boundary clean.
- `reference.md` at the root is a curated collection of external references — append new entries under the appropriate topic heading following the existing format.
- **All repository content (markdown, YAML, comments) must be written in English.** The user may communicate in other languages, but all file content committed to the repo must be English only.

## Adding a New Agent

1. Create `agents/{agent-name}/` with `README.md`, `AGENT.md`, and `config.yaml`.
2. Follow the conventions in this file and the agent anatomy described above.
3. AGENT.md must include: YAML frontmatter, step-by-step workflow with numbered gates, I/O contract table, constitutional constraints, error handling section, and common pitfalls.
4. config.yaml must list all skill dependencies with `ref` fields.
5. Update the "Available Agents" table in the root `README.md`.

## Authoring Checklist

Before adding or changing an agent, verify:

- The workflow can be described as composition of existing skills
- The agent names the exact skill dependencies in `config.yaml`
- Hard gates and non-fatal failures are clearly separated
- Inputs are unambiguous and mutually exclusive where needed
- Outputs are concrete enough to validate
- Final summary fields and status values are stable enough for downstream consumption
- Any new cross-agent rule has been added to this file

## Change Policy

Default sequence:

1. Update this file when the lesson is generalizable.
2. Update the specific agent when the lesson affects its local workflow.

Exception:

- If the current agent has an immediate correctness problem, fix it first, then backfill the shared rule here.

In practice, the right move is usually both: add the reusable rule here, then apply it to the current agent if relevant.
