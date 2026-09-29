# High-Level Architecture & Design (HLD) 🏗️

Meeseek is built around strict separation of concerns, deterministic isolation, and modular abstractions. This document provides the high-level architectural blueprints, networking topology, and component contracts of the system.

---

## 1. System Context & Actor Topology

The system is governed by a strict division of responsibility across five primary actors:

| Actor | Responsibility | Decides |
|---|---|---|
| **Event Source** (GitHub / Jira / Slack) | Ingests the task specification and developer triggers | *What* needs to be built |
| **Omnigent Agent Runtime** | Orchestrates the LLM reasoning session and tool calls | *Which* agent runs and under what policy |
| **Coding Agent** (Claude Code / Gemini Native) | Analyzes code, writes patches, and exercises the app | *How* to write the code |
| **Meeseek Substrate** | Provisions isolated CoW sandboxes and independently certifies results | *Where* it runs and **proof that it ran** |
| **Human Reviewer** | Interacts via live preview URLs and approves PRs | *Whether* the work is accepted |

### High-Level Architectural Flow Diagram

```mermaid
flowchart TD
  subgraph Ingestion ["1. Event Ingestion Layer"]
    GH_PUSH["GitHub Push Event<br/>(Main branch merge)"]
    GH_ISSUE["GitHub Issue / PR Event<br/>(Task / Review Feedback)"]
    JIRA["Jira Cloud Webhook / Poll<br/>(Task / Ticket Trigger)"]
  end

  subgraph MeeseekCP ["2. Meeseek Control Plane (:8099)"]
    API["FastAPI Lease & Webhook Router"]
    GSYNC["GoldenSyncService<br/>(Debounce & Atomic Rebuilds)"]
    CMGR["ConsoleManager<br/>(Task Correlator & State Machine)"]
    LSVC["LeaseService<br/>(Lease Allocator, PortPool & WarmPool)"]
    STORE[("SQLite Store<br/>Leases, Ports, Tasks")]
    NOTARY["Host Notary Engine<br/>(Readiness, Seed SQL & Test Exit Codes)"]
    
    API --> GSYNC
    API --> CMGR
    CMGR --> LSVC
    LSVC <--> STORE
    LSVC --> NOTARY
  end

  subgraph Orchestration ["3. Agent Orchestration Layer (:8000)"]
    OMNI["Omnigent Server (Docker)"]
    WHEEL["Meeseek Sandbox Provider Plugin<br/>(omnigent-community-sandbox-meeseek)"]
    AGENT["Autonomous Coding Agent<br/>(Claude Code / Gemini ADK)"]
    
    OMNI --> WHEEL
    OMNI --> AGENT
  end

  subgraph Substrate ["4. Ephemeral Host Substrate (XFS Reflink / APFS)"]
    PROV["ComposeProvider (WorkspaceProvider)"]
    GOLDEN[("Golden Image Baseline<br/>Pre-built images, warm DB")]
    WS1["Workspace: ws-TASK-101<br/>Docker Compose Stack (:18001)"]
    WS2["Workspace: ws-TASK-102<br/>Docker Compose Stack (:18002)"]
    
    PROV -->|CoW Strike / Destroy| WS1
    PROV -->|CoW Strike / Destroy| WS2
    GOLDEN -.->|Reflink Clone| WS1
    GOLDEN -.->|Reflink Clone| WS2
  end

  subgraph Delivery ["5. Delivery & Human Review"]
    PR["GitHub Draft PR<br/>(PR.md Evidence Bundle attached)"]
    PREVIEW["Clickable Live HTTPS Preview<br/>(https://p18001.preview.domain.com)"]
  end

  %% Ingress connections
  GH_PUSH -->|HMAC Verified POST| API
  GH_ISSUE -->|Webhook / API| API
  JIRA -->|Webhook / Poll| API

  %% Orchestration connections
  CMGR -->|POST /v1/sessions| OMNI
  WHEEL -->|POST /leases, /exec| LSVC
  AGENT -->|In-Sandbox Terminal Exec| WS1

  %% Execution & Notary
  LSVC --> PROV
  NOTARY -->|Health Probes & Test Proofs| WS1
  NOTARY -->|git push & gh pr create| PR
  WS1 -.->|Routed via Caddy| PREVIEW
```

