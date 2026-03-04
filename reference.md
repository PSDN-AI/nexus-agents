# References

Curated external resources relevant to AI agent engineering and operations.

## Harness Engineering

| Resource | Summary |
|----------|---------|
| [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) | OpenAI's methodology for agent-first development — covers six core practices (AGENTS.md as map, app legibility, architectural invariants, automated cleanup, capability diagnosis, repo-as-knowledge-base), three pillars (context engineering, architectural constraints, entropy management), and lessons from building a 1M-line codebase with zero manually-written code. |

## Agent Architecture

| Resource | Summary |
|----------|---------|
| [How I built an Autonomous AI Agent team that runs 24/7](https://x.com/Saboo_Shubham_/status/2022014147450614038) | Shubham Saboo's multi-agent coordination case study — 6 specialized agents collaborate via filesystem, automating a content operations pipeline (research → draft → publish → review → triage), demonstrating single-responsibility agents with pipeline orchestration in practice. |
| [Agentic Software Engineering](https://x.com/ashpreetbedi/status/2028176285575594465) | Ashpreet Bedi's framework for production agent systems — distinguishes "building agents" from "running them in production" with a Build→Serve→Connect lifecycle and six pillars (Durability, Isolation, Governance, Persistence, Scale, Composability). Key insight: agents are distributed systems requiring checkpoint/resume, per-user isolation, layered authority (auto-execute vs human-approval vs admin-approval), and composability as services. Includes practical examples using Agno framework with governance patterns (UserFeedbackTools, tiered tool approval). |

## Context Engineering

_No entries yet._

## DevOps and Infrastructure

_No entries yet._

## Industry Standards

_No entries yet._
