---
name: prd-planner
description: "Orchestrates end-to-end PRD decomposition and task planning by chaining prd-decompose and spec-plan skills. Takes a raw PRD and produces domain-specific task graphs ready for implementation."
license: MIT
compatibility: "Requires prd-decompose 0.1.0 and spec-plan 0.1.0, each resolved through the `ref` declared in config.yaml."
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

This agent follows the shared conventions in the repository root `AGENTS.md`. This file defines only the `prd-planner`-specific workflow and constraints.

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

- [ ] Accept exactly one PRD input source: `prd_path` or `prd_markdown`
- [ ] If `prd_path` is used, verify the file exists and is readable
- [ ] If `prd_markdown` is used, verify the inline content is non-empty
- [ ] Resolve the output directory path (`prd-output/` by default), but do not create it yet
- [ ] If `out` already exists as a file, STOP with error
- [ ] If `out` already exists as a directory and contains any entries, STOP with error to avoid mixing stale artifacts with the new run
- [ ] If `out` already exists as an empty directory, it may be reused
- [ ] If `out` does not exist, verify its parent path is resolvable so the directory can be created after the gates pass

**Gate**: If neither input is provided, both are provided, the chosen input is empty/unreadable, or the output path is unsafe to reuse, STOP with error.

### Step 2: RESOLVE

- [ ] Read `config.yaml` to get skill sources, versions, and refs
- [ ] Fetch `SKILL.md` for `prd-decompose` from the configured `source` at the configured `ref`
- [ ] Fetch `SKILL.md` for `spec-plan` from the configured `source` at the configured `ref`
- [ ] Confirm both skill definitions are accessible

**Gate**: If either SKILL.md is missing, inaccessible, or lacks a `ref` in `config.yaml`, STOP with error.

### Step 3: DECOMPOSE

- [ ] Create the output directory after Step 1 and Step 2 have passed
- [ ] If the directory already exists and is empty, reuse it as-is; do not clear or mutate any pre-existing files
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
- [ ] Build `plannable_domains` from all remaining domain folders
- [ ] For each domain in `plannable_domains`:
  - [ ] Pass the domain folder path to `spec-plan`
  - [ ] `spec-plan` reads `spec.md`, `boundary.yaml`, and `config.yaml` from that folder
  - [ ] Invoke `spec-plan` using its normal planning flow, including its own built-in validation phase before output is finalized
  - [ ] `spec-plan` produces `tasks.yaml` inside the same domain folder
  - [ ] If `spec-plan` fails for a domain, log the error and continue to next domain
  - [ ] Never create a fallback or placeholder `tasks.yaml` for a failed domain
  - [ ] If a failed planning attempt leaves a `tasks.yaml` behind, treat it as an untrusted artifact until Step 6 validates it

**Key detail**: `spec-plan` processes ONE domain folder at a time. The agent loops over all domain folders and invokes `spec-plan` once per domain.

### Step 6: VALIDATE

- [ ] For each domain folder that was planned:
  - [ ] Verify `tasks.yaml` exists
  - [ ] Verify `tasks.yaml` has valid YAML structure
  - [ ] If `spec-plan` publishes a machine-readable schema, verify the required top-level fields
  - [ ] If no published schema exists, mark the domain as structurally unverified and continue
  - [ ] Record task counts only when the `tasks.yaml` schema exposes a machine-readable task list
- [ ] If `tasks.yaml` exists but fails structural validation, record a validation failure for the domain and mark the file as untrusted
- [ ] Record which domains succeeded, failed, remain structurally unverified, or contain untrusted `tasks.yaml` artifacts

**Boundary**: Limit validation to file existence, YAML parsing, and published schema checks. Do not infer requirement-to-task coverage or cross-domain conflicts unless a dedicated validation skill is added to `config.yaml`.

### Step 7: REPORT

- [ ] Count: total plannable domains, planned domains, failed domains, skipped entries, unverified domains, tainted domains
- [ ] Build a machine-readable final summary using the field names and ordering defined in `Summary Output`
- [ ] List any domains that failed planning with error details
- [ ] List any domains with untrusted `tasks.yaml` artifacts that must not be consumed downstream
- [ ] Summarize total tasks generated across all domains
- [ ] Output the final directory structure
- [ ] If a human-readable recap is included, it must be derived from the machine-readable summary and must not contradict it

**Result classification**: If `failed_domains` or `tainted_domains` is greater than 0, the run is a partial failure. If both are 0 but `unverified_domains` is greater than 0, the run is a partial success and requires manual review before downstream execution.

## Input / Output Contract

### Input

| Field | Type   | Required | Description                    |
|-------|--------|----------|--------------------------------|
| prd_path | string | conditional | Path to the PRD markdown file |
| prd_markdown | string | conditional | Inline PRD markdown content |
| out   | string | no       | Output directory (default: `prd-output/`) |

