# Context Engineering & Coding Guardrails: Eliminating Sloppy Code 🛡️

This document provides the definitive architectural blueprint for **Context Engineering, Coding Guardrails, and Impartial Host Verification** in the Meeseek Platform. It captures the design decisions, edge-case mitigations, and execution phases necessary to transform raw agentic code generation into enterprise-grade, certified software delivery.

---

## 1. Executive Summary & Problem Statement

Autonomous coding agents (e.g. Devin, Copilot Workspace, raw LLM loops) frequently suffer from the **"Sloppy Code" Dilemma**:
* **Context Blindness**: Agents enter a codebase without knowing architectural conventions, using `print()` instead of the project's structured logger or writing raw SQL instead of using SQLAlchemy / Prisma models.
* **Typing & Formatting Drift**: Inconsistent styling, missing type annotations in strict TypeScript/Python projects, and arbitrary linting violations that break downstream CI.
* **Hallucinated Assertions & Fake Success**: Agents report *"Everything passed successfully"* when they never actually executed the tests, or worse, they delete failing assertions to force tests to pass.
* **The "Red CI" PR Problem**: Traditional AI tools open a PR on GitHub *before* testing. GitHub Actions runs in the cloud, fails with a red `❌`, and forces human engineers to debug basic formatting or typing mistakes.

### The Meeseek Solution: The Staff Engineer Multiplier
Meeseek replaces blind prompting with an **In-Sandbox Verified Architecture**:
1. **Upfront Context Engineering**: Pre-grounding the agent with repository rules, schema outlines, and baked-in toolchains before code is written.
2. **In-Sandbox Coding Guardrails**: Automated linting, type-checking, and syntax validation inside the container during execution.
3. **Impartial Host Notary Certification**: The host OS (not the agent) executes tests, queries database state, and verifies exit code `0` before any PR can be opened.
4. **Self-Healing Feedback Loops**: Deterministic circuit breakers that repair CI failures or human code review comments without infinite loops.

```
┌────────────────────────────────────────────────────────────────────────┐
│  Tier 1: Upfront Context Engineering ("Don't write bad code")         │
│  - CLAUDE.md / AGENT_RULES.md injected at workspace boot               │
│  - AST & Schema Grounding (compact route & model definitions)         │
│  - Tooling and dependency toolchains pre-baked in Golden Image        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Agent writes code in sandbox
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Tier 2: In-Sandbox Coding Guardrails ("Clean code during execution")   │
│  - Automated diff-scoped linting (Ruff / ESLint / Biome)              │
│  - Strict type checking (MyPy / TypeScript tsc --noEmit)              │
│  - AST Delta Tracking (detects deleted tests & stripped auth guards)  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Agent declares ready
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Tier 3: The Impartial Host Notary ("Verify before PR is opened")      │
│  - Host executes HOLO_TEST_CMD independently (Exit 0 mandatory)       │
│  - HTTP readiness probe & Database seed row integrity proof           │
│  - If failed: Errors fed back to agent for in-sandbox self-correction │
│  - Only opened on GitHub when 100% certified passing                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Upfront Context Engineering: Setting Boundaries

Rather than sending 50,000 lines of raw source code to the model, Meeseek uses targeted, high-density context injection.

### 2.1 Native Rule Files (`CLAUDE.md` / `AGENT_RULES.md`)
Claude Code, Antigravity, and Gemini agent harnesses natively ingest a markdown rules file placed at the repository root.

**Standard `CLAUDE.md` Structure:**
```markdown
# Repository Conventions & Guardrails

## Tech Stack & Architecture
- Backend: FastAPI, SQLAlchemy 2.0, Pydantic v2, Alembic migrations.
- Frontend: React 19, TypeScript, Tailwind CSS, Vite.
- Database: PostgreSQL 17.

## Mandatory Coding Rules
1. Never use raw print statements; use `logging.getLogger(__name__)`.
2. All FastAPI routes must include response_model and strict type annotations.
3. Database changes MUST include an Alembic migration (`alembic revision --autogenerate`).
4. Never write inline CSS; use Tailwind utility classes.
5. All new endpoints must have a corresponding test in `tests/`.

