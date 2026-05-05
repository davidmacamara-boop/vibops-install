# VibOps

**Intent-driven AI Infrastructure Operating System** — manage GPU clusters and LLM deployments in plain language.

```
"Deploy llama3 on the prod cluster with 3 GPU replicas"
"Scale the gpu-ng node group to 5 nodes"
"Increase open-webui CPU limits to 1 core in the infra repo"
```

---

## Architecture

```mermaid
flowchart LR
    Console["Console\n(Alpine.js)"] --> Agent
    Agent["VibOps Agent"] --> Core
    Core["Core\n(FastAPI + PolicyEngine)"] --> Gateway
    Gateway["Gateway\n(VibOps Connect)"] --> Abstraction
    Abstraction["Accelerator Abstraction\n(vendor-agnostic API)"] --> Connectors
    Connectors["23 Connectors\nkubectl · helm · EKS · NIM · AMD · Groq · ..."] --> Infra["GPU Infrastructure\n(NVIDIA · AMD · Intel · Groq · TPU)"]
```

| Component | Role |
|-----------|------|
| **Core** | Execution engine — jobs, pipelines, PolicyEngine, audit, secrets, multi-tenancy |
| **VibOps Agent** | LLM tool loop — intent parsing, guardrails, dry-run preview, confirmation flow |
| **Console** | Web UI — chat, monitoring, cluster view, org/team admin |
| **Gateway (Connect)** | On-prem edge agent — bridges Core to local infrastructure via Bearer token |
| **Accelerator Abstraction** | Vendor-agnostic layer over NVIDIA, AMD, Intel, Groq, Trainium, TPU — unified tools (`accelerator_diagnose`, `accelerator_detect_waste`, `accelerator_workload_match`, …) replace all vendor-specific APIs |
| **Connectors (23)** | kubectl, helm, argocd, git, EKS, GKE, AKS, NIM, NVIDIA, AMD, Intel, Groq, AWS Trainium, Google TPU, ollama, dgxcloud, datadog, GPU (unified), terraform, HPE, outscale, scaleway, benchmark |

See [docs/architecture/overview.md](docs/architecture/overview.md) for the full architecture with request flow and trust boundaries.

---

## Prerequisites

- Kubernetes cluster (kind, EKS, GKE, or AKS)
- Helm 3.x
- PostgreSQL 14+ (or let the Bitnami sub-chart provision it)
- LLM inference: **LLM API key** (`LLM_API_KEY`) — or an on-prem endpoint (Ollama,
  vLLM, Mistral, LLaMA) via `LLM_PROVIDER=openai` + `OPENAI_BASE_URL`. VibOps itself requires no
  GPU — see [docs/technical-architecture.md](docs/technical-architecture.md).

---

## Quick Start

**New to VibOps?** Follow the step-by-step guide: [QUICKSTART.md](QUICKSTART.md)
— no technical background required, covers Docker installation, API key setup, and first conversation.

```bash
make quickstart  # creates .env, generates secrets, starts stack
make check       # validates all components are healthy
make hash PASSWORD=yourpassword  # generates bcrypt hash for password setup
```

Console available at **http://localhost:8003** — API Swagger UI at **http://localhost:8000/docs**

Grafana available at **http://localhost:3000** (admin / vibops) — Prometheus at **http://localhost:9090**

### Onboarding a pilot client

```bash
# Creates org + admin user + budget in one command (idempotent)
make pilot-create-client ORG="Acme Corp" EMAIL=admin@acme.com PASSWORD=s3cr3t BUDGET=5000
```

See [`docs/runbooks/pilot-runbook.md`](docs/runbooks/pilot-runbook.md) for the full go-live checklist.

### Helm (production)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm dependency update ./helm/vibops

helm install vibops ./helm/vibops \
  -n vibops --create-namespace \
  -f my-values.yaml
