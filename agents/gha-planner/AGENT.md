---
name: gha-planner
description: "Orchestrates end-to-end GitHub Actions workflow generation by scanning a target repository's tech stack, planning needed workflows, and invoking gha-create to produce hardened, production-grade workflow YAML files with validation and structured reporting."
license: MIT
compatibility: "Requires gha-create 0.1.0, resolved through the `ref` declared in config.yaml."
metadata:
  author: PSDN-AI
  version: "0.1.0"
  category: CI/CD & DevOps
  tags:
    - orchestration
    - github-actions
    - ci-cd
    - devops
    - workflow-generation
---

# gha-planner

Analyzes a target repository and generates a complete set of hardened GitHub Actions workflows tailored to its tech stack.

This agent follows the shared conventions in the repository root `AGENTS.md`. This file defines only the `gha-planner`-specific workflow and constraints.

## Problem Statement

A repository needs GitHub Actions workflows tailored to its specific tech stack — but manually determining which workflows are needed, then authoring each one to production-grade standards (SHA-pinned actions, least-privilege permissions, OIDC, injection prevention, caching, concurrency) is tedious and error-prone. This agent automates the full pipeline: scan, plan, generate, validate, report.

## When to Use

- You have a repository and want to bootstrap its entire GitHub Actions CI/CD setup
- You want workflows that follow elite-level security and efficiency practices from day one
- You need a structured inventory of what workflows were generated and their validation status

## When NOT to Use

- You only need a single workflow for a known purpose — use `gha-create` directly
- You need non-GitHub-Actions CI/CD (e.g., GitLab CI, CircleCI)
- You want to modify or review existing workflows without generating new ones — use `gha-create` directly
- The target is not a software repository (no tech stack to scan)

## Pipeline

```
+------------+       +-----------+       +------------+
| Target Repo| ----> |   SCAN    | ----> | Tech Stack |
+------------+       +-----------+       +------+-----+
                                                |
                                                v
                                         +------+-----+
                                         |    PLAN    |
                                         +------+-----+
                                                |
                                                v
                                    +-----------+-----------+
                                    | workflow_1 ... workflow_n |
                                    +-----------+-----------+
                                                |
                                                v  (per workflow)
                                         +------+-----+
                                         | gha-create |  ---> .yml
                                         +------+-----+
                                                |
                                                v
                                         +------+-----+
                                         |  VALIDATE  |
                                         +------+-----+
                                                |
                                                v
                                         +------+-----+
                                         |   REPORT   |
                                         +------------+
```

### Detailed Flow

```
Target Repo path
  |
  v
+------------------+
|  1. RECEIVE      |  Validate target repo and workflow directory
+------------------+
  |
  v
+------------------+
|  2. RESOLVE      |  Fetch SKILL.md for gha-create
+------------------+
  |
  v
+------------------+
|  3. SCAN         |  Analyze repo structure and tech stack
+------------------+
  |
  v
+------------------+
|  4. PLAN         |  Decide which workflows are needed
+------------------+
  |
  v
+------------------+
|  5. GENERATE     |  Invoke gha-create per planned workflow
+------------------+
  |
  v
+------------------+
|  6. VALIDATE     |  Run validation rules per generated workflow
+------------------+
  |
  v
+------------------+
|  7. REPORT       |  Aggregate structured summary
+------------------+
```

## Prerequisites

- Access to the `gha-create` skill definition (SKILL.md)
- A target repository with source code to analyze
- Skill source configured in `config.yaml`

## Orchestration Workflow

### Step 1: RECEIVE

- [ ] Accept `repo_path`: path to the target repository root (required)
- [ ] Verify `repo_path` exists and is a directory
- [ ] Verify `repo_path` looks like a repository root (contains at least some source files or a recognizable project structure)
- [ ] Verify that `.github/workflows/` under `repo_path` either already exists as a directory or can be created (i.e., `.github/` is a directory or does not yet exist)

**Gate**: If `repo_path` is missing, does not exist, or is not a directory, STOP with error.

### Step 2: RESOLVE

