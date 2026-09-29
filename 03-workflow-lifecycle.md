# Workflow Lifecycle: End-to-End Task Journey 🔄

This document traces the complete lifecycle of tasks and environments through Meeseek, derived directly from the control-plane source code (`holodeck/console/manager.py`, `holodeck/api.py`, `holodeck/golden_sync.py`, and `holodeck/service.py`).

---

## 1. Golden Build vs. Strike (Two Critical Cadences)

A fundamental design aspect of Meeseek is separating background environment baking from per-task workspace provisioning:

| Dimension | Golden Build Lifecycle | Task Strike Lifecycle |
|---|---|---|
| **Trigger** | GitHub `push` to main branch / Scheduled cron / Manual `/ops/golden/rebuild` | GitHub Issue label / Jira ticket webhook / API `POST /leases` |
| **Frequency** | Infrequent (nightly or on merged PRs) | Frequent (on every task or bug report) |
| **Operations** | `git clone` from scratch, full image builds, database migrations, offline seed execution | Copy-on-Write (`reflink`) clone of the golden tree, git branch cut, container boot |
| **Execution Time** | Minutes (3–10 minutes depending on stack depth) | **~12–30 seconds** (cold strike) / **~50 milliseconds** (warm pool claim) |
| **Disk Impact** | Creates the durable base snapshot at `$HOLO_GOLDEN` | Ephemeral delta blocks only; completely deleted upon task completion |

---

## 2. End-to-End Sequence Diagram

The following sequence diagram details the full progression of a task, including the interactive clarification loop and host-side Notary certification:

```mermaid
sequenceDiagram
    autonumber
    actor Human as Developer / Reviewer
    participant Trigger as Event Source (GitHub/Jira)
    participant CMgr as ConsoleManager
    participant Omni as Omnigent Runtime (:8000)
    participant LeaseAPI as Meeseek LeaseService (:8099)
    participant Prov as ComposeProvider
    participant Substrate as Workspace (ws-TASK-101)
    participant Notary as Host Notary Engine
    participant GitHub as GitHub API

    Note over Human,GitHub: Phase 1: Ingestion & Workspace Strike
    Human->>Trigger: Label Issue or Ticket ("meeseek")
    Trigger->>CMgr: Inbound Webhook Event (ticket, app, prompt, target_repo)
    CMgr->>CMgr: Validate target_repo against manifest
    CMgr->>Omni: POST /v1/sessions (host_type="managed")
    Omni->>LeaseAPI: POST /leases (acquire workspace)
    alt Warm Pool Slot Available
        LeaseAPI->>Prov: Claim pre-struck slot (~50ms branch cut)
    else Cold Strike Required
        LeaseAPI->>Prov: CoW Reflink Clone from Golden Image (~15–30s)
        Prov->>Substrate: docker compose -p ws-TASK-101 up -d (Ports Stripped)
    end
    LeaseAPI-->>Omni: Return Lease Handle (preview_port=18001, token)
    Omni->>Substrate: Exec omnigent host daemon & Claude/Gemini Agent

    Note over Substrate,Human: Phase 2: Agent Execution & Interactive Clarification
    loop Coding & Investigation
        Substrate->>Substrate: Agent inspects code, edits files, runs in-sandbox linter
    end
    opt Ambiguity Encountered (Elicitation)
        Substrate->>Omni: Agent raises question / clarification need
        Omni->>CMgr: Task state updated to "waiting-input"
        CMgr->>Trigger: Post comment in ticket thread with question
        Human->>Trigger: Reply in thread ("yes, proceed with Option B")
        Trigger->>CMgr: Inbound Reply Event
        CMgr->>CMgr: parse_verdict() resolves approval or freeform guidance
        CMgr->>Omni: answer() / reiterate() sends guidance to agent
        Omni->>Substrate: Agent resumes execution
    end

    Note over Substrate,GitHub: Phase 3: Host Notary Certification & PR Delivery
    Substrate->>Omni: Agent completes edits and declares readiness
    Omni->>LeaseAPI: POST /leases/{id}/finalize
    LeaseAPI->>Notary: Execute Host-Side Independent Validation
    Notary->>Substrate: 1. Probe HTTP readiness (HOLO_READINESS_PATH)
    Notary->>Substrate: 2. Execute SQL seed-proof query (HOLO_SEED_PROOF_SQL)
    Notary->>Substrate: 3. Run test command (HOLO_TEST_CMD) outside agent control
    Notary->>Substrate: 4. Capture real git diff
    
    alt Test Failed (Exit != 0)
        Notary-->>Substrate: Feed stderr/failures back into agent (Self-Healing Loop)
    else Certified Green (Exit == 0 & Proof Valid)
        Notary->>GitHub: Push branch agent/TASK-101 & Open Draft PR
        Note right of GitHub: PR body includes PR.md Evidence Bundle<br/>+ Clickable Live Preview: https://p18001.preview.domain.com
        Notary-->>Trigger: Post PR link and preview URL in thread
    end

    Note over Human,Substrate: Phase 4: Human Review & Teardown
    Human->>Substrate: Interactively test UI via Live Preview URL
    Human->>GitHub: Approve and Merge PR
    Trigger->>CMgr: Ticket closed / release signal
    CMgr->>LeaseAPI: DELETE /leases/{id}
    LeaseAPI->>Prov: Teardown Workspace
    Prov->>Substrate: docker compose down -v && rm -rf workspace_dir
    Note over Substrate: Workspace destroyed completely. Poof!
```