```

See [docs/installation.md](docs/installation.md) for full setup instructions.

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture/overview.md) | Mermaid diagram, accelerator abstraction layer, destructive action flow, trust boundaries, full ADR index |
| [Security Architecture](docs/architecture/security.md) | 4 defense layers, threat model, SOC 2 posture |
| [Installation Guide](docs/installation.md) | Helm deployment, CSP delivery, bootstrap, configuration reference |
| [User Guide](docs/user-guide.md) | Console walkthrough, chat commands, team management |
| [API Reference](docs/api-reference.md) | 132 endpoints — curl examples grouped by domain |
| [Swagger UI](http://localhost:8000/docs) | Interactive API — auto-served by Core on every install |
| [OpenAPI Spec](docs/openapi.json) | Full REST API contract (machine-readable, auto-generated) |
| [Technical Architecture (DAT)](docs/technical-architecture.md) | Customer-facing architecture document |
| [Runbooks](docs/runbooks/) | Backup/restore, multi-region, pilot go-live checklist |
| [Status](STATUS.md) | Current version, what's shipped, known limitations, next sprint |

---

## Configuration

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | — | `postgresql+asyncpg://user:pass@host/db` |
| `JWT_SECRET_KEY` | `change-me` | Shared secret with Agent — **change in prod** |
| `JWT_EXPIRE_HOURS` | `24` | Token lifetime |
| `AUTH_PASSWORD_HASH` | `""` | Password hash — empty = auth disabled (dev) |
| `SECRET_KEY` | — | Encryption key for VibOps secrets vault |
| `DATASET_PSEUDONYMIZATION_SALT` | — | HMAC key for dataset pseudonymization — store in vault, never rotate without migration plan |

### Agent

| Variable | Default | Description |
|----------|---------|-------------|
| `LLM_API_KEY` | — | Required if `LLM_PROVIDER=cloud` (default). Leave empty for on-prem. |
| `CORE_API_URL` | `http://localhost:8000` | Core service URL |
| `JWT_SECRET_KEY` | `change-me` | Must match Core |
| `LLM_MODEL` | `<model-id>` | LLM model ID |

### Connectors (optional)

| Variable | Description |
|----------|-------------|
| `ARGOCD_SERVER` | ArgoCD URL (e.g. `https://argocd.mycompany.com`) |
| `ARGOCD_TOKEN` | ArgoCD API token |
| `AWS_REGION` | AWS region for EKS |
| `GIT_TOKEN` | GitHub/GitLab Personal Access Token |
| `GIT_PROVIDER` | `github` or `gitlab` |
| `DATADOG_API_KEY` | Datadog API key |
| `NGC_API_KEY` | NVIDIA NGC key for NIM |

---

## Multi-tenancy & Reselling

```
Organisation [reseller]              — CSP that resells VibOps
  └── Organisation [reseller_customer]  — CSP's end customer
        └── Team (scope: namespaces, clusters, envs, allowed actions)
              └── Member (role: admin | developer | readonly)

Organisation [direct]               — Direct VibOps customer
  └── Team → Member
```

Each CSP client gets a dedicated VibOps instance — isolation is at the deployment level, not RBAC intra-instance.

**Tier 3 reselling** — CSPs can onboard their own customers with white-label branding (`white_label_name`, `white_label_slug`), custom pricing markup, and per-customer chargeback reports. See [ADR 0013](core/docs/adr/0013-tier3-reselling.md).

### Reselling integration patterns

CSPs can adopt VibOps at three integration depths, depending on how much of their existing client experience they want to preserve:

**1. Console-replacement** — VibOps console is white-labeled with the CSP's branding (`white_label_name`, `white_label_slug`). End customers log into the VibOps console and see only the CSP's brand. Fastest path to market for CSPs without an existing client portal.

**2. Hybrid embedding** — CSP keeps its existing client console and embeds VibOps capabilities through iframes or component-level integration. The end customer sees one unified UI with VibOps features appearing native. Suitable for CSPs with mature portals who want to add GPU operations capabilities without console fragmentation.

**3. API-only / headless** — CSP consumes VibOps purely through REST endpoints and integrates the data into its own UI. End customers never see VibOps in any form. Most flexibility, suitable for CSPs with heavy investment in their existing portal who want VibOps as a backend service.

Each pattern is supported by the same API surface — CSPs can move between patterns or mix them across customer segments.

---

## FinOps Engine

GPU cost visibility and chargeback — built in, no external tooling required.

| Feature | Description |
|---------|-------------|
| **Budget management** | Per-org budget with configurable alert thresholds |
| **Chargeback reports** | Monthly cost breakdown per org/team, exportable |
| **Waste detection** | Idle GPU detection with scale-down recommendations |
| **Pricing rules** | Per-cluster GPU rates (cloud formula or on-prem TCO) |
| **Customer markup** | CSP resellers apply markup per customer or per workload type |
| **Cost prediction** | At-submission cost estimate frozen with each job |

