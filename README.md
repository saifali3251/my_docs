# Meeseek Platform Documentation 📚

> *"I'm Mr. Meeseeks, look at me!"*  
> The comprehensive architectural blueprints, workflow lifecycles, and operational guides for **Meeseek**—an autonomous agentic engineering platform and ephemeral execution substrate.

---

## Overview

This documentation suite provides a complete, end-to-end technical reference for the Meeseek platform. It details how Meeseek transforms ambiguous tickets and bug reports into verified, certified GitHub Pull Requests with live HTTPS preview URLs in under 3 minutes—bypassing the traditional 15-minute cold build and database seeding bottleneck.

---

## Documentation Sitemap & Reading Guide

| Document | Title | Purpose & Audience |
|:---:|---|---|
| [**`00-start-here.md`**](00-start-here.md) | **Start Here 🧭** | Project index, 5-sentence executive elevator pitch, and high-level ecosystem map across `meeseek/` and `omnigent-deploy/`. |
| [**`01-project-overview.md`**](01-project-overview.md) | **Project Overview & The Plausible-Diff Trap 📦** | Core problem statement, cold-start bottlenecks, the Meeseek philosophy, hard metrics (reducing 2–4 hr turnaround to <3 mins), and one-shot vs. complex task execution. |
| [**`02-architecture.md`**](02-architecture.md) | **High-Level Design & System Topology 🏗️** | 4 comprehensive Mermaid diagrams covering system context, network/port isolation, internal modular control plane, and Copy-on-Write (CoW) filesystem reflink layout. |
| [**`03-workflow-lifecycle.md`**](03-workflow-lifecycle.md) | **Workflow Lifecycle: End-to-End Task Journey 🔄** | Code-grounded sequence diagram tracing a task from event trigger through warm pool claiming, interactive elicitation questions, host Notary certification, and self-destruction. |
| [**`04-event-ingestion-and-webhooks.md`**](04-event-ingestion-and-webhooks.md) | **Event Ingestion & Webhooks Architecture ⚡** | Technical specification for GitHub and Jira webhooks: timing-safe HMAC SHA-256 verification, merge-storm push debouncing (`golden_sync.py`), and PR review comment self-healing loops. |
| [**`05-context-engineering-and-guardrails.md`**](05-context-engineering-and-guardrails.md) | **Context Engineering & Coding Guardrails 🛡️** | Eliminating sloppy code across 3 tiers of defense: upfront rules injection (`CLAUDE.md`), in-sandbox diff-scoped linting and AST Delta Tracking (catching deleted tests), and impartial host-side Notary proofs. |
| [**`06-deployment-and-scaling.md`**](06-deployment-and-scaling.md) | **Deployment & Scaling Guide 🚀** | Operational runbook for local development (Docker Desktop), single-VM cloud deployment with automated Caddy TLS, and the enterprise Kubernetes scaling roadmap. |
| [**`07-glossary.md`**](07-glossary.md) | **Glossary & Concepts 📖** | Plain-language definitions of core concepts: Golden Image, CoW Reflink, Strike, Host Notary, Evidence Bundle, Lease, Warm Pool, and Port Stripping. |
| [**`08-ci-failure-remediation-and-pr-loops.md`**](08-ci-failure-remediation-and-pr-loops.md) | **CI Failure Remediation & PR Review Loops 🔄** | Technical architecture for remote CI failure webhooks, self-healing retries, and the production RFC for PR review comment loops and debouncing. |
| [**`09-jira-commands-and-developer-guide.md`**](09-jira-commands-and-developer-guide.md) | **Jira Commands & Developer Guide 📘** | Comprehensive user manual for interacting with Meeseek: labels, slash commands, multi-repo targeting (`Repo:`, `Base:`), plan-only mode, and Host Notary verification. |
| [**`10-console-ui-architecture-and-plan.md`**](10-console-ui-architecture-and-plan.md) | **React Console UI Architecture & Plan 🖥️** | Architectural specification and phased plan for migrating the operator console and onboarding wizard to React, Vite, Tailwind CSS, with interactive workflow DAGs. |
| [**`11-pitch-deck-presentation.md`**](11-pitch-deck-presentation.md) | **Executive Pitch Deck & Product Whitepaper 🎯** | Hackathon presentation slides, product positioning, business value proposition, and strategic competitive differentiation. |

---

## Core Innovations at a Glance

1. **Instant Copy-on-Write (CoW) Strikes:** Uses OS-level filesystem cloning (`reflink=1` on Linux XFS/btrfs or APFS `clonefile` on macOS) to duplicate gigabytes of code, dependencies, and pre-seeded database fixtures in **under 300ms**, booting full-stack workspaces in **~12–30 seconds** (or **~50ms from a warm pool**).
2. **Impartial Host-Side Notary:** *"The actor that does the work never certifies it."* Meeseek executes container readiness checks, runs authoritative SQL seed-verification queries, and records authentic test exit codes outside of agent control before opening a PR.
3. **Hot-Reloaded Live Previews:** Every workspace maps its web frontend to a dynamic preview port routed through Caddy, attaching a clickable HTTPS URL (`https://p18001.preview.domain.com`) to the PR so reviewers can click-test UI changes immediately.
4. **Interactive Elicitation Loop:** When facing ambiguous requirements, the agent pauses, suspends token consumption, and asks clarifying questions directly in the GitHub/Jira/Slack thread, resuming instantly once a human responds.
5. **Like a Meeseeks, It Ceases to Exist:** Once verified and merged, the workspace containers and CoW clone are deleted completely. Zero drift, zero lingering state.

---

## Recommended Pathways

- **For Hackathon Judges & Product Leads:**  
  Start with [**`00-start-here.md`**](00-start-here.md) $\to$ [**`01-project-overview.md`**](01-project-overview.md) $\to$ [**`11-pitch-deck-presentation.md`**](11-pitch-deck-presentation.md).
- **For Software Architects & Core Contributors:**  
  Read [**`02-architecture.md`**](02-architecture.md) $\to$ [**`03-workflow-lifecycle.md`**](03-workflow-lifecycle.md) $\to$ [**`04-event-ingestion-and-webhooks.md`**](04-event-ingestion-and-webhooks.md) $\to$ [**`05-context-engineering-and-guardrails.md`**](05-context-engineering-and-guardrails.md).
- **For DevOps & Platform Engineers:**  
  Review [**`06-deployment-and-scaling.md`**](06-deployment-and-scaling.md) with [**`07-glossary.md`**](07-glossary.md).