- [ ] Read `config.yaml` to get the gha-create skill source, version, and ref
- [ ] Fetch `SKILL.md` for `gha-create` from the configured `source` at the configured `ref`
- [ ] Confirm the skill definition is accessible

**Gate**: If `config.yaml` is missing, `ref` is missing, or SKILL.md cannot be fetched at the configured ref, STOP with error.

### Step 3: SCAN

- [ ] Analyze the target repository to detect:
  - **Languages**: file extensions (`.ts`, `.js`, `.py`, `.go`, `.rs`, `.java`, `.rb`, `.cs`, etc.)
  - **Package managers**: `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `requirements.txt`, `Pipfile`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, `build.gradle`, `*.csproj`
  - **Frameworks**: config files (`next.config.*`, `nuxt.config.*`, `angular.json`, `vite.config.*`, `Dockerfile`, `docker-compose.yml`, `terraform/`, `.tf` files, `serverless.yml`, `k8s/`, `helm/`)
  - **Existing workflows**: `.github/workflows/` contents
  - **Deployment targets**: cloud config files (`samconfig.toml`, `cdk.json`, `app.yaml`, `vercel.json`, `netlify.toml`, `fly.toml`)
  - **Test runners**: config files or package.json scripts (jest, vitest, pytest, go test, cargo test, etc.)
  - **Linters/formatters**: `.eslintrc*`, `.prettierrc*`, `ruff.toml`, `golangci-lint.yml`, `.rubocop.yml`, etc.
  - **Container usage**: `Dockerfile`, `docker-compose.yml`
  - **Security scanning markers**: `SECURITY.md`, dependency lock files suitable for dependabot or renovate
- [ ] Retain scan results in agent context for use in subsequent steps

**Gate**: If scanning detects zero languages and zero package managers (repo appears empty or non-software), STOP with error.

### Step 4: PLAN

- [ ] Based on the scan results, decide which workflows are needed using the decision matrix:

| Workflow Type | Trigger Condition | Description |
|---|---|---|
| `ci` | Any language detected | Lint, test, build for all detected languages |
| `cd` | Deployment target detected | Deploy to detected cloud/platform |
| `docker-build` | Dockerfile detected | Build and optionally push container image |
| `release` | `package.json` or `pyproject.toml` or `Cargo.toml` detected | Automate versioning and release |
| `security` | Any lock file detected | Dependency scanning / CodeQL analysis |

- [ ] For each planned workflow, record:
  - `workflow_id`: short identifier (e.g., `ci`, `cd`, `docker-build`, `release`, `security`)
  - `workflow_filename`: target filename (e.g., `ci.yml`, `cd.yml`)
  - `description`: human-readable purpose
  - `triggers`: which events and paths should trigger it
  - `tech_context`: relevant tech stack details to pass to gha-create
- [ ] If existing workflows are detected in `.github/workflows/`, record them in the plan under `existing_workflows` for reference, but do not modify them
- [ ] For each planned workflow, check if `.github/workflows/{workflow_filename}` already exists — if so, mark that workflow as `skipped_conflict` in the plan
- [ ] Retain the plan in agent context for use in subsequent steps

**Gate**: If planning produces zero workflows (nothing to generate), STOP with error code `no_workflows_planned`.

### Step 5: GENERATE

- [ ] For each planned workflow that is not marked `skipped_conflict`:
  - [ ] Invoke the `gha-create` skill in knowledge-driven mode
  - [ ] Provide the tech context from the plan so gha-create generates an appropriate workflow
  - [ ] The agent passes the gha-create SKILL.md rules as context and requests a workflow following all 8 rules (S1-S4, E1-E4)
  - [ ] Hold the generated workflow content in agent context (do not write to disk yet)
  - [ ] If gha-create fails for a workflow, log the error and continue to the next workflow
  - [ ] Never create a placeholder workflow for a failed generation

**Key detail**: gha-create is invoked once per planned workflow. The agent provides tech stack context and the skill's rules; the runtime produces the hardened YAML. Generated content is held in context until validation passes in Step 6.

### Step 6: VALIDATE

- [ ] For each generated workflow held in context:
  - [ ] Verify the content is non-empty
  - [ ] Verify the content is valid YAML
  - [ ] Check against gha-create's 8-rule checklist:
    - S1: Every `uses:` pinned to full 40-char SHA with version comment
    - S2: Top-level `permissions:` block present
    - S3: OIDC preferred over static secrets (advisory)
    - S4: No dangerous `github.event.*` interpolation in `run:` blocks
    - E1: Path filtering on triggers (advisory)
    - E2: Native caching on `setup-*` actions
    - E3: Concurrency control present
    - E4: Matrix `fail-fast: true` (advisory)
  - [ ] Record per-rule pass/fail/advisory status
  - [ ] If all required checks pass (`validated` or `advisory_only`): create `.github/workflows/` if it does not exist, then write the workflow to `.github/workflows/{workflow_filename}`
  - [ ] If validation fails (`invalid` or `failed`): do NOT write the file to disk
- [ ] A workflow passes validation if all required checks (S1, S2, S4, E2, E3) pass
- [ ] Advisory failures (S3, E1, E4) produce warnings but do not fail the workflow
- [ ] Record which workflows passed, failed, or have advisory warnings

### Step 7: REPORT

- [ ] Count: total planned workflows, generated workflows, passed workflows, failed workflows, skipped workflows
- [ ] Build a machine-readable final summary using the field names and ordering defined in `Summary Output`
- [ ] List any workflows that failed generation or validation with error details
- [ ] List any workflows skipped due to filename conflicts
- [ ] List advisory warnings per workflow
- [ ] If a human-readable recap is included, it must be derived from the machine-readable summary and must not contradict it

**Result classification**: If `failed_workflows` is greater than 0, the run is a partial failure. If `failed_workflows` is 0 but `advisory_warnings` is greater than 0, the run is a partial success.

## Input / Output Contract

### Input

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| repo_path | string | yes | Path to the target repository root |

### Output

The agent writes validated workflow files directly to `.github/workflows/` in the target repository. Only workflows that pass all required validation checks are written to disk. No intermediate metadata files (scan results, plan, validation results) are produced — that data is held in agent context and reported in the final summary.

```
{repo_path}/.github/workflows/
  |- ci.yml                  # Written only if validation passed
  |- docker-build.yml        # Written only if validation passed
  |- security.yml            # Written only if validation passed