See [ADR 0014](core/docs/adr/0014-sprint8-finops.md), [ADR 0015](core/docs/adr/0015-sprint9-pricing.md).

---

## Pipelines & Automation

Multi-step deployment pipelines with rollback guards — triggered by the agent or on schedule.

```
Pipeline: deploy-llama3-prod
  Step 1 — deploy staging      (on_failure: stop)
  Step 2 — health check        (on_failure: rollback step 1)
  Step 3 — promote to prod     (on_failure: rollback)
  Step 4 — health check prod   (on_failure: rollback step 3)
```

**Trigger rules** — auto-trigger pipelines or jobs based on conditions (GPU utilization threshold, alert state, schedule). Supports `AND`/`OR` multi-condition logic.

---

## Alerts & SLOs

- **Alert rules** — threshold-based (GPU utilization, cost, job failure rate) with configurable channels (Slack, webhook, email)
- **SLO monitoring** — define uptime/latency targets per deployment; breach triggers automatic remediation or escalation
- **Notification channels** — Slack, PagerDuty, webhook, email — configurable per org

---

## Secrets Vault

Encrypted key-value store for connector credentials — scoped per org, never logged.

```bash
# Store a secret
POST /api/v1/secrets   {"name": "git_token", "value": "ghp_..."}

# Reference in agent commands
"Clone the infra repo"  →  git_clone uses token=@secret:git_token automatically
```

Secrets are AES-encrypted at rest using `SECRET_KEY`. The agent injects them at execution time — credentials never appear in job payloads or audit logs.

---

## Operational Dataset

VibOps builds a proprietary GPU workload dataset from production operations — the foundation for domain-specialized model fine-tuning.

| Signal | Source | Sprint |
|--------|--------|--------|
| WorkloadSignature (vendor, accelerator, framework) | Job submission | 10 |
| Job outcomes (success/oom/timeout/failure category) | Worker completion | 10 |
| Recommendation events (followed/ignored/overridden) | Operator response | 10 |
| Framework auto-detection from container image | WorkloadDetector | 11 |
| Agent feedback (thumbs up/down per response) | Chat UI | 13 |

**Consent model** (per org, set by org admin):
- `pseudonymized` — stable hashed identifiers, cross-job correlation preserved
- `anonymized` — identifiers stripped, full anonymity
- `opted_out` — excluded from all exports

Export: `GET /api/v1/dataset/export` (jobs, JSONL) · `GET /api/v1/training/export` (exchanges + feedback, alpaca/sharegpt/chatml)

Requires `DATASET_PSEUDONYMIZATION_SALT` env var for pseudonymization. See [ADR 0016](core/docs/adr/0016-sprint10-dataset-foundation.md), [ADR 0017](core/docs/adr/0017-sprint11-dataset-enrichissement.md), [ADR 0018](core/docs/adr/0018-sprint12-dataset-consent.md).

---

## Security

Six independent defense layers — each sufficient to block most attacks:

0. **Agent behavioral correctness** — 3-layer testing stack (L1 form, L2 coherence, L3 behavior) prevents LLM misrouting before any action reaches infrastructure ([ADR 0009](agent/docs/adr/0009-agent-testing-layers.md))
1. **Endpoint auth invariant** — router-level JWT enforcement + CI test that fails if any endpoint is added without auth ([ADR 0005](core/docs/adr/0005-endpoint-auth-invariant.md))
2. **PolicyEngine default-deny** — every action must be in `TOOL_CATALOG`; unknown actions → 403 ([ADR 0001](core/docs/adr/0001-policy-engine-default-deny.md))
3. **Two-tier authorization** — `evaluate()` for API callers (role + destructive confirmation), `evaluate_system()` for Celery workers ([ADR 0002](core/docs/adr/0002-two-tier-authorization.md))
4. **Multi-tenant isolation** — `org_id` from JWT on every row, never from request body ([ADR 0003](core/docs/adr/0003-multi-tenant-isolation.md))
5. **Tier 3 secret isolation** — cross-org fallback restricted to `is_system=True` secrets; org-scoped credentials never accessible from another org (Sprint 15)

Destructive actions return a **dry-run preview** (`reversibility`, `resolved_params`) and require explicit user confirmation before executing. The confirmation token can only be injected by `confirm_action` — `create_job` strips it structurally.

See [docs/architecture/security.md](docs/architecture/security.md) for the full threat model.

---

## Multi-Accelerator Support