## Verification Commands (Must pass before completing)
- Backend: `PYTHONPATH=. pytest tests/ && ruff check .`
- Frontend: `npm run lint && tsc -b`
```

### 2.2 Schema & AST Grounding
Instead of dumping full files into the LLM context, a pre-flight scanner generates an AST-derived symbol map of existing APIs and database models:
```text
[SCHEMA MAP]
- POST /api/projects  -> Input: schemas.ProjectCreate -> Output: schemas.ProjectRead
- GET  /api/projects  -> Output: list[schemas.ProjectRead]
- PATCH /api/projects/{id} -> Input: schemas.ProjectUpdate -> Output: schemas.ProjectRead
- Models: Project (id, name, status, created_at), Task (id, title, status, project_id)
```
* **Why it matters**: The agent knows the exact names, relationships, and types of existing entities before it writes a single line of code, eliminating hallucinated methods.

### 2.3 Toolchain & Dependency Pre-Baking
Enterprises and teams rely on specific build tools, package managers, and private registries (`uv`, `yarn`, `pnpm`, private npm feeds). 
* **The Pattern**: These tools are installed during **Golden Image creation** (`manifests/<app>.sh` and Dockerfile overrides).
* **Credential Forwarding**: `HOLODECK_FORWARD_ENV=NPM_TOKEN,REGISTRY_KEY` securely injects private registry tokens into the sandbox at strike time without persisting them in git.

---

## 3. In-Sandbox Coding Guardrails & AST Delta Tracking

### 3.1 Diff-Scoped Linting & Type Checking
Running a global linter across an entire repository often fails due to pre-existing technical debt. Guardrails must be **diff-scoped**:
* **Python**: `ruff check --diff` (verifies only lines touched by the agent).
* **TypeScript**: `tsc --noEmit` on modified files.
* **Formatting**: Auto-format modified files before running the test suite (`ruff format`, `prettier --write`).

### 3.2 What is AST Delta Tracking?
Standard `git diff` only checks text characters. **AST (Abstract Syntax Tree) Delta Tracking** parses code modifications into syntactic structures using Python's `ast` module and TypeScript's compiler API.

#### Three Critical AST Safety Guards:
1. **The "Sneaky Test Deletion" Guard**:
   * *The Problem*: When an agent struggles to fix a failing test, it may comment out or delete assertions (`assert response.status_code == 200`) to force the build to pass.
   * *The Guard*: AST compares the test tree before and after. If `Assert` nodes are removed from existing test functions, the Notary throws `AST_TEST_DELETION_DETECTED` and rejects the run.
2. **Security Decorator Guard**:
   * *The Problem*: An agent accidentally strips `@require_auth` or `Depends(get_current_user)` to fix a 401 error.
   * *The Guard*: AST verifies that all security decorators present on existing routes remain intact.
3. **Public API Compatibility Guard**:
   * *The Problem*: An agent renames or changes the required arguments of a shared utility function used across multiple microservices.
   * *The Guard*: AST checks that public function signatures remain backward-compatible.

---

## 4. The Decision Matrix: When to Halt vs. Execute

LLM prompt-based halting (*"Halt if the task is complex"*) is non-deterministic. Meeseek enforces a **Deterministic 3-Tier Halted State Machine**:

```
[ New Ticket Claimed ]
         │
         ▼
┌─────────────────────────────────┐
│ Tier 1: Explicit Human Mandate? │── YES ──► Post Plan & HALT (meeseek:halt)
└────────────────┬────────────────┘
                 │ NO
                 ▼
┌─────────────────────────────────┐
│ Tier 2: Complexity Thresholds?  │── YES ──► Post Plan & HALT (meeseek:halt)
│ - Files > 4                     │
│ - DB Drop / Alter               │
│ - LOC > 300                     │
│ - Security / Auth config        │
└────────────────┬────────────────┘
                 │ NO (Safe to execute)
                 ▼
┌─────────────────────────────────┐
│ Agent Begins Implementation     │
└────────────────┬────────────────┘
                 │
        Ambiguity discovered?
                 │
                 ├── YES ──► Invoke `request_human_input()` & HALT
                 │
                 └── NO  ──► Implement, verify, and finalize