```

### Summary Output

The agent must produce a final machine-readable summary object with these top-level fields, in this order:

- `status`: one of `success`, `partial_success`, `partial_failure`, `error`
- `total_workflows`: number of workflows planned
- `generated_workflows`: number of workflows where generation was attempted and produced content (includes `validated`, `advisory_only`, `invalid`, and `failed`; excludes `generation_failed` and `skipped_conflict`)
- `passed_workflows`: number of workflows that passed all required validation checks (includes `validated` and `advisory_only`)
- `failed_workflows`: number of workflows that failed generation or required validation
- `skipped_workflows`: number of workflows skipped due to filename conflict with an existing workflow
- `advisory_warnings`: total advisory warning count across all workflows
- `tech_stack`: condensed summary of detected languages, frameworks, and package managers
- `existing_workflows_found`: number of existing workflows detected in the target repo
- `error_stage`: `null` unless `status` is `error`
- `error_reason`: `null` unless `status` is `error`
- `error_message`: `null` unless `status` is `error`
- `workflow_results`: array of per-workflow result records, sorted by `workflow_id`
- `failed_workflow_details`: array of per-workflow failure records, sorted by `workflow_id`

The `status` field is derived as follows:

- `error`: a hard gate failed before the generation loop
- `partial_failure`: `failed_workflows` is greater than 0
- `partial_success`: `failed_workflows` is 0 but `advisory_warnings` is greater than 0
- `success`: `failed_workflows` is 0 and `advisory_warnings` is 0

Top-level `status` must be derived from the finalized `workflow_results` state machine below when the generation loop is reached. Skipped workflows do not affect the `status` derivation. If a hard gate fails before generation begins, emit `status: error`, set `workflow_results: []`, set all workflow count fields to `0`, and populate `error_stage`, `error_reason`, and `error_message`.

When `status` is `error`, populate:

- `error_stage`: one of `receive`, `resolve`, `scan`, `plan`
- `error_reason`: one of the gate-level reason codes defined in `Reason Code Registry`
- `error_message`: concise human-readable explanation

Each item in `workflow_results` must contain:

- `workflow_id`: short identifier (e.g., `ci`, `cd`, `docker-build`)
- `workflow_filename`: output filename
- `status`: one of `validated`, `advisory_only`, `invalid`, `failed`, `generation_failed`, `skipped_conflict`
- `description`: what the workflow does
- `output_path`: path to the written file (`.github/workflows/{filename}`), or `null` if the workflow was not written to disk
- `validation_passed`: number of required checks passed
- `validation_failed`: number of required checks failed
- `advisory_count`: number of advisory warnings
- `notes`: short explanation of the final classification

`failed_workflow_details` must contain objects with:

- `workflow_id`: short identifier
- `stage`: one of `generate`, `validate`
- `reason`: one of the failure reason codes defined in `Reason Code Registry`
- `message`: concise human-readable explanation

`workflow_results` and `failed_workflow_details` must both be sorted by `workflow_id` for deterministic output.

Illustrative shape:

```yaml
status: partial_failure
total_workflows: 5
generated_workflows: 4
passed_workflows: 3
failed_workflows: 1
skipped_workflows: 1
advisory_warnings: 2
tech_stack:
  languages: [typescript, python]
  package_managers: [npm, pip]
  frameworks: [next.js]
  has_docker: true
  deployment_targets: [vercel]