VibOps abstracts GPU vendor differences behind a unified interface — the same commands work regardless of the underlying hardware.

| Vendor | Supported operations |
|--------|---------------------|
| **NVIDIA** | GPU Operator install, MIG partitioning, DCGM metrics, NIM deployment |
| **AMD** | ROCm device plugin, SMI metrics |
| **Intel** | OpenVINO device plugin |
| **Groq** | Device listing and workload matching |

**Unified tools** — vendor-agnostic:

| Tool | Description |
|------|-------------|
| `accelerator_list_devices` | List all accelerators across vendors (memory, precision, partitioning capability) |
| `accelerator_get_metrics` | Unified utilization, temperature, power draw |
| `accelerator_diagnose` | Structured diagnostic — checks operator, driver, device plugin in one call |
| `accelerator_workload_match` | Match workload requirements (precision, memory, FLOPS) against available hardware |
| `accelerator_cost_estimate` | Hourly/daily/monthly cost projection |
| `accelerator_portability_check` | Migration cost assessment (CUDA → ROCm, multi-cloud scenarios) |
| `accelerator_deploy_workload` | Deploy a GPU workload with correct vendor resource requests |
| `accelerator_partition_device` | Enable/disable MIG or equivalent partitioning |
| `accelerator_install_operator` | Install vendor device operator via Helm |

Placement is automatic: `accelerator_workload_match` ranks available hardware against workload requirements (workload type, bottleneck profile, dominant precision, memory breakdown) before deployment.

---

## API-First / Headless

VibOps is fully operable without any UI. Every capability is exposed via REST — the console is just one client among many.

**Authentication:**
```bash
# Long-lived API token (for CI/CD, scripts, integrations)
POST /api/v1/auth/tokens
→ {"token": "vbops_xxxxxxxx", "name": "ci-pipeline"}

# Use it
curl -H "Authorization: Bearer vbops_xxxxxxxx" https://vibops.mycompany.com/api/v1/jobs
```

**Typical headless workflows:**

```bash
# Trigger a deployment
curl -X POST /api/v1/jobs \
  -d '{"action": "deploy_model", "payload": {"image": "vllm/vllm-openai:v0.6.2", "replicas": 3}}'

# Run a pipeline
curl -X POST /api/v1/pipelines/{id}/trigger

# Get GPU cost report
curl /api/v1/finops/waste

# Export dataset (RLHF-ready JSONL)
curl /api/v1/training/export?format=alpaca > dataset.jsonl
```

**Integration surfaces:**
| Surface | Use case |
|---------|----------|
| REST API + JWT/API tokens | CI/CD pipelines, internal tooling, operator scripts |
| Webhooks | Push job/alert events to external systems (Slack, PagerDuty, custom) |
| OpenAPI spec (`/docs`, `docs/openapi.json`) | SDK generation, Postman collections, contract testing |
| Trigger rules | Event-driven automation without human intervention |

The console (`localhost:8003`) and the agent chat (`POST /api/v1/chat`) are optional layers — teams that prefer direct API control or want to embed VibOps into their own UI can do so without compromise.

---

## GitOps Workflow

```
Agent receives the request
  └── git_clone → git_read_file → git_patch_yaml → git_diff
        └── git_commit_push (branch vibops/xxx)
              └── git_create_pr (optional — human review)
                    └── ArgoCD sync → cluster
```

Requires: `GIT_TOKEN` + `GIT_PROVIDER` in env, ArgoCD pointing at the client infra repo.

---

## Project Structure

```
vibops/
├── core/           # REST API + DB models + Alembic migrations
│   ├── app/
│   │   ├── api/v1/ # Endpoints (jobs, auth, audit, tenants…)
│   │   ├── models/ # SQLAlchemy (Job, User, Team, AuditLog…)
│   │   └── services/
│   ├── alembic/    # Migrations
│   └── scripts/    # bootstrap.py
├── agent/          # LLM agent (tool use, guardrails, user scope)
│   └── prompts/    # system_prompt.md
├── connectors/     # Kubectl, Helm, ArgoCD, NIM, EKS, GKE, AKS, Git, Ollama, NVIDIA, AMD, Intel, Groq, Datadog, Terraform…
├── console/        # Web interface (Alpine.js + FastAPI proxy)
├── helm/vibops/    # Helm chart — K8s deployment
├── docs/           # Installation guide, user guide, runbooks, OpenAPI spec
├── infra/dev/      # K8s manifests for dev environment
├── Makefile        # pilot-create-client, backup-now, backup-list, up/down/logs
└── docker-compose.yml
```