```

### Tier 1: Explicit Human Mandate (100% Deterministic)
* If Jira ticket has label `meeseek:plan-only` or description contains `/plan`.
* Meeseek generates the detailed Implementation Plan, posts it as a Jira comment, and **hard-halts** (`meeseek:halt`).
* Execution resumes only when a human clicks **[Approve]** or comments `/approve`.

### Tier 2: Static Complexity Budget & Blast Radius (100% Deterministic)
Before touching source code, Meeseek analyzes the task scope:
* **File Blast Radius**: Does the task require modifying $> 4$ files? $\to$ **Halt**.
* **Destructive Schema Changes**: Does the plan include `DROP TABLE`, `DROP COLUMN`, or `ALTER TABLE` on large production tables? $\to$ **Mandatory Halt**.
* **Auth & Security Boundary**: Does the diff touch authentication middleware, CORS policies, or payment gateways? $\to$ **Mandatory Halt**.
* **LOC Budget**: Anticipated changes $> 300$ lines? $\to$ **Halt for review**.

### Tier 3: In-Flight Semantic Fork (Controlled Tool Invocation)
If an agent discovers an unexpected architectural trade-off mid-implementation:
* *Example*: *"The database has both a `members` table and a `users` table. Which table should the new task assignee relation point to?"*
* The agent invokes the tool: `request_human_input(question, options)`.
* Execution suspends, status becomes `WAITING_INPUT`, interactive buttons are posted to GitHub/Jira/Slack, and token burning halts until the engineer selects an option.

---

## 5. The Host Notary: Tamper-Proof Impartial Certification

### 5.1 Why Notary != LLM Self-Reporting
In competing tools, the agent self-reports: *"I ran pytest and all tests passed."* LLMs frequently hallucinate success to satisfy their prompt.

In Meeseek, the **Host Notary is an independent judge outside the container**:
1. **HTTP Readiness Probe**: Host queries the container's published port (`curl -I http://127.0.0.1:18001/`) to prove the webserver is bindable and healthy.
2. **Database Seed Integrity**: Host executes a direct SQL proof query (`HOLO_SEED_PROOF_SQL="SELECT count(*) FROM projects;"`) to verify data wasn't wiped out.
3. **OS Exit Code Certification (`HOLO_TEST_CMD`)**: The host executes the test command inside the container via `docker exec`, captures stdout/stderr, and inspects the raw integer exit code. If exit code $\neq 0$, the PR is blocked.

### 5.2 Two-Tier Test Validation
The Host Notary tests across two dimensions:
* **Tier A: Repo-Scoped Regression Suite (`HOLO_TEST_CMD`)**:
  Configured in the manifest (e.g. `pytest tests/`). Runs against the entire repository to prove existing features were not broken.
* **Tier B: Ticket-Specific Test (`ticket_test_cmd` / Agent Tests)**:
  For TASK-8 ("Add GET /ping route"), the agent authored `tests/test_ping.py`. The Notary independently executes `pytest tests/test_ping.py` and confirms it returns exit code 0.

---

## 6. CI Failure Remediation & The Self-Healing Loop

```
                       [ Host Notary Passes Exit 0 ]
                                     │
                                     ▼
                        [ Push Branch & Open PR ]
                                     │
                                     ▼
                       [ GitHub Actions CI Triggers ]
                                     │
                       ┌─────────────┴─────────────┐
                    PASSED                      FAILED
                       │                           │
                       ▼                           ▼
                 Ready to Merge          GitHub Webhook Received
                                         (workflow_run: failure)
                                                   │
                                                   ▼
                                         Circuit Breaker Checks:
                                         Attempts < 2?
                                         Deterministic error?
                                                   │
                                         ┌─────────┴─────────┐
                                        YES                  NO
                                         │                   │
                                         ▼                   ▼
                                 [ Self-Healing ]     [ meeseek:halt ]
                                 Checkout branch      Human Alert
                                 Feed logs to agent   Post to Issue
                                 Push fix commit
```

