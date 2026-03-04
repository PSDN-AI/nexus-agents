---
name: prd-planner
description: "Orchestrates end-to-end PRD decomposition and task planning by chaining prd-decompose and spec-plan skills. Takes a raw PRD and produces domain-specific task graphs ready for implementation."
license: MIT
compatibility: "Requires prd-decompose 0.1.0+ and spec-plan 0.1.0+ skills from nexus-skills."
metadata:
  author: PSDN-AI
  version: "0.1.0"
  category: Product Engineering
  tags:
    - orchestration
    - prd
    - planning
    - multi-agent
    - task-graph
---

# prd-planner

Transforms a Product Requirements Document into domain-scoped task graphs ready for implementation agents.

## Problem Statement

A raw PRD must be decomposed into domain-specific specifications and then planned into concrete task graphs before any implementation can begin — this agent automates that two-stage pipeline.

## When to Use

- You have a PRD (markdown) and need actionable task plans per domain
- You want deterministic decomposition followed by per-domain planning
- You need the full pipeline: PRD -> domain specs -> tasks.yaml files

## When NOT to Use

- You only need decomposition without planning — use `prd-decompose` directly
- You only need task planning for a single, pre-existing spec — use `spec-plan` directly
- You need implementation / code generation — use `agent-launcher` downstream
- The input is not a PRD (e.g., a bug report, design doc, or API spec)

## Pipeline

```
+-------+       +-----------------+       +-------------------+
|  PRD  | ----> |  prd-decompose  | ----> |  domain folders   |
+-------+       +-----------------+       +---+---+---+---+---+
                                              |   |   |   |
                                              v   v   v   v
                                        +-----------+  (per domain)
                                        |  spec-plan | ----------> tasks.yaml
                                        +-----------+
```

### Detailed Flow

```
PRD (markdown)
  |
  v
+------------------+
|  1. RECEIVE      |  Validate PRD input exists and is readable
+------------------+
  |
  v
+------------------+
|  2. RESOLVE      |  Fetch SKILL.md for prd-decompose and spec-plan
+------------------+
  |
  v
+------------------+
|  3. DECOMPOSE    |  Run prd-decompose -> domain folders
+------------------+
  |
  v
+------------------+
|  4. CHECK        |  Validate decomposition output structure
+------------------+
  |
  v
+------------------+
|  5. PLAN         |  Loop domain folders, run spec-plan each
+------------------+
  |
  v
+------------------+
|  6. VALIDATE     |  Verify tasks.yaml per domain
+------------------+
  |
  v
+------------------+
|  7. REPORT       |  Aggregate summary
+------------------+
```

## Prerequisites

- Access to `prd-decompose` and `spec-plan` skill definitions (SKILL.md files)
- A valid PRD in markdown format as input
- Skill sources configured in `config.yaml`

## Orchestration Workflow

### Step 1: RECEIVE

- [ ] Accept PRD input (file path or inline markdown)
- [ ] Verify the PRD is non-empty and contains readable content
- [ ] Identify or create the output directory (`prd-output/`)

**Gate**: If PRD is empty or unreadable, STOP with error.

### Step 2: RESOLVE

- [ ] Read `config.yaml` to get skill sources and versions
- [ ] Fetch `SKILL.md` for `prd-decompose` from configured source
- [ ] Fetch `SKILL.md` for `spec-plan` from configured source
- [ ] Confirm both skill definitions are accessible

**Gate**: If either SKILL.md is missing or inaccessible, STOP with error.

### Step 3: DECOMPOSE

- [ ] Invoke `prd-decompose` with the PRD input
- [ ] Pass the output directory as the target
- [ ] Wait for completion

Expected output after this step:

```
prd-output/
  |- meta.yaml
  |- contracts/
  |    |- ...
  |- {domain}/
  |    |- spec.md
  |    |- boundary.yaml
  |    |- config.yaml
  |- uncategorized/
       |- spec.md
```

### Step 4: CHECK

- [ ] Verify `meta.yaml` exists in the output root
- [ ] Verify `coverage_percent` in `meta.yaml` is greater than 0
- [ ] Verify at least one domain folder exists (beyond `contracts/` and `uncategorized/`)
- [ ] For each domain folder, verify `spec.md`, `boundary.yaml`, and `config.yaml` exist

**Gate**: If `meta.yaml` is missing or `coverage_percent` is 0, STOP with error.

### Step 5: PLAN

