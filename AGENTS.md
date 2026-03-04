# AGENTS.md

Shared authoring and operating conventions for all agents in `nexus-agents`.

This file is the repo-level contract. Individual agents still define their own workflow in `agents/{agent-name}/AGENT.md`, but reusable rules should live here first.

## Why This File Exists

- Keep cross-agent conventions in one place instead of duplicating them in every agent
- Make the repository legible to an AI runtime before it opens any specific agent
- Separate stable architectural rules from agent-specific workflow steps

## Core Best Practices

### 1. Treat Agents As Orchestrators

- An agent coordinates skills. It does not replace them.
- Reusable logic belongs in `nexus-skills`; sequencing, gates, and aggregation belong here.
- Agents must not fabricate outputs that should be produced by a referenced skill.

### 2. Make Inputs, Outputs, And Gates Explicit

- Every agent must define accepted inputs, produced outputs, and stop conditions.
- Validation gates should happen before expensive or irreversible steps.
- If partial success is allowed, state exactly what can fail without aborting the whole run.

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

## Required Agent Shape

Each agent directory must contain:

- `README.md`: human-facing usage and invocation guidance
- `AGENT.md`: orchestration contract with steps, gates, constraints, and error handling
- `config.yaml`: skill dependencies, versions, refs, and source locations

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