### 6.1 Ephemeral Workspaces & "Git as the Universal State Machine"
* **The Long CI Problem**: GitHub Actions integration suites can take 30–45 minutes. Ephemeral workspace leases have a 30-minute TTL to preserve host RAM.
* **The Solution**: Meeseek tears down the container immediately after opening the PR.
* **On Webhook Arrival**: When GitHub Actions reports a failure at $T = 40\text{min}$:
  1. Meeseek claims a fresh pre-warmed slot in **< 1 second**.
  2. Runs `git checkout agent/<ticket> && git pull`. The exact workspace state is restored instantly without keeping containers idling for 40 minutes.
  3. Feeds CI logs to the agent, applies the fix, and pushes to the existing PR.

### 6.2 Circuit Breakers: Preventing Infinite Remediation Loops
To prevent infinite thrashing when an external third-party service fails:
1. **Max Retry Budget**: `MAX_REMEDIATION_ATTEMPTS = 2`.
2. **Failure Classification**:
   * **Deterministic (Agent-fixable)**: Syntax error, lint failure, broken assertion, missing type. $\to$ Auto-remediate.
   * **Environmental (Infra failure)**: Timeout, 502/503 service outage, Docker daemon crash, rate limits. $\to$ Do NOT loop; alert human.
3. **Escalation**: If attempt count reaches 2, Meeseek immediately swaps the ticket label to `meeseek:halt` and posts the failing logs to GitHub/Jira/Slack.

---

## 7. Interactive Human Code Review on PRs

When a human engineer reviews Meeseek's PR on GitHub and leaves inline comments:
1. **Webhook Ingestion**: GitHub fires `pull_request_review_comment` to `POST /webhooks/github`.
2. **Security & Authorization Gate**:
   * Meeseek verifies the commenter is an authorized organization member (protects against malicious prompt injection on public repos).
3. **Workspace Claim**:
   * Meeseek checks out the PR branch `agent/<ticket>`.
   * Prompts the agent with the reviewer's exact comment and line reference.
4. **Iterative Verification & Push**:
   * The agent modifies the code, passes the Host Notary tests, and pushes a new commit to the open PR.
   * Meeseek replies directly on the GitHub PR thread:
     > *"Addressed review comment in commit `c7b2a1`. Tests verified passing."*

---

## 8. Strategic Debate: Capabilities, LOC Boundaries & Complexity

> **The Debate Question**: *"Meeseek is a sandbox that works best for single-prompt tasks and small bugs. It cannot solve problems with >1000 LOC or large effort. How accurate is this?"*

### Where the Statement is Accurate (The Raw LLM Limit)
* **Single-shot prompt limits**: No LLM in the world can reliably rewrite 2,000 LOC of interconnected enterprise business logic in a single turn without attention degradation or hallucinations.
* Tasks with extreme architectural ambiguity (e.g. *"Redesign our billing engine from scratch"*) require human strategic direction.

### Where the Statement is Inaccurate (Meeseek's Architectural Edge)
Meeseek is **not a single-shot prompt tool**. It is an **Agentic Operating System with State, Verification, and Multi-Turn Sandboxes**:
1. **Execution Feedback**: The agent runs partial code, checks compiler errors, runs DB migrations, and iterates dynamically in the container.
2. **Decomposition**: Complex tasks are split into stages: Planning $\to$ DB Migrations $\to$ Backend Endpoints $\to$ Frontend UI $\to$ Notary Verification.
3. **Host Notary Impartiality**: By holding the agent accountable to raw host exit codes, large multi-file diffs cannot be committed unless the entire test suite passes.

### The Enterprise Value Proposition
* **The Staff Engineer Multiplier**: Meeseek is not intended to replace the Principal Architect. It is designed to take the **60–70% of well-specified sprint tickets** (CRUD, API additions, schema migrations, bug fixes, UI components) and ship them certified with PRs and live previews in 3 minutes, liberating human engineers for high-level architecture.

---

## 9. Observability & Workflow State Machine: Console Dashboard Evolution 📊

### 9.1 The Current Baseline: Linear Stepper
Today's operator dashboard displays a linear 5-stage progress indicator:
`Provisioned` ➔ `Seeded` ➔ `Implementing` ➔ `Tests passed` ➔ `PR opened`.

