# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

`nexus-agents` is a **knowledge-only** repository — no scripts, no builds, no tests, no CI. There is nothing to compile or run. Each agent is a set of markdown and YAML files consumed by an AI runtime.

Agents orchestrate multi-step workflows by chaining skills from the companion repo [nexus-skills](https://github.com/PSDN-AI/nexus-skills). Skills are atomic, single-purpose units; agents compose them into pipelines.

## Repository Structure

```
agents/{agent-name}/
  ├── README.md        # Human-facing usage and invocation guide
  ├── AGENT.md         # Orchestration contract: pipeline steps, I/O contracts, constraints, error handling
  └── config.yaml      # Skill dependencies with version, ref, and source location
reference.md           # Curated reference material (AI agent engineering practices, in Chinese)
```

## Agent Anatomy

- **AGENT.md** is the primary contract. It uses YAML frontmatter (name, description, license, metadata) followed by a detailed step-by-step workflow with explicit gates, I/O contracts, constitutional constraints, and error handling.
- **config.yaml** declares skill dependencies. Each skill entry has `name`, `version`, `ref`, `source`, and `path`. During development, `ref` tracks `main`; for releases, pin to an immutable tag or commit SHA.
- **README.md** provides human-oriented usage instructions including example prompts for invoking the agent from Claude Code.

## Adding a New Agent

1. Create `agents/{agent-name}/` with `README.md`, `AGENT.md`, and `config.yaml`
2. AGENT.md must include: YAML frontmatter, step-by-step workflow with numbered gates, I/O contract table, constitutional constraints, error handling section, and common pitfalls
3. config.yaml must list all skill dependencies with `ref` fields
4. Update the "Available Agents" table in the root `README.md`

## Key Conventions

- Agents **never contain executable code** — no scripts, no GitHub Actions
- Agents invoke only skills listed in their `config.yaml` — no inline task fabrication
- Pipeline failures in one domain should not abort the entire pipeline (fail-safe iteration)
- The input PRD is always read-only; all output goes to a separate directory
- Skills come from `nexus-skills` repo; agents from this repo. Keep the boundary clean.
- `reference.md` at the root is a curated collection of external references (primarily in Chinese) — append new entries under the appropriate topic heading following the existing format