---

## 3. The State Machine & Task Lifecycle

The `ConsoleManager` and `LeaseService` maintain strict, synchronized state transitions:

```mermaid
stateDiagram-v2
    [*] --> PENDING: Inbound Trigger (trigger())
    PENDING --> READY: Workspace Struck & Containers Healthy
    READY --> WAITING_INPUT: Agent raises clarifying question (Elicitation)
    WAITING_INPUT --> READY: Human answers in thread (answer() / reiterate())
    READY --> FINALIZING: Agent signals completion (finalize())
    FINALIZING --> READY: Notary detected test failure (Self-healing retry)
    FINALIZING --> FINALIZED: Notary Certified Green (PR Created & Evidence Attached)
    FINALIZED --> RELEASED: Task closed / Lease TTL expired (release())
    PENDING --> FAILED: Substrate boot or capacity error
    READY --> FAILED: Hard crash or unrecoverable error
    RELEASED --> [*]: Reflink directory deleted & ports freed
    FAILED --> [*]
```

### State Definitions
- **`PENDING` / `provisioning`**: Workspace is being claimed from the warm pool or cloned via CoW reflink. Docker containers are booting and health-checks are pending.
- **`READY`**: Workspace containers are healthy and answering HTTP readiness probes. The agent process is active and running commands inside the sandbox.
- **`WAITING-INPUT`**: The agent has paused execution because requirements or edge-cases were ambiguous. An elicitation question has been forwarded to the human developer.
- **`FINALIZING`**: The host-side Notary is actively probing the workspace, executing seed-verification queries, and re-running test suites.
- **`FINALIZED`**: All tests passed with authentic exit code `0`, seed rows were verified, a draft pull request was opened on GitHub with evidence attached, and the live preview URL was published.
- **`RELEASED`**: Containers have been halted, networks deleted, and the CoW disk directory removed.

---

## 4. Background Golden Sync Lifecycle

In addition to per-task execution, Meeseek manages automated background golden image synchronization via `holodeck/golden_sync.py`:

```
GitHub Push to main
       │
       ▼  POST /webhooks/github (HMAC SHA-256 verified)
┌────────────────────────────────────────────────────────┐
│  GoldenSyncService (Debounce & Coalescing Engine)      │
│  • Pushes arriving within 30s debounce window coalesce │
│  • Prevents overlapping expensive golden builds       │
└────────────────────────────────────────────────────────┘
       │
       ▼  Trigger atomic golden-build.sh in background
┌────────────────────────────────────────────────────────┐
│  Atomic Staging Directory                              │
│  1. Fresh git clone of all composite repos             │
│  2. Build updated container images                     │
│  3. Run Alembic/Prisma migrations                      │
│  4. Seed database fixtures                             │
│  5. Validate readiness endpoint                        │
└────────────────────────────────────────────────────────┘
       │
       ▼  Atomic Directory Swap
┌────────────────────────────────────────────────────────┐
│  Golden Master Active ($HOLO_GOLDEN)                   │
│  • Sub-second atomic rename replaces old golden image   │
│  • PoolManager discards stale warm slots and strikes    │
│    new warm slots from the updated golden image        │
└────────────────────────────────────────────────────────┘
```