Exactly one of `prd_path` or `prd_markdown` must be provided.

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

The agent must produce a final machine-readable summary object with these top-level fields, in this order:

- `status`: one of `success`, `partial_success`, `partial_failure`, `error`
- `total_domains`: number of plannable domain folders discovered (excludes `contracts/` and `uncategorized/`); may be `0` when a hard gate fails before planning
- `planned_domains`: number of domains with `domain_results.status: planned`
- `failed_domains`: number of domains with `domain_results.status: failed`
- `skipped_entries`: number intentionally skipped (`contracts/`, `uncategorized/`)
- `unverified_domains`: number of domains with `domain_results.status: unverified`
- `tainted_domains`: number of domains with `domain_results.status: tainted`
- `total_tasks`: aggregate task count across validated or structurally acceptable `tasks.yaml` files only
- `output_root`: resolved output directory path used for the run
- `error_stage`: `null` unless `status` is `error`
- `error_reason`: `null` unless `status` is `error`
- `error_message`: `null` unless `status` is `error`
- `failed_domain_details`: array of per-domain failure records
- `tainted_domain_details`: array of per-domain taint records
- `skipped_entry_details`: array of skipped entry records
- `domain_results`: array of per-domain result records for every plannable domain, sorted by domain name

The `status` field is derived as follows:

- `error`: a hard gate failed before the planning loop completed
- `partial_failure`: `failed_domains` or `tainted_domains` is greater than 0
- `partial_success`: `failed_domains` and `tainted_domains` are 0, but `unverified_domains` is greater than 0
- `success`: `failed_domains`, `tainted_domains`, and `unverified_domains` are all 0

Top-level `status` must be derived from the finalized `domain_results` state machine below when the planning loop is reached. If a hard gate fails before planning begins, emit `status: error`, set `domain_results: []`, set all domain count fields to `0`, and populate `error_stage`, `error_reason`, and `error_message`.

When `status` is `error`, populate:

- `error_stage`: one of `receive`, `resolve`, `check`
- `error_reason`: one of the gate-level reason codes defined in `Reason Code Registry`
- `error_message`: concise human-readable explanation

`failed_domain_details` must contain objects with:

- `domain`: domain folder name
- `stage`: fixed value `plan` or `validate`
- `reason`: one of the failure reason codes defined in `Reason Code Registry`
- `message`: concise human-readable explanation

`tainted_domain_details` must contain objects with:

- `domain`: domain folder name
- `reason`: one of the taint reason codes defined in `Reason Code Registry`
- `message`: concise human-readable explanation
- `tasks_yaml_path`: path to the untrusted file

`skipped_entry_details` must contain objects with:

- `name`: skipped directory name
- `reason`: fixed value `non_plannable_entry`

Each item in `domain_results` must contain:

- `domain`: domain folder name
- `status`: one of `planned`, `failed`, `unverified`, `tainted`
- `tasks_yaml_path`: path to `tasks.yaml` when present, otherwise `null`
- `task_count`: integer when machine-readable, otherwise `null`
- `notes`: short explanation of the final classification

`failed_domain_details`, `tainted_domain_details`, and `domain_results` must all be sorted by domain name for deterministic output.

### Domain Result State Machine

`domain_results.status` is the final exclusive classification for each plannable domain after the planning loop begins.

General invariants:

- Each plannable domain must appear exactly once in `domain_results`
- Each domain must end in exactly one final status
- Top-level count fields are derived from `domain_results`, not computed independently
- `total_domains` must equal the length of `domain_results`
- `planned_domains` must equal the count of `domain_results` items with `status: planned`
- `unverified_domains` must equal the count of `domain_results` items with `status: unverified`
- `failed_domains` must equal the count of `domain_results` items with `status: failed`
- `tainted_domains` must equal the count of `domain_results` items with `status: tainted`
- `planned_domains + unverified_domains + failed_domains + tainted_domains` must equal `total_domains`
- If `status: error`, `domain_results` must be empty and all domain count fields must be `0`

Allowed final states:

- `planned`: a usable `tasks.yaml` exists, passed all required structural checks, and is safe for downstream consumption
- `unverified`: a structurally acceptable `tasks.yaml` exists, but full validation could not be completed because no published machine-readable schema was available
- `failed`: no usable `tasks.yaml` is available for the domain
- `tainted`: a `tasks.yaml` file exists but is explicitly untrusted and must not be consumed downstream

Per-state field rules:

- `planned`: `tasks_yaml_path` must be non-null; `task_count` may be an integer or `null`; the domain must not appear in `failed_domain_details` or `tainted_domain_details`
- `unverified`: `tasks_yaml_path` must be non-null; `task_count` may be an integer or `null`; the domain must not appear in `failed_domain_details` or `tainted_domain_details`
- `failed`: `tasks_yaml_path` must be `null`; `task_count` must be `null`; the domain must appear exactly once in `failed_domain_details`; the domain must not appear in `tainted_domain_details`
- `tainted`: `tasks_yaml_path` must be non-null; `task_count` must be `null`; the domain must appear exactly once in `tainted_domain_details`; the domain may also appear once in `failed_domain_details` when the taint was caused by a planning or validation failure

Transition rules:

- Initial decomposition output begins in an implicit pre-planning state and must not be emitted in `domain_results`
- A domain moves to `planned` only after `tasks.yaml` exists and passes all required checks
- A domain moves to `unverified` only after `tasks.yaml` exists, YAML parsing succeeds, and validation stops only because no published schema is available
- A domain moves to `failed` when planning errors, no `tasks.yaml` is produced, or no trusted artifact remains
- A domain moves to `tainted` when an untrusted `tasks.yaml` exists after failure or invalidation
- Once a domain reaches `tainted`, its final emitted state must remain `tainted` even if the underlying failure is also recorded in `failed_domain_details`

Illegal combinations:

- `status: planned` with `tasks_yaml_path: null`
- `status: unverified` with `tasks_yaml_path: null`
- `status: failed` with a non-null `tasks_yaml_path`
- `status: failed` with a non-null `task_count`
- `status: tainted` with `tasks_yaml_path: null`
- `status: tainted` with a non-null `task_count`
- Any domain appearing more than once in `domain_results`
- Any mismatch between top-level counts and the derived counts from `domain_results`

### Reason Code Registry

All `reason` fields are closed enums. Do not invent ad hoc values at runtime.

Gate-level `error_reason` values:

- `missing_input`: neither `prd_path` nor `prd_markdown` was provided
- `conflicting_input`: both `prd_path` and `prd_markdown` were provided
- `unreadable_prd`: `prd_path` does not exist or cannot be read
- `empty_prd_markdown`: inline PRD content is empty
- `output_path_is_file`: `out` resolves to an existing file
- `output_dir_not_empty`: `out` resolves to an existing non-empty directory
- `skill_config_missing`: `config.yaml` is missing or unreadable
- `missing_skill_ref`: a required skill entry is missing a `ref`
- `skill_definition_unavailable`: a required `SKILL.md` cannot be resolved
- `missing_meta_yaml`: decomposition completed without `meta.yaml`
- `zero_coverage`: `meta.yaml` reports `coverage_percent` as 0
- `no_plannable_domains`: decomposition produced no plannable domain directories

`failed_domain_details.reason` values:

- `spec_plan_error`: `spec-plan` returned an execution failure for the domain
- `missing_tasks_yaml`: planning completed or returned, but no `tasks.yaml` was produced
- `invalid_tasks_yaml`: `tasks.yaml` exists but is not valid YAML
- `schema_validation_failed`: `tasks.yaml` is valid YAML but fails a published schema check

`tainted_domain_details.reason` values:

- `leftover_tasks_yaml_after_failure`: a failed planning attempt left a `tasks.yaml` behind
- `structurally_invalid_tasks_yaml`: `tasks.yaml` exists but is structurally invalid and must not be consumed downstream

`skipped_entry_details.reason` values:

- `non_plannable_entry`: directory is intentionally excluded from planning

Mapping rules:

- If `spec-plan` itself errors, use `spec_plan_error`
- If no `tasks.yaml` exists after a nominally completed planning attempt, use `missing_tasks_yaml`
- If YAML parsing fails, use `invalid_tasks_yaml` and also taint the artifact only if a file exists
- If schema validation fails, use `schema_validation_failed` and mark the artifact as tainted with `structurally_invalid_tasks_yaml`

If `tainted_domains` is greater than 0, the result must be treated as a partial failure. If `tainted_domains` is 0 but `unverified_domains` is greater than 0, the result must be treated as partial success rather than a fully verified successful run.

Illustrative shape:

```yaml
status: partial_failure
total_domains: 3
planned_domains: 1
failed_domains: 1
skipped_entries: 2
unverified_domains: 0
tainted_domains: 1
total_tasks: 12
output_root: prd-output
error_stage: null
error_reason: null
error_message: null
failed_domain_details:
  - domain: api
    stage: plan
    reason: spec_plan_error
    message: spec-plan failed while generating tasks
tainted_domain_details:
  - domain: api
    reason: leftover_tasks_yaml_after_failure
    message: tasks.yaml exists after a failed planning attempt
    tasks_yaml_path: prd-output/api/tasks.yaml
skipped_entry_details:
  - name: contracts
    reason: non_plannable_entry
  - name: uncategorized
    reason: non_plannable_entry
domain_results:
  - domain: api
    status: tainted
    tasks_yaml_path: prd-output/api/tasks.yaml
    task_count: null
    notes: planning failed and left an untrusted tasks file
  - domain: frontend
    status: planned
    tasks_yaml_path: prd-output/frontend/tasks.yaml
    task_count: 12
    notes: tasks file passed structural validation
  - domain: infra
    status: failed
    tasks_yaml_path: null
    task_count: null
    notes: spec-plan did not produce a usable tasks file
```