existing_workflows_found: 1
error_stage: null
error_reason: null
error_message: null
workflow_results:
  - workflow_id: cd
    workflow_filename: cd.yml
    status: failed
    description: "Deploy to Vercel"
    output_path: null
    validation_passed: 3
    validation_failed: 2
    advisory_count: 0
    notes: "S1 and E3 required checks failed"
  - workflow_id: ci
    workflow_filename: ci.yml
    status: skipped_conflict
    description: "Lint, test, and build for TypeScript and Python"
    output_path: null
    validation_passed: 0
    validation_failed: 0
    advisory_count: 0
    notes: "Existing ci.yml detected; skipped to avoid overwrite"
  - workflow_id: docker-build
    workflow_filename: docker-build.yml
    status: validated
    description: "Build and push Docker image"
    output_path: .github/workflows/docker-build.yml
    validation_passed: 5
    validation_failed: 0
    advisory_count: 0
    notes: "All checks passed"
  - workflow_id: release
    workflow_filename: release.yml
    status: advisory_only
    description: "Automate versioning and release"
    output_path: .github/workflows/release.yml
    validation_passed: 5
    validation_failed: 0
    advisory_count: 1
    notes: "All required checks passed; E1 advisory"
  - workflow_id: security
    workflow_filename: security.yml
    status: advisory_only
    description: "Dependency scanning and CodeQL analysis"
    output_path: .github/workflows/security.yml
    validation_passed: 5
    validation_failed: 0
    advisory_count: 1
    notes: "All required checks passed; E4 advisory"
failed_workflow_details:
  - workflow_id: cd
    stage: validate
    reason: validation_check_failed
    message: "S1 SHA pinning and E3 concurrency control checks failed"