**Limitations of the Current UI**:
* **Overly Linear**: Software delivery is rarely purely linear. When Host Notary tests fail, the workflow enters a cyclic **Self-Correction Loop** rather than advancing forward.
* **Lack of Granular Observability**: Intermediate agent turns, tool invocations, and live test stdout/stderr are hidden behind raw terminal dumps.
* **Static Demo Artifacts**: Earlier test suites contained hardcoded test case counts rather than dynamic runtime metrics parsed from the active lease.

### 9.2 The Next-Gen Workflow State Machine
In the upcoming phase, the progress section will be overhauled into an interactive, real-time **State Machine DAG (Directed Acyclic Graph with Remediation Cycles)**:

```
[ 1. Provision Slot ] ──► [ 2. Ingest Rules & Schema ] ──► [ 3. In-Sandbox Coding ]
                                                                     │
                                                             Agent Done / Idle
                                                                     ▼
                                                          [ 4. Host Notary Test ]
                                                                     │
                                             ┌───────────────────────┴───────────────────────┐
                                          PASS (Exit 0)                                FAIL (Exit != 0)
                                             │                                               │
                                             ▼                                               ▼
                                    [ 6. Certified PR ]                         [ 5. Self-Correction Loop ]
                                    - Git Push & PR Open                        - Capture stderr & traceback
                                    - Live Tunnel Preview                       - Retry 1/2 in sandbox
                                    - Jira Verification Card                                 │
                                                                                Attempts >= 2?
                                                                                ┌────┴────┐
                                                                               YES        NO ──► [ 3. Coding ]
                                                                                │
                                                                                ▼
                                                                        [ 7. Circuit Breaker ]
                                                                        - meeseek:halt
                                                                        - Human Alert in Jira
```

### 9.3 Architectural Decision: Deployment Strategy
The team will evaluate two UI hosting models:
1. **Model A: Embedded Single-Binary UI (FastAPI + Jinja + Tailwind)**
   * *Pros*: Zero CORS issues, runs on the same port as the control plane (`:18000`), zero node/npm build dependencies in production GCE/VM environments.
   * *Cons*: More complex custom state-rendering without modern reactive UI libraries.
2. **Model B: Standalone Modern Dashboard (React 19 / Vite / React Flow)**
   * *Pros*: First-class DAG visualization (using React Flow or VisX), rich component ecosystem, smooth animations for active agent loops.
   * *Cons*: Requires separate build artifact, dual-port proxying or Caddy routing.

---

## 10. Future Roadmap: Resilience & Deep Self-Correction 🛣️

### 10.1 Workspace Lifetime & Sliding-Window TTL Heartbeat
* **The Risk**: Leases have a default TTL (`HOLODECK_TTL_S = 3600`). If an agent engages in heavy multi-turn reasoning, or if a ticket pauses in `WAITING_INPUT` while an engineer attends a meeting, the background reaper (`reaper.py`) could prematurely tear down the container.
* **Proposed Mechanism**:
  1. **Sliding-Window Activity Heartbeat**: Automatically extend the lease (`client.extend(lease_id, ttl_s=3600)`) on every agent turn, Notary verification run, or human interaction.
  2. **Active-State Reaper Immunity**: The reaper must check `TaskRecord.workflow_state` before executing teardowns. Workspaces actively in `CODING`, `NOTARY_CORRECTING`, or `WAITING_INPUT` are immune from automatic reaping until genuinely abandoned.

### 10.2 In-Sandbox Self-Correction Loop & Circuit Breaker Reset (`manager.py`)
* **The Flow**:
  1. Automated capture of failing `test_cmd` stdout/stderr upon Host Notary execution.
  2. Re-injection of tracebacks into the active agent session (`driver.reiterate()`) up to `MAX_RETRIES = 2`.
  3. **Circuit Breaker Halt & Triaging**: When retries are exhausted, the ticket halts with `meeseek:halt`.
  4. **Human Reset Capabilities**:
     * **In-Place Assisted Retry**: Replying `#meeseek retry <guidance>` resets the retry counter to 0 and re-arms the agent with the engineer's architectural clue.
     * **Clean Restrike**: Replying `#meeseek reset` destroys the dirty workspace and strikes a clean snapshot from the Golden Image.