### Output Directory Policy

- `out` must be a new directory path or an already-empty directory
- The agent must never merge a new run into a non-empty output directory
- The agent must never delete unrelated files to make `out` usable
- Re-running the workflow should use a fresh `out` path unless the previous empty directory was intentionally preserved

### Failure Artifact Policy

- Always preserve decomposition artifacts (`meta.yaml`, `contracts/`, `spec.md`, `boundary.yaml`, `config.yaml`) for inspection after a failed run
- A valid `tasks.yaml` from a successfully planned domain may remain in place even if another domain fails
- A `tasks.yaml` left behind by a failed or structurally invalid planning attempt is an untrusted artifact; report it, exclude it from success counts, and do not treat it as downstream-ready
- The agent must not fabricate cleanup markers or replacement task files to hide an upstream failure

## Constitutional Constraints

1. **Read-only PRD**: Never modify the input PRD. All output goes to the output directory.
2. **Skill boundary**: Only invoke skills listed in `config.yaml`. Non-skill checks are limited to file existence, syntax parsing, and published schema validation.
3. **Fail-safe iteration**: If `spec-plan` fails for one domain, log and continue. Never abort the entire pipeline for a single domain failure.
4. **No fabrication**: Do not generate task content outside of `spec-plan`. The agent orchestrates; it does not author tasks.
5. **Deterministic output**: Given the same PRD bytes and the same resolved skill refs, the output structure should be identical. This guarantee is strongest when `ref` points to an immutable tag or commit SHA.
6. **No implicit overwrite**: Never reuse a non-empty output directory for a new run.
7. **Preserve evidence**: Do not delete failed-run artifacts merely to make the run look clean; classify them explicitly instead.

## Error Handling

### Gate 1: Input Validation (Step 1)

Triggers when neither input source is provided, both are provided, the PRD file is missing/unreadable, the inline markdown is empty, the `out` path points to an existing file, or the `out` directory already exists and is non-empty. The agent stops immediately and reports the input error. No new output directory is created.

Use one of these `error_reason` values: `missing_input`, `conflicting_input`, `unreadable_prd`, `empty_prd_markdown`, `output_path_is_file`, `output_dir_not_empty`.

### Gate 2: Skill Resolution (Step 2)

Triggers when `config.yaml` is missing, a referenced `ref` is missing, or a referenced SKILL.md cannot be fetched at the configured ref. The agent stops before any decomposition. Reports which skill failed to resolve.

Use one of these `error_reason` values: `skill_config_missing`, `missing_skill_ref`, `skill_definition_unavailable`.

### Gate 3: Decomposition Validation (Step 4)

Triggers when `prd-decompose` produces no `meta.yaml`, or `coverage_percent` is 0, or no domain folders are created. The agent stops before planning and reports the decomposition failure.

Use one of these `error_reason` values: `missing_meta_yaml`, `zero_coverage`, `no_plannable_domains`.

Partial decomposition output may remain on disk for inspection. These artifacts are not a successful run result and must not be treated as validated output.

Note: There is no gate at the planning step (Step 5). Individual domain failures are logged but do not halt the pipeline.

## Common Pitfalls

- **Passing the entire output directory to spec-plan**: `spec-plan` expects a single domain folder, not the root. Always pass `prd-output/{domain}/`, never `prd-output/`.
- **Planning contracts/ or uncategorized/**: These directories do not contain plannable domains. Always skip them in the loop.
- **Assuming all domains will succeed**: Some domains may have incomplete specs or boundary conflicts that cause `spec-plan` to fail. Always handle partial success.
- **Reusing a non-empty output directory**: This can mix stale artifacts from a previous run with fresh output and corrupt the summary. Use a new `out` path or an empty directory only.
- **Ignoring coverage_percent**: A `coverage_percent` of 0 in `meta.yaml` means decomposition produced no meaningful output. Do not proceed to planning.
- **Modifying skill behavior inline**: The agent must invoke skills as-is. Customizing or overriding skill logic within the agent violates the skill boundary constraint.
- **Creating the output directory before input validation**: Resolve the output path first, but only create it after the input and skill gates pass.
- **Treating leftover `tasks.yaml` as success**: A `tasks.yaml` file that survived a failed or invalid planning attempt is untrusted until validated. Do not count it as planned output.
- **Inventing semantic validation rules**: Requirement-to-task coverage and cross-domain conflict checks require a dedicated validation skill or published schema contract.