```

### Workflow Result State Machine

`workflow_results.status` is the final exclusive classification for each planned workflow.

General invariants:

- Each planned workflow must appear exactly once in `workflow_results`
- Each workflow must end in exactly one final status
- Top-level count fields are derived from `workflow_results`, not computed independently
- `total_workflows` must equal the length of `workflow_results`
- `passed_workflows` must equal the count of `workflow_results` items with `status: validated` plus items with `status: advisory_only`
- `failed_workflows` must equal the count of items with `status: failed` plus items with `status: invalid` plus items with `status: generation_failed`
- `skipped_workflows` must equal the count of items with `status: skipped_conflict`
- `generated_workflows` must equal `total_workflows` minus the count of items with `status: generation_failed` minus `skipped_workflows`
- `passed_workflows + failed_workflows + skipped_workflows` must equal `total_workflows`
- If `status: error`, `workflow_results` must be empty and all workflow count fields must be `0`

Allowed final states:

- `validated`: workflow generated and passed all required validation checks (S1, S2, S4, E2, E3) with no advisory warnings; written to `.github/workflows/`
- `advisory_only`: workflow generated, passed all required checks, but has advisory warnings (S3, E1, E4); written to `.github/workflows/`
- `invalid`: generated content is unusable — either empty or not valid YAML; the 8-rule validation was never reached; not written to disk
- `failed`: workflow generated and is valid YAML but failed one or more required validation checks; not written to disk
- `generation_failed`: workflow could not be generated at all; not written to disk
- `skipped_conflict`: a file with the same name already exists in `.github/workflows/`; generation was not attempted

Per-state field rules:

- `validated`: `output_path` must be non-null; `validation_failed` must be 0; `advisory_count` must be 0
- `advisory_only`: `output_path` must be non-null; `validation_failed` must be 0; `advisory_count` must be greater than 0
- `invalid`: `output_path` must be `null`; `validation_passed`, `validation_failed`, and `advisory_count` must all be 0
- `failed`: `output_path` must be `null`; `validation_failed` must be greater than 0
- `generation_failed`: `output_path` must be `null`; `validation_passed`, `validation_failed`, `advisory_count` must all be 0
- `skipped_conflict`: `output_path` must be `null`; `validation_passed`, `validation_failed`, `advisory_count` must all be 0

Illegal combinations:

- `status: validated` with `validation_failed > 0`
- `status: validated` with `advisory_count > 0`
- `status: validated` with `output_path: null`
- `status: advisory_only` with `advisory_count == 0`
- `status: advisory_only` with `validation_failed > 0`
- `status: advisory_only` with `output_path: null`
- `status: invalid` with non-null `output_path`
- `status: invalid` with `validation_passed > 0` or `validation_failed > 0` or `advisory_count > 0`
- `status: failed` with `validation_failed == 0`
- `status: failed` with non-null `output_path`
- `status: generation_failed` with non-null `output_path`
- `status: skipped_conflict` with non-null `output_path`
- `status: skipped_conflict` with `validation_passed > 0` or `validation_failed > 0` or `advisory_count > 0`
- Any workflow appearing more than once in `workflow_results`
- Any mismatch between top-level counts and the derived counts from `workflow_results`

### Failure Artifact Policy

- Only workflows with status `validated` or `advisory_only` are written to `.github/workflows/`
- Workflows that fail generation or validation are reported in the summary only — they are not written to disk
- The agent must not fabricate cleanup markers or replacement files to hide an upstream failure

## Constitutional Constraints

1. **Controlled writes only**: The agent writes only to `.github/workflows/` in the target repository. It never modifies or deletes existing files anywhere in the repository.
2. **Skill boundary**: Only invoke skills listed in `config.yaml`. Non-skill checks are limited to file existence, syntax parsing, and the gha-create validation checklist.
3. **Fail-safe iteration**: If gha-create fails for one workflow, log and continue. Never abort the entire pipeline for a single workflow failure.
4. **No fabrication**: Do not generate workflow content outside of gha-create. The agent orchestrates; it does not author workflows. All workflow YAML must be produced through the gha-create skill's rules and patterns.
5. **Deterministic output**: Given the same repository state and the same resolved skill refs, the output structure should be identical.
6. **No implicit overwrite**: If a planned workflow filename conflicts with an existing file in `.github/workflows/`, skip it with `skipped_conflict` — never overwrite.
7. **Preserve evidence**: Report all outcomes (passed, failed, skipped) in the summary; do not suppress failures to make the run look clean.
8. **Do not modify existing workflows**: If the target repo already has files in `.github/workflows/`, record them for reference. Conflicting filenames result in `skipped_conflict`; existing files are never altered or deleted.

## Error Handling

### Gate 1: Input Validation (Step 1)

Triggers when `repo_path` is missing, non-existent, or not a directory. The agent stops immediately and reports the input error.

Use one of these `error_reason` values: `missing_repo_path`, `repo_path_not_found`, `repo_path_not_directory`.

### Gate 2: Skill Resolution (Step 2)

Triggers when `config.yaml` is missing, `ref` is missing, or SKILL.md cannot be fetched at the configured ref. The agent stops before any scanning. Reports which skill failed to resolve.

Use one of these `error_reason` values: `skill_config_missing`, `missing_skill_ref`, `skill_definition_unavailable`.

### Gate 3: Scan Validation (Step 3)

Triggers when scanning detects no languages and no package managers. The agent stops before planning and reports the scan failure.

Use one of these `error_reason` values: `empty_scan_results`.

### Gate 4: Plan Validation (Step 4)

Triggers when planning produces zero workflows. The agent stops before generation and reports the plan failure.

Use one of these `error_reason` values: `no_workflows_planned`.

Note: There is no gate at the generation step (Step 5). Individual workflow failures are logged but do not halt the pipeline.

### Reason Code Registry

All `reason` fields are closed enums. Do not invent ad hoc values at runtime.

Gate-level `error_reason` values:

- `missing_repo_path`: `repo_path` was not provided
- `repo_path_not_found`: `repo_path` does not exist
- `repo_path_not_directory`: `repo_path` exists but is not a directory
- `skill_config_missing`: `config.yaml` is missing or unreadable
- `missing_skill_ref`: gha-create entry is missing a `ref`
- `skill_definition_unavailable`: gha-create SKILL.md cannot be resolved
- `empty_scan_results`: scanning detected no languages or package managers
- `no_workflows_planned`: planning produced zero workflows to generate

`failed_workflow_details.reason` values:

- `gha_create_error`: gha-create skill returned an execution failure
- `empty_workflow_output`: generation completed but produced empty content
- `invalid_workflow_yaml`: generated content is not valid YAML
- `validation_check_failed`: workflow failed one or more required validation checks (S1, S2, S4, E2, E3)
- `workflow_name_conflict`: a file with the same name already exists in `.github/workflows/`

Mapping rules:

- If gha-create itself errors during generation, use reason `gha_create_error`, stage `generate`, workflow status `generation_failed`
- If generation completes but the output is empty, use reason `empty_workflow_output`, stage `validate`, workflow status `invalid`
- If the generated content is not valid YAML, use reason `invalid_workflow_yaml`, stage `validate`, workflow status `invalid`
- If the content is valid YAML but fails required validation checks, use reason `validation_check_failed`, stage `validate`, workflow status `failed`
- If the planned filename conflicts with an existing file, use reason `workflow_name_conflict`, stage `generate`, workflow status `skipped_conflict`

## Common Pitfalls

- **Overwriting existing workflows**: The agent must never overwrite files that already exist in `.github/workflows/`. Conflicts are detected during planning and result in `skipped_conflict`.
- **Writing failed workflows to disk**: Only `validated` and `advisory_only` workflows are written to `.github/workflows/`. Failed, invalid, and generation-failed workflows are reported in the summary only.
- **Generating workflows without gha-create rules**: Every workflow must follow all 8 rules (S1-S4, E1-E4). Do not produce naive workflows.
- **Confusing advisory vs required checks**: S3, E1, E4 are advisory. S1, S2, S4, E2, E3 are required. A workflow with only advisory failures is `advisory_only`, not `failed`.
- **Ignoring existing workflows**: If the target repo already has workflows, acknowledge them in the report. Do not generate duplicates for the same purpose without noting the potential overlap.
- **Planning workflows for undetected tech**: Do not plan a Docker workflow if no Dockerfile is found. Stick to what the scan detects.
- **Treating an empty repo as scannable**: If no languages are detected, stop at the scan gate.
- **SHA pinning with stale SHAs**: The gha-create skill's SKILL.md and its SHA_LOOKUP.md reference provide current SHAs. Always consult them rather than using outdated values.