- [ ] List all subdirectories in the output root
- [ ] Skip `contracts/` — contains shared contract definitions, not a plannable domain
- [ ] Skip `uncategorized/` — contains unclassified requirements, not plannable
- [ ] For each remaining domain folder:
  - [ ] Pass the domain folder path to `spec-plan`
  - [ ] `spec-plan` reads `spec.md`, `boundary.yaml`, and `config.yaml` from that folder
  - [ ] `spec-plan` produces `tasks.yaml` inside the same domain folder
  - [ ] If `spec-plan` fails for a domain, log the error and continue to next domain

**Key detail**: `spec-plan` processes ONE domain folder at a time. The agent loops over all domain folders and invokes `spec-plan` once per domain.

### Step 6: VALIDATE

- [ ] For each domain folder that was planned:
  - [ ] Verify `tasks.yaml` exists
  - [ ] Verify `tasks.yaml` has valid YAML structure
  - [ ] Verify task coverage: every requirement in `spec.md` maps to at least one task
  - [ ] Check for cross-domain conflicts using `boundary.yaml` constraints
- [ ] Record which domains succeeded and which failed

### Step 7: REPORT

- [ ] Count: total domains, planned domains, failed domains, skipped domains
- [ ] List any domains that failed planning with error details
- [ ] Summarize total tasks generated across all domains
- [ ] Output the final directory structure

## Input / Output Contract

### Input

| Field | Type   | Required | Description                    |
|-------|--------|----------|--------------------------------|
| prd   | string | yes      | Path to PRD markdown file      |
| out   | string | no       | Output directory (default: `prd-output/`) |

### Output

```
prd-output/
  |- meta.yaml                    # from prd-decompose
  |- contracts/                   # from prd-decompose
  |- {domain}/
  |    |- spec.md                 # from prd-decompose
  |    |- boundary.yaml           # from prd-decompose
  |    |- config.yaml             # from prd-decompose
  |    |- tasks.yaml              # NEW -- from spec-plan
  |- uncategorized/
       |- spec.md                 # no tasks.yaml (skipped)
```

### Summary Output

The agent produces a final summary with:

- `total_domains`: number of domain folders discovered
- `planned_domains`: number that received a `tasks.yaml`
- `failed_domains`: number where `spec-plan` errored
- `skipped_domains`: number intentionally skipped (`contracts/`, `uncategorized/`)
- `total_tasks`: aggregate task count across all `tasks.yaml` files

## Constitutional Constraints

1. **Read-only PRD**: Never modify the input PRD. All output goes to the output directory.
2. **Skill boundary**: Only invoke skills listed in `config.yaml`. Do not inline skill logic.
3. **Fail-safe iteration**: If `spec-plan` fails for one domain, log and continue. Never abort the entire pipeline for a single domain failure.
4. **No fabrication**: Do not generate task content outside of `spec-plan`. The agent orchestrates; it does not author tasks.
5. **Deterministic output**: Given the same PRD and skill versions, the output structure must be identical.

## Error Handling

### Gate 1: Input Validation (Step 1)

Triggers when the PRD file is missing, empty, or not valid markdown. The agent stops immediately and reports the input error. No output directory is created.

### Gate 2: Skill Resolution (Step 2)

Triggers when `config.yaml` is missing or a referenced SKILL.md cannot be fetched. The agent stops before any decomposition. Reports which skill failed to resolve.

### Gate 3: Decomposition Validation (Step 4)

Triggers when `prd-decompose` produces no `meta.yaml`, or `coverage_percent` is 0, or no domain folders are created. The agent stops before planning and reports the decomposition failure.

Note: There is no gate at the planning step (Step 5). Individual domain failures are logged but do not halt the pipeline.

## Common Pitfalls

- **Passing the entire output directory to spec-plan**: `spec-plan` expects a single domain folder, not the root. Always pass `prd-output/{domain}/`, never `prd-output/`.
- **Planning contracts/ or uncategorized/**: These directories do not contain plannable domains. Always skip them in the loop.
- **Assuming all domains will succeed**: Some domains may have incomplete specs or boundary conflicts that cause `spec-plan` to fail. Always handle partial success.
- **Ignoring coverage_percent**: A `coverage_percent` of 0 in `meta.yaml` means decomposition produced no meaningful output. Do not proceed to planning.
- **Modifying skill behavior inline**: The agent must invoke skills as-is. Customizing or overriding skill logic within the agent violates the skill boundary constraint.
