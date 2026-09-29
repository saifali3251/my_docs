# Deployment & Scaling Guide 🚀

This document covers running Meeseek and Omnigent across three deployment tiers:
1. **Local Development Setup** (Docker Desktop / macOS)
2. **Single-VM Production / Staging Deployment** (AWS EC2 / GCP Compute Engine with Caddy TLS)
3. **Production Scaling Roadmap** (Moving Omnigent to Kubernetes and Autoscaling the Substrate Fleet)

---

## 1. System Deployment Model: Two Decoupled Services

Meeseek and Omnigent run as two decoupled, independent services communicating over loopback HTTP:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COLOCATED CLOUD VM (EC2 / GCE)                  │
│                                                                        │
│   Public Ports :80 / :443                                              │
│            │                                                           │
│            ▼                                                           │
│   [ Caddy Reverse Proxy (Auto Let's Encrypt TLS) ]                     │
│            │                                                           │
│            ├──► :8000 ──► [ Omnigent Server Container (Docker) ]       │
│            │                      │                                    │
│            │              HTTP :8099 (HOLODECK_URL)                    │
│            │                      ▼                                    │
│            └──► :8099 ──► [ Meeseek Control Plane (Host Python Process)]│
│                                   │                                    │
│                           Docker CLI / CoW Reflinks                    │
│                                   ▼                                    │
│                           [ Ephemeral Workspaces ws-* ]                │
└────────────────────────────────────────────────────────────────────────┘
```

- **Meeseek (`meeseek/`)**: Python FastAPI control-plane process running natively on the host OS to manipulate Docker Compose and XFS/APFS filesystem reflink clones directly.
- **Omnigent (`omnigent-deploy/`)**: Docker Compose stack running the agent orchestrator, PostgreSQL session database, and Caddy reverse proxy.
- **The Bridge (`scripts/sync-wheel.sh`)**: The Omnigent container includes Meeseek's `omnigent-community-sandbox-meeseek` Python wheel, which implements Omnigent's `SandboxLauncher` interface to call Meeseek's `/leases` API.

---

## 2. Local Development Setup (macOS / Docker Desktop)

Local bring-up requires no TLS certificates and runs entirely on `localhost`.

### Prerequisites
- Docker Desktop running (Docker Compose $\ge 2.24$).
- Python $\ge 3.11$.
- macOS APFS filesystem (default).

### Step 1: Start Meeseek Control Plane
```bash
cd meeseek/control-plane
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt   # or poetry install

# Run control plane in fake mode (for API testing) or compose mode (with Docker)
HOLODECK_PROVIDER=fake uvicorn holodeck.main:app --host 127.0.0.1 --port 8099
```
* The control plane binds to `127.0.0.1:8099`.
* Operator console is available at `http://127.0.0.1:8099/ops`.

### Step 2: Build & Start Omnigent
In a separate terminal:
```bash
cd omnigent-deploy

# 1. Sync the Meeseek provider wheel from the control plane
./scripts/sync-wheel.sh

# 2. Generate local environment file
CLOUD=local ./scripts/pull-env.sh

# 3. Boot Omnigent and local PostgreSQL
cd omnigent
docker compose up -d postgres omnigent
docker compose logs -f omnigent
```
* Visit `http://localhost:8000` to access the Omnigent agent dashboard and create an initial admin account.

---

## 3. Single-VM Cloud Deployment (AWS EC2 / GCP GCE)

For staging, demo, and hackathon deployments, colocating both services on a single Linux virtual machine provides the best performance with zero Kubernetes cluster management overhead.

### VM Sizing & Filesystem Requirements
- **Instance Type:** `m5.4xlarge` (AWS) or `n2-standard-16` (GCP) (16 vCPU, 64 GB RAM recommended to support multiple concurrent full-stack workspaces).
- **Disk:** 200 GB+ SSD. **Must be formatted as XFS with reflink support:**
  ```bash
  sudo mkfs.xfs -m reflink=1 -f /dev/nvme1n1
  sudo mkdir -p /opt/holo/holodeck-data
  sudo mount -o noatime /dev/nvme1n1 /opt/holo/holodeck-data
  ```

### Automated TLS via Caddy
The included `omnigent-deploy/omnigent/Caddyfile` issues automated Let's Encrypt certificates using dynamic `*.sslip.io` hostnames or a custom wildcard domain:
- Inbound ports `80` and `443` must be open in your cloud Security Group / Firewall.
- Caddy automatically binds to `https://<public-ip>.sslip.io` and routes:
  - `/` $\to$ Omnigent Web UI (:8000)
  - `p<port>.*` $\to$ Struck Workspace live preview ports (:18001–18020)

### Bring-up Script
```bash
cd omnigent-deploy
./scripts/sync-wheel.sh

# Export required secrets
export HOLODECK_TOKEN=$(openssl rand -hex 16)
export ANTHROPIC_API_KEY="sk-ant-..."
export OMNIGENT_ACCOUNTS_COOKIE_SECRET=$(openssl rand -hex 32)
export POSTGRES_PASSWORD=$(openssl rand -hex 16)

# Launch automated provisioning
./scripts/provision-omnigent-aws.sh   # or GCE equivalent
```

---

## 4. Production Scaling Roadmap (Enterprise Blueprint)

When scaling beyond a single VM to support hundreds of concurrent software engineering tasks across an enterprise, Meeseek evolves into a **distributed multi-tier fleet**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        KUBERNETES LAYER (EKS / GKE)                    │
│                                                                        │
│   Ingress (Traefik / Cloud ALB) with Wildcard TLS (*.preview.corp.com) │
│            │                                                           │
│            ▼                                                           │
│   Omnigent Server Pods (Stateless HPA Autoscaling)                     │
│   • Redis Cluster (Distributed session locking & registry)             │
│   • Managed PostgreSQL (Cloud SQL / Amazon RDS)                        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                            Internal gRPC / HTTP
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     DISTRIBUTED MEESEEK CONTROL PLANE                  │
│                                                                        │
│   Central Lease Coordinator API                                        │
│   • Distributed PostgreSQL Store (Replaces SQLite WAL)                 │
│   • Redis Distributed Lease Locks & Capacity Allocator                 │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                            Node Agent Dispatch
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   AUTOSCALING WORKER SUBSTRATE FLEET                   │
│                                                                        │
│   Auto Scaling Group (ASG / MIG) of Reflink Worker Nodes               │
│   ┌──────────────────────────┐      ┌──────────────────────────┐       │
│   │ Node 1 (XFS Reflink SSD) │      │ Node 2 (XFS Reflink SSD) │       │
│   │ • Pre-pulled Golden EBS  │      │ • Pre-pulled Golden EBS  │       │
│   │ • Workspaces ws-1..ws-10 │      │ • Workspaces ws-11..ws-20│       │
│   └──────────────────────────┘      └──────────────────────────┘       │
└────────────────────────────────────────────────────────────────────────┘
```

### Pillar A: Moving Omnigent to Kubernetes (EKS / GKE)
1. **Stateless Web Tier:** Move the Omnigent container into a Kubernetes `Deployment`.
2. **External Session & State Broker:** Replace the in-memory host registry with a shared Redis instance and an external PostgreSQL database (Amazon RDS or Cloud SQL). This enables scaling Omnigent horizontally to $N$ pod replicas behind a Horizontal Pod Autoscaler (HPA).
3. **Init-Container Wheel Injection:** An init container dynamically bakes the latest `omnigent-community-sandbox-meeseek` provider wheel into an `emptyDir` volume shared with the Omnigent pod.

### Pillar B: Autoscaling Substrate Fleet & Bin-Packing
The sandboxes running Docker Compose and holding database data require high I/O throughput:
1. **Separation of Control & Compute:** The Meeseek control plane runs as a stateless API service, while actual workspace strikes are dispatched to a fleet of **Reflink Worker Nodes**.
2. **Golden Snapshot Distribution:** Nightly golden builds bake an EBS / Persistent Disk snapshot. New worker nodes joining the Auto Scaling Group (ASG) mount the pre-seeded golden snapshot in under 30 seconds.
3. **Bin-Packing Scheduler:** The Lease Coordinator schedules new tasks to worker nodes based on available RAM, CPU, and free port slots (e.g. up to 10 workspaces per 64 GB worker node).
4. **PostgreSQL Lease Store:** Replace the local SQLite store with a managed PostgreSQL database, using row-level locking for atomic lease claims.

### Pillar C: Global Wildcard Live Preview Routing
In enterprise production, developers should access previews via clean URLs:
- **Wildcard DNS:** `*.preview.yourdomain.com` points to an enterprise Cloud Load Balancer.
- **Dynamic Ingress:** Traefik or an edge API Gateway inspects the subdomain prefix (e.g. `p18001`), queries the Meeseek Lease Coordinator to locate the worker node IP, and transparently proxies traffic to that container's hot-reloaded dev server with zero manual DNS updates.