---

## Agent Behavioral Model

The agent (`agent/prompts/system_prompt.md`) is a rule-driven LLM tool loop — not a vanilla chatbot. Its behavior is codified in 13 mandatory rules:

| Rule | Behavior |
|------|----------|
| **ACT DIRECTLY** | No permission-seeking. Execute requests immediately. |
| **VERIFY VIA TOOLS** | Never answer from memory. Always call the tool. MANDATORY: `list_clusters` if cluster unspecified. |
| **PARALLEL EXECUTION** | All independent actions launched in the same turn. |
| **GPU AUDIT** | `accelerator_diagnose` + `get_cluster_resources` mandatory before `nim_deploy` / `deploy_model`. |
| **ANTI-LOOP** | `setup_kubeconfig` called once per cluster per turn. No re-verification after success. |
| **FAILURE CLASSIFICATION** | 400/404 → permanent stop. 502/503 → retry once. Config error → stop and report missing prerequisite. |
| **NAMESPACE** | Always explicit. If not found, auto-discover via `get_cluster_resources`. |
| **SCALE/PATCH** | `patch_deployment` acts directly — no pre/post `get_deployment_status`. |
| **JOB POLLING** | Poll `get_job` max twice after pending. Never relaunch a running job. |
| **CONFIRMATION** | Destructive actions (scale_cluster, rollback, deploy_model, nim_deploy) require explicit confirmation on staging/prod. |
| **ENVIRONMENT** | `env` (dev/staging/prod) always included. Ask if unspecified. |
| **LANGUAGE** | Respond in the user's language. |
| **STYLE** | Concise. Job ID in confirmation. No emojis. No filler. |

**Incident response** — mandatory 4-step workflow triggered by any mention of service degradation: `correlate_incident` → `analyze_pod_failure` (if crash) → diagnosis → `remediate_incident` (with confirmation).

**Epistemic honesty** — `prometheus.observation` transcribed verbatim. "Not observed" ≠ "not present". Confidence levels (`confirmed` / `unknown`) govern next action.

**Multi-accelerator routing** — vendor-agnostic tools (`accelerator_detect_waste`, `accelerator_diagnose`, `accelerator_get_metrics`) are the default. NVIDIA-specific tools (`get_idle_gpus`, `get_gpu_operator_status`) are explicitly FORBIDDEN for general use.

See [`agent/prompts/system_prompt.md`](agent/prompts/system_prompt.md) for the full rule set.

---

## Tests

```bash
cd core       && pytest   # 580 tests
cd agent      && pytest   # 50 tests
cd connectors && pytest   # ~67 tests
```

~697 tests, 0 failures.

### Testing Architecture — Three Layers

The agent has a dedicated 3-layer behavioral testing stack ([ADR 0009](agent/docs/adr/0009-agent-testing-layers.md)) that validates correctness at each level of abstraction:

| Layer | What it tests | How |
|-------|--------------|-----|
| **L1 — Form** | Tool call structure, required params, forbidden params | Assert tool name + param schema, no LLM needed |
| **L2 — Coherence** | Routing logic, tool sequencing, anti-patterns | Mock LLM responses, verify the right tool is called in the right order |
| **L3 — Behavioral** | End-to-end intent → action correctness | Real `LLM_API_KEY` required; skipped in CI without key |

**L3 reference scenarios** include: deploy llama3 with GPU audit, scale node group vs. pod replicas (must not confuse them), incident triage (`correlate_incident` called first), NIM workflow (`nim_list_catalog` → `nim_profiles` → audit → deploy), GitOps workflow (full `git_clone` → `git_patch_yaml` → `git_commit_push` chain), and accelerator placement (`accelerator_workload_match` before deployment when cluster unspecified).

**Cost model** — L3 tests are gated behind `LLM_API_KEY` to avoid per-run LLM costs in standard CI. Run them explicitly before releases.

**Core coverage** includes PolicyEngine (default-deny, role matrix, destructive confirmation), dry-run preview (reversibility + resolved_params), pipeline/trigger workers, rate limiter, gateway E2E, CI auth enforcement invariant (fails if any endpoint is added without JWT auth), and the full dataset pipeline (WorkloadDetector, consent, anonymization, export formats, RLHF feedback loop).

---

## License

Proprietary — © VibOps 2026