---

## 2. Network & Port Isolation Architecture

A core technical hurdle in multi-agent sandboxing is preventing **port and networking collisions** on the host machine. If two workspaces run simultaneously, they cannot both bind host port `5432` for PostgreSQL or `5173` for the frontend.

Meeseek solves this with **strict Docker Compose project namespacing**, **automatic internal port stripping**, and **dedicated preview port allocation**:

```mermaid
flowchart LR
  subgraph PublicInternet ["Public Internet / External Network"]
    DEV["Developer / Reviewer"]
    GH_NET["GitHub Webhooks"]
  end

  subgraph HostVM ["Host Machine / Cloud VM Boundary (EC2 / GCE)"]
    CADDY["Caddy Reverse Proxy<br/>Ports :80 / :443 (Auto Let's Encrypt TLS)"]
    
    subgraph CoreServices ["Core Control Plane & Orchestrator"]
      OMNI_SRV["Omnigent Server<br/>Container Port :8000"]
      MEESEEK_API["Meeseek Control Plane<br/>Host Process Port :8099"]
    end

    subgraph WS1 ["Workspace ws-TASK-101 (Docker Compose)"]
      WS1_FE["frontend: Vite Dev Server<br/>Internal :5173 ──► Host :18001"]
      WS1_BE["backend: API Server<br/>Internal :8000 (No host port)"]
      WS1_DB[("db: PostgreSQL<br/>Internal :5432 (No host port)")]
    end

    subgraph WS2 ["Workspace ws-TASK-102 (Docker Compose)"]
      WS2_FE["frontend: Vite Dev Server<br/>Internal :5173 ──► Host :18002"]
      WS2_BE["backend: API Server<br/>Internal :8000 (No host port)"]
      WS2_DB[("db: PostgreSQL<br/>Internal :5432 (No host port)")]
    end
  end

  DEV -->|https://preview.domain.com| CADDY
  GH_NET -->|https://api.domain.com/webhooks/github| CADDY
  
  CADDY -->|Reverse Proxy /api| MEESEEK_API
  CADDY -->|Reverse Proxy /omnigent| OMNI_SRV
  CADDY -->|Reverse Proxy p18001.*| WS1_FE
  CADDY -->|Reverse Proxy p18002.*| WS2_FE

  OMNI_SRV <-->|Loopback HTTP :8099| MEESEEK_API
  MEESEEK_API -->|docker compose exec| WS1
  MEESEEK_API -->|docker compose exec| WS2
```

### Port Stripping Mechanics
- **Compose Override Injection:** When a workspace strikes, Meeseek generates an on-the-fly `compose.ws.yaml` utilizing Compose $\ge 2.24$ YAML merge keys (`ports: !reset []`).
- **Internal Ports Stripped:** Databases, caches, and private APIs are isolated within the workspace's private Docker bridge network (`ws-<id>_default`).
- **Single Preview Exposure:** Only the designated entrypoint service (e.g. `frontend`) is re-exposed on an allocated preview port (`ports: !override ["127.0.0.1:18001:5173"]`).

---

## 3. Control Plane Internal Modular Architecture

The Meeseek Control Plane (`meeseek/control-plane/`) is a high-performance, single-worker FastAPI application designed with clean hexagonal seams:

```mermaid
flowchart TD
  subgraph HTTP_Layer ["HTTP / REST Layer (api.py & console/routes.py)"]
    R_LEASES["/leases API<br/>(acquire, get, finalize, release, exec)"]
    R_WEBHOOKS["/webhooks API<br/>(github, jira, slack)"]
    R_OPS["/ops Operator Console<br/>(state, warm pool, golden sync)"]
  end

  subgraph CoreLogic ["Core Domain Logic"]
    CMGR["ConsoleManager<br/>Task lifecycle, correlation, human feedback"]
    GSYNC["GoldenSyncService<br/>Debouncing, push coalescing, atomic golden build"]
    LSVC["LeaseService<br/>State machine: PENDING → READY → FINALIZED → RELEASED"]
  end

  subgraph ResourceManagers ["Resource & Pool Managers"]
    PPOOL["PortPool<br/>Bitmap/range allocator for preview ports"]
    WPOOL["PoolManager<br/>Pre-struck idle workspace pool (50ms branch cut)"]
    REAPER["LeaseReaper<br/>Background TTL enforcement & cleanup"]
  end

  subgraph StateStorage ["State & Persistence"]
    DB[("SQLite Database<br/>WAL-mode, ACID Lease & Task tables")]
  end

  subgraph SubstrateSeam ["Substrate Abstraction (WorkspaceProvider)"]
    WP_IF["WorkspaceProvider (Interface)<br/>acquire(), release(), exec(), finalize(), open_pr()"]
    C_PROV["ComposeProvider (Production)<br/>Invokes strike.sh, destroy.sh, docker compose"]
    F_PROV["FakeProvider (Testing)<br/>Pure in-memory mock for 100+ unit tests"]
  end

  R_LEASES --> LSVC
  R_WEBHOOKS --> GSYNC
  R_WEBHOOKS --> CMGR
  R_OPS --> CMGR
  R_OPS --> GSYNC

  CMGR --> LSVC
  GSYNC --> C_PROV
  LSVC --> PPOOL
  LSVC --> WPOOL
  LSVC --> REAPER
  LSVC <--> DB
  LSVC --> WP_IF

  WP_IF --> C_PROV
  WP_IF --> F_PROV
```

### The `WorkspaceProvider` Seam
The control plane never makes raw shell calls directly inside HTTP endpoints. All operations invoke the `WorkspaceProvider` interface (`holodeck/providers/base.py`). This guarantees that the control plane can promote from single-host Docker Compose (`ComposeProvider`) to distributed container substrates without altering API routes or orchestration drivers.

---

## 4. Copy-on-Write (CoW) Filesystem Reflink Blueprint

Traditional CI/CD systems clone repositories and rebuild Docker images for every test runner. In contrast, Meeseek uses **OS-level reflink pointers**, achieving near-instant workspace duplication:

```mermaid
flowchart TD
  subgraph PhysicalDisk ["Physical Disk Storage Blocks (XFS reflink=1 / APFS)"]
    BLOCKS_BASE["Shared Physical Disk Blocks<br/>• Checked-out Repos & Dependencies<br/>• Pre-built Docker Image Caches<br/>• Migrated PostgreSQL pgdata Files (100MB–2GB)"]
    BLOCKS_WS1["Delta Blocks (ws-TASK-101)<br/>• Modified src/App.tsx<br/>• New Session Log Files"]
    BLOCKS_WS2["Delta Blocks (ws-TASK-102)<br/>• Modified src/api/routes.py<br/>• Temporary DB Query Rows"]
  end

  subgraph FileSystemView ["Filesystem Directory Tree View"]
    GOLDEN_DIR["Golden Master Directory<br/>/data/golden/full-stack-application/<br/>(Read-Only Reference)"]
    WS1_DIR["Workspace Directory 1<br/>/data/workspaces/ws-task-101/<br/>(Active Agent 1)"]
    WS2_DIR["Workspace Directory 2<br/>/data/workspaces/ws-task-102/<br/>(Active Agent 2)"]
  end

  GOLDEN_DIR ===> BLOCKS_BASE
  WS1_DIR -.->|Reflink (Shared Blocks)| BLOCKS_BASE
  WS1_DIR ===>|Copy-on-Write (Only Changes)| BLOCKS_WS1
  WS2_DIR -.->|Reflink (Shared Blocks)| BLOCKS_BASE
  WS2_DIR ===>|Copy-on-Write (Only Changes)| BLOCKS_WS2
```

### Storage Efficiency & Reflink Performance
- **Time Complexity:** Duplicating a 10 GB golden image directory takes **$<200$ milliseconds** of kernel metadata manipulation (`cp --reflink=always -R golden/ ws-1/`).
- **Disk Allocation:** Zero additional disk blocks are consumed when a workspace is created. Physical disk blocks are only allocated when the agent writes to code files or the database logs new WAL records.
- **Fail-Safe Integrity:** Standard `ext4` filesystems do not support reflinks. The Meeseek startup scripts actively verify `reflink` capability at boot and fail fast if a non-CoW volume is detected.

