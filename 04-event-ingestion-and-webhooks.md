# Event Ingestion & Webhooks Architecture ⚡

Meeseek acts as an autonomous engineer embedded directly in the engineering team's existing workflows. This document details the inbound event ingestion engine, webhook security contracts, event routing logic, and the self-healing feedback mechanisms implemented in the Meeseek Control Plane.

---

## 1. Event Ingestion Overview

The Control Plane exposes dedicated HTTP webhook endpoints to handle two distinct categories of events:
1. **Repository Lifecycle Events (GitHub `push`)**: Triggers automated, debounced golden image rebuilding when code merges into the default branch.
2. **Task & Review Lifecycle Events (GitHub Issues / PR Comments / Jira)**: Triggers workspace strikes, agent investigation, interactive elicitation replies, and automated self-healing loops based on human review comments.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        INBOUND EVENT INGESTION                         │
│                                                                        │
│   GitHub Push Event               GitHub Issues / Comments / PR Reviews│
│   [X-GitHub-Event: push]          [X-GitHub-Event: issues / comment]   │
│            │                                       │                   │
│            ▼                                       ▼                   │
│   POST /webhooks/github                   POST /webhooks/github        │
│   (HMAC SHA-256 Validated)                (HMAC SHA-256 Validated)     │
│            │                                       │                   │
│            ▼                                       ▼                   │
│   GoldenSyncService                       ConsoleManager / LeaseAPI    │
│   (Debounce & Coalesce)                   (Correlate & Strike)         │
│            │                                       │                   │
│            ▼                                       ▼                   │
│   Atomic Golden Rebuild                   Ephemeral Agent Workspace    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Webhook Security & Verification

All inbound webhook requests are authenticated before processing:

### GitHub Webhook Security (`POST /webhooks/github`)
- **HMAC Signature Validation:** Every incoming request must provide an `X-Hub-Signature-256` header.
- **Timing-Safe Comparison:** The control plane hashes the raw request body with the configured `GITHUB_WEBHOOK_SECRET` using HMAC SHA-256 and compares signatures using `hmac.compare_digest` to prevent timing attacks.
```python
computed = "sha256=" + hmac.new(
    cfg.github_webhook_secret.encode("utf-8"),
    body,
    hashlib.sha256,
).hexdigest()
if not hmac.compare_digest(computed, sig):
    raise HTTPException(status_code=401, detail="Invalid webhook signature")
```
- **Replay Protection:** Webhook timestamps and GitHub Delivery UUIDs (`X-GitHub-Delivery`) are recorded to prevent replay duplication.

### Jira Webhook Security (`POST /jira/webhook`)
- Authenticated via shared secret token query parameter or header against `HOLODECK_JIRA_WEBHOOK_SECRET`.

---

## 3. GitHub Event Handlers Specification

### Event 1: `push` (Automated Golden Sync — Implemented)
When a pull request merges into `main`, the golden image must be refreshed so future tasks run on current dependencies and schema.

- **Header:** `X-GitHub-Event: push`
- **Payload Inspection:**
  - `ref`: Evaluated against `refs/heads/main`.
  - `repository.name`: Matched against registered application manifests or composite component repos.
  - `after`: Git commit SHA of the merge.
- **Debouncing & Coalescing Engine (`holodeck/golden_sync.py`):**
  - Merge storms (e.g. 5 PRs merged within 2 minutes) do not trigger 5 concurrent expensive golden builds.
  - Pushes enter a **30-second debounce window**. Intermediate pushes coalesce into a single rebuild targeting the latest commit SHA.
  - Rebuild executes in a clean staging directory and atomically replaces `$HOLO_GOLDEN` via an OS rename.

---

### Event 2: `issues` (Task Trigger — In-Flight)
Allows developers or product managers to assign a bug or feature to Meeseek directly from GitHub.

- **Header:** `X-GitHub-Event: issues`
- **Trigger Conditions:**
  - `action == "labeled"` and `label.name == "meeseek"`, OR
  - `action == "opened"` with label `meeseek` pre-applied.
- **Payload Extraction:**
  - `issue.number`: Becomes the ticket identifier (e.g. `ISSUE-42`).
  - `issue.title` + `issue.body`: Formats the prompt and acceptance criteria for the agent.
  - `repository.name`: Identifies the target application.
- **Execution Flow:**
  1. Calls `ConsoleManager.trigger(ticket="ISSUE-42", prompt=...)`.
  2. Claims a warm pool workspace or executes a CoW strike.
  3. Deploys the agent session and posts an introductory acknowledgment comment to the issue thread with the active session status.

---

### Event 3: `issue_comment` (Human Clarification Reply — In-Flight)
Enables conversational collaboration when an agent pauses with an elicitation question.

- **Header:** `X-GitHub-Event: issue_comment`
- **Trigger Conditions:**
  - `action == "created"`
  - `comment.user.type != "Bot"` (ignores automated bot echoes)
  - Issue state in Meeseek is currently `waiting-input`.
- **Parsing & Resolution (`parse_verdict`):**
  - If the human replies with an affirmation (`"yes"`, `"approve"`, `"proceed"`, `"LGTM"`, `"👍"`), the approval elicitation resolves `True`.
  - If the human replies with a rejection (`"no"`, `"reject"`, `"stop"`), it resolves `False`.
  - If the human provides open-ended instruction (`"Use SQLAlchemy models instead of raw SQL"`), it routes via `reiterate()` as a guidance message.
  - The agent resumes execution inside the sandbox immediately.

---

### Event 4: `pull_request_review_comment` (Self-Healing Review Loop — In-Flight)
Senior engineers review code on GitHub by leaving inline diff comments. Meeseek listens for review comments on its opened PRs and repairs the code automatically.

- **Header:** `X-GitHub-Event: pull_request_review_comment`
- **Trigger Conditions:**
  - `action == "created"`
  - PR was opened by Meeseek's service account (e.g. `agent/*` branch).
- **Feedback Injection Flow:**
  1. Extracts the comment diff hunk, file path, line number, and reviewer critique.
  2. Synthesizes a targeted remediation prompt:
     ```
     Code Review Feedback on file '{path}' at line {line}:
     "{comment_body}"
     Please update your implementation inside the workspace to address this review comment.
     ```
  3. Sends prompt into the existing workspace via `reiterate()`.
  4. Agent edits the file and re-runs in-sandbox tests.
  5. The host-side Notary re-validates the workspace via `finalize()`.
  6. Upon passing, a new commit is pushed to the existing PR branch, and an automated response is posted confirming resolution.

---

## 4. Webhook Routing Table

| Route | Provider | Supported Events | Target Internal Handler |
|---|---|---|---|
| `POST /webhooks/github` | GitHub | `ping` | Health pong check |
| `POST /webhooks/github` | GitHub | `push` | `GoldenSyncService.notify_push()` |
| `POST /webhooks/github` | GitHub | `issues` (`labeled`, `opened`) | `ConsoleManager.trigger()` |
| `POST /webhooks/github` | GitHub | `issue_comment` (`created`) | `ConsoleManager.answer()` / `reiterate()` |
| `POST /webhooks/github` | GitHub | `pull_request_review_comment` | Self-healing PR remediation loop |
| `POST /jira/webhook` | Jira Cloud | `jira:issue_created`, `jira:issue_updated` | `JiraBridge.handle()` |
| `POST /jira/webhook` | Jira Cloud | `comment_created` | `ConsoleManager.answer()` |

