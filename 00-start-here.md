# Start Here 🧭

Welcome to **Meeseek**—an autonomous agentic engineering platform and ephemeral execution substrate built for AI software engineers.

---

## What is Meeseek in 5 Sentences

1. **The Core Philosophy:** Meeseek gives an autonomous AI coding agent a real, isolated, and pre-seeded full-stack environment to work in—complete with running databases, active microservices, and live hot-reloaded frontends—instead of an unverified code diff.
2. **Instant Copy-on-Write (CoW) Sandboxing:** By leveraging OS-level Copy-on-Write filesystem snapshots (Linux XFS `reflink` or macOS APFS `clonefile`), Meeseek summons a fully booted, warm multi-container stack in **under 15–30 seconds** (or **~50ms from a warm pool**) instead of paying a 15-minute cold build and database seeding penalty.
3. **Impartial Host-Side Notary:** Following the golden principle *"The actor that does the work never certifies it,"* Meeseek—not the LLM—independently executes tests, verifies database seed invariants via SQL, confirms HTTP readiness, and captures genuine git diffs outside of agent control.
4. **Zero-Friction Review with Live Previews:** Every task produces not only a clean pull request with certified evidence (`PR.md`), but also an immediate, clickable HTTPS preview URL so human reviewers can test UI and backend changes directly in the browser without stashing code or running local stacks.
5. **Like a Meeseeks, It Ceases to Exist:** Once the task is completed and verified, the entire workspace self-destructs. Zero environment drift, zero state contamination, and zero lingering containers.

---

## Directory & Ecosystem Map

The project is structured into three clean, decoupled layers:

```
hackathon_project/
├── meeseek/             # Core substrate engine: FastAPI lease service, CoW strikes,
│                        #   port allocator, warm pool manager, and host-side Notary.
├── omnigent-deploy/     # Standalone agent orchestration server: Docker Compose,
│                        #   PostgreSQL, Caddy automated TLS reverse proxy, and provider wheel.
└── docs/                # Architectural blueprints, workflow lifecycles, and runbooks.
```

---

## Recommended Reading Order

Whether you are a hackathon judge, a systems engineer, or a product lead, follow this reading roadmap:

| Step | Document | Purpose & Key Takeaways |
|:---:|---|---|
| **1** | [**`01-project-overview.md`**](01-project-overview.md) | **The Big Idea & Vision:** The Plausible-Diff trap, cold-start bottlenecks, developer augmentation metrics, one-shot vs. complex task execution, and core business value. |
| **2** | [**`02-architecture.md`**](02-architecture.md) | **System Topology & HLD:** 4 comprehensive Mermaid diagrams covering system context, network/port isolation, internal modular control plane, and CoW storage layout. |
| **3** | [**`03-workflow-lifecycle.md`**](03-workflow-lifecycle.md) | **The Ground-Truth Lifecycle:** Sequence diagram traced directly from the codebase—from event trigger to warm pool claim, interactive elicitation, Notary certification, and teardown. |
| **4** | [**`04-event-ingestion-and-webhooks.md`**](04-event-ingestion-and-webhooks.md) | **Webhook Engine:** GitHub push debouncing (automated golden sync), issue triggers, PR review comment self-healing loops, and HMAC security. |
| **5** | [**`05-context-engineering-and-guardrails.md`**](05-context-engineering-and-guardrails.md) | **Quality Engineering:** Eliminating sloppy code through 3 tiers of defense: upfront rules, in-sandbox AST/linting checks, and impartial host-side Notary proofs. |
| **6** | [**`06-deployment-and-scaling.md`**](06-deployment-and-scaling.md) | **Operations & Scaling Roadmap:** Local dev bring-up, single-VM cloud deployment (AWS/GCP), Caddy TLS, and the enterprise Kubernetes scaling blueprint. |
| **7** | [**`07-glossary.md`**](07-glossary.md) | **Terminology Reference:** Quick definitions for Golden Image, Strike, CoW Reflink, Notary, Lease, Warm Pool, and Port Stripping. |
| **8** | [**`11-pitch-deck-presentation.md`**](11-pitch-deck-presentation.md) | **Pitch Deck & Whitepaper:** Hackathon executive presentation, market positioning, problem slides, and competitive differentiation. |
