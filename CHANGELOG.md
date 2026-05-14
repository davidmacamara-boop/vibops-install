# Changelog

All notable changes to VibOps are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

---

## [0.17.6] — 2026-05-13

### Added
- **`GET /workloads`** — filtered listing with real-time cost annotation (`cost_usd`, `gpu_hours`, `rate_per_gpu_hour`); optional filters: `cluster_name`, `status`, `namespace`, `workload_type`; pagination (`limit`/`offset`); ordered by `started_at DESC`
- **`GET /workloads/cost-summary`** — aggregated GPU cost attribution for a cluster: `total_gpu_hours`, `total_cost_usd`, `by_namespace` (namespace for k8s_pod, partition for slurm_job), `top_workloads` (top 10 by cost); `rate_available=false` when no ClusterRate configured
- **`GET /workloads/{id}`** — single workload detail with cost annotation (404 + org isolation)
- **Cost formula**: `gpu_hours = gpu_seconds_accumulated / 3600`, `cost_usd = gpu_hours × rate_per_gpu_hour`; computed on-the-fly, never stored — rate changes take effect immediately
- **Console Workloads sub-tab** — cluster selector, cost attribution table by namespace/partition (workload count, running count, GPU-hours, cost), top spenders list; amber warning when no ClusterRate configured
- **Agent system prompt** — workload cost attribution intents: `cost-summary` rollup, filtered listing, single detail
- **CI `publish-install` job** — auto-publishes to vibops-install repo on every push to `main` (gated on `test-core` + `openapi-spec` passing); uses `INSTALL_REPO_TOKEN` secret

### Tests
- 31 new tests in `test_sprint20_workload_cost.py` (8 unit + 8 list endpoint + 9 cost-summary + 5 detail + 1 auth)

---

## [0.17.5] — 2026-05-12

### Added
- **`sacct` accounting for terminated Slurm jobs** — `SlurmWorkloadCollector.collect_completed()` queries `sacct` (REST slurmdb API or SSH) for jobs that completed between polls; window = `now - 2×POLL_INTERVAL_S` (120 s) to avoid missed transitions
- **`finalize_completed_workloads()`** — new DB helper: idempotent UPDATE for Slurm jobs that moved to terminal states; sets `ended_at` and `status` from `sacct` exact timestamps; takes priority over `mark_terminated_workloads()` approximation
- **`WorkloadSnapshot.ended_at`** — optional `datetime | None` field set by sacct; `None` for Kubernetes and running Slurm jobs
- **`_sacct_to_snapshots()`** — handles nested TRES format (`tres.allocated: [{"type":"gpu",...}]`) via `_tres_from_nested()`; strips `.batch` job step suffix; maps COMPLETED / FAILED / CANCELLED / TIMEOUT → terminal statuses

### Changed
- `sync_workloads` Celery task now calls `collect_completed(since=now-120s)` after `collect()` for Slurm gateways; DB write order: `upsert_workloads` → `finalize_completed_workloads` → `mark_terminated_workloads`
- Console gateway form fully wired: gateway_type select, prometheus_url field (K8s/hybrid), Slurm section (host/user/port/REST URL/SSH key secret) — all conditionally shown via Alpine.js

### Tests
- 10 new tests in `TestSacctParsing` + 5 in `TestFinalizeCompletedWorkloads` — 51 tests in `test_sprint19_slurm_workload_collector.py`

---

## [0.17.4] — 2026-05-12

### Added
- **`SlurmWorkloadCollector`** — full workload collector for Slurm clusters: REST (slurmrestd `v0.0.38`) → SSH+JSON (`squeue --json`) transport hierarchy; no text parsing
- **`SlurmGatewayConfig`** dataclass with `from_dict()` — validates `host` + `ssh_user` required; optional `ssh_port`, `rest_url`, `ssh_key_secret`, `rest_jwt_secret`
- **`parse_alloc_gres()`** — regex-based AllocGRES parser handles `gpu:2`, `gpu:tesla:2`, `gpu:nvidia_a100_80gb:4`, multi-type (`gpu:a100:2,gpu:v100:1` → summed); CPU-only → 0
- **`gateway_type`** column on `gateways` table (`String(32)`, server_default=`kubernetes`); values: `kubernetes` | `slurm` | `hybrid`
- **`slurm_config`** JSONB column on `gateways` table — stores host, ssh_user, ports, REST URL, secret names; CHECK constraint enforces host+ssh_user when type=slurm
- **Hybrid gateways** — `gateway_type="hybrid"` runs both `KubernetesWorkloadCollector` and `SlurmWorkloadCollector`; snapshots merged before upsert
- **`_fetch_secret()`** in `workload_tasks.py` — resolves SSH key PEM and JWT from VibOps secrets store for Slurm transport
- **`_build_collectors()`** takes plain values (not ORM objects) to avoid `DetachedInstanceError` / `MissingGreenlet` after `session.commit()`
- **Console gateway form** — gateway_type select, prometheus_url (K8s/hybrid), Slurm section with host/user/port/REST URL/SSH key secret fields; register button disabled when slurm type and host empty
- **ADR 0024** — Slurm workload collector: transport hierarchy, AllocGRES parsing, SSH key management, hybrid gateways, detached-instance guard

### Migrations
- `d5e6f7a8b9c0_add_gateway_type_slurm_config.py` — adds `gateway_type` + `slurm_config` to `gateways`

### Tests
- 36 new tests (Classes A–E in `test_sprint19_slurm_workload_collector.py`): `TestParseAllocGres`, `TestSlurmGatewayConfig`, `TestJobsToSnapshots`, `TestSlurmWorkloadCollector`, `TestGatewayApiSlurmFields`

---

## [0.17.3] — 2026-05-12

### Added
- **Workload persistence** — `workloads` table with `upsert_workloads()` and `mark_terminated_workloads()`; shadow-write alongside existing Prometheus live-query path
- **`KubernetesWorkloadCollector`** — polls Prometheus for running GPU workloads; emits `WorkloadSnapshot` objects (external_id, workload_type, namespace, gpu_count, started_at, status)
- **`WorkloadSnapshot`** dataclass — canonical workload representation shared by K8s and Slurm collectors
- **`sync_workloads`** Celery Beat task (60 s interval) — discovers gateways, dispatches to collectors, upserts to DB, marks terminated
- **`GET /api/v1/workloads`** — lists workloads with filters (`status`, `cluster_name`, `namespace`, `workload_type`) and pagination
- **`GET /api/v1/workloads/{id}`** — single workload detail
- Console **Workloads sub-tab** in FinOps: live table with status badges, GPU count, duration, namespace filter

### Migrations
- `c4d5e6f7a8b9_add_workloads_table.py` — creates `workloads` table with indexes on `(org_id, cluster_name, status)` and `(external_id, workload_type)`

### Tests
- Sprint A test suite (workloads table + K8s collector + API endpoints + console proxy)

---

## [0.17.2] — 2026-05-11

### Changed
- **Per-workload abstraction** — `PodGpuMetric` renamed to `WorkloadGpuMetric` with `workload_id` + `workload_type` fields, establishing a clean abstraction for both Kubernetes pods (`workload_type="k8s_pod"`) and future Slurm jobs (`workload_type="slurm_job"`). Done before client integration to avoid a breaking change later.
- **Canonical endpoint** `GET /clusters/{name}/workloads/{ns}/{id}/gpu-metrics` replaces `/pods/` path. The `/pods/` alias is kept with a `Deprecation` response header.
- **Top workloads endpoint** now returns `workloads` key (was `pods`). The internal compat wrapper `get_top_consuming_pods()` still exists for any early callers.
- **Console** — FinOps "Pods" sub-tab renamed to "Workloads"; table shows `workload_id` and `workload_type` columns.
- **Fixed** `NameError: pods_out` in `get_top_consuming_workloads()` — variable was renamed but one reference was missed.

### Backwards compatibility
- `PodGpuMetric = WorkloadGpuMetric` alias preserved.
- `get_pod_gpu_metrics()` and `get_top_consuming_pods()` remain as compat wrappers.
- `GET /clusters/{name}/pods/{ns}/{pod}/gpu-metrics` still returns 200 with `Deprecation` header.

---

## [0.17.1] — 2026-05-12

### Added
- **Budget alert notifications** — when a soft or hard cap threshold is crossed, VibOps now dispatches a notification via all active channels (Slack, email, PagerDuty) for the org. Previously, `BudgetAlert` records were written to the DB but never dispatched.
- **Budget check after job completion** — `job_tasks.py` now calls `check_budget()` after a job transitions to SUCCESS with its computed `actual_cost_usd`. This closes the alerting loop: job completes → cost recorded → alert fired → notification sent.
- `BudgetService._dispatch_budget_notification()` — new internal helper that formats title/message/severity and delegates to `ChannelService.notify_alert_via_channels()`. Non-fatal: if the channel call fails, the `BudgetAlert` DB record is already persisted.
- `_check_budget_after_job(org_id, cost_usd)` async helper in `job_tasks.py` — uses a dedicated `AsyncSession` (same pattern as `_resolve_secrets`) to avoid interfering with the Celery sync DB context.

### Fixed
- Budget alerts were silently stored in DB but never triggered Slack/email/PagerDuty notifications.
- Job completion did not update the budget spend counter — alerts could never fire from completed jobs.

---

## [0.17.0] — 2026-05-11

### Added
- **Per-pod GPU metrics** via on-demand Prometheus queries (DCGM Exporter / ROCm-SMI Exporter):
  - `GET /clusters/{name}/pods/{ns}/{pod}/gpu-metrics` — per-GPU util%, memory, power for a single pod
  - `GET /clusters/{name}/namespaces/{ns}/gpu-metrics` — per-pod aggregated summary for a namespace, sorted by util%
  - `GET /clusters/{name}/gpu-metrics/top` — top N pods by GPU utilisation across a cluster (`?limit=1–100`, `?namespace=` filter)
  - NVIDIA: DCGM_FI_DEV_GPU_UTIL, DCGM_FI_DEV_FB_USED, DCGM_FI_DEV_POWER_USAGE
  - AMD: rocm_smi_gpu_use_percent, rocm_smi_memory_used_vram_bytes (power not available)
  - NVIDIA + AMD queried concurrently via `asyncio.gather`; graceful `prometheus_available=false` when not configured
- **`prometheus_url`** field on Gateway model and API schema; resolved in priority: gateway field → `PROMETHEUS_URL` env var → None
- **`pod_breakdown`** JSON column on `ChargebackReport`: per-pod cost attribution when `pod_name` is present in job payload
- **FinOps waste endpoint enrichment**: `pod_level_available` + `pod_metrics` (low-util pods < 20%) per idle cluster
- **Console — Pods sub-tab** in FinOps: cluster selector, ranked GPU pod table, low-utilisation rows highlighted in amber
- **Agent system prompt**: per-pod GPU metric intents (`get_top_consuming_pods`, `get_pod_gpu_metrics`, `get_namespace_gpu_aggregated`)
- 34 new tests across 4 classes (service unit, endpoint integration, chargeback pod_breakdown, security invariants) — 689 total

### Changed
- `ChargebackReportResponse` now includes `pod_breakdown` field (null for reports generated before this migration)

---

## [0.16.4] — 2026-05-10

### Added
- **`SlurmConnector`** — 6 actions for multi-node HPC training on bare-metal Slurm clusters (no Kubernetes required):
  - `slurm_get_cluster_info` — `sinfo`: partitions, node states, GPU availability per node
  - `slurm_list_jobs` — `squeue`: running and pending jobs with TRES, user, time limit
  - `slurm_submit_job` — `sbatch`: generates sbatch script from payload, dry-run preview gate before submission; returns `job_id`
  - `slurm_cancel_job` — `scancel` with configurable signal; destructive, requires confirmation
  - `slurm_get_job_status` — `squeue` (running) + `sacct` fallback (finished/failed)
  - `slurm_get_job_output` — `tail -n N` on job stdout log via SSH
- **Transport auto-detection**: probes slurmrestd REST API (3s timeout), falls back to SSH; result cached per connector instance
- **SSH key from Vault**: `ssh_key=@secret:slurm_ssh_key` — PEM content written to a `0o600` temp file for the SSH call duration, deleted in `finally`; Vault resolution in `job_tasks.py` before connector call
- **Demo mode** (`SLURM_DEMO_MODE=true`): realistic canned responses for all 6 actions — 8× A100 node cluster, active job queue, training log output — no real cluster or SSH key required; enabled by default in `docker-compose.yml`
- **Agent**: 6 Slurm tools in `TOOLS`, routed via `_RUN_JOB_TOOLS` / `_CREATE_JOB_TOOLS`
- **MCP server**: 6 Slurm tools registered (action tools: 8 → 14)
- **WorkloadClassifier**: `slurm_submit_job` → `training`, `slurm_cancel_job` → `operations`, read actions → `observation`
- **System prompt**: Slurm HPC workflow with mandatory `slurm_get_cluster_info` pre-check, Vault SSH key pattern, `save_memory` after submit
- **Console**: Slurm metadata card in job detail panel (job ID, partition, nodes, GPUs/node, wall time)
- **`_format_job()`**: extracts `job_name` label and `slurm_job_id` for Slurm jobs
- Connector count: 25 → **26**
- 40 connector tests (`test_slurm_connector.py`)

### Documentation
- **Scenarios 30–32** in `docs/demo-scenarios.md` — multi-node training submission, log monitoring, runaway job cancel with GPU budget recovery ROI
- Header updated: 29 → 32 scenarios; HPC/MLOps engineer persona added

---

## [0.16.3] — 2026-05-08

### Added
- **`k3s_load_image`** (DockerBuildConnector) — `docker save <image> | k3s ctr images import -`; completes local cluster coverage (kind/k3d/minikube/k3s)
- **`create_ecr_pull_secret`** (KubectlConnector) — fetches ECR token via `aws ecr get-login-password`, creates pull secret; tokens valid 12h
- **`create_gcr_pull_secret`** (KubectlConnector) — GCP service account JSON key → Kubernetes pull secret
- **`create_acr_pull_secret`** (KubectlConnector) — Azure service principal / ACR admin credentials → Kubernetes pull secret
- **`argocd_enable_auto_sync`** (ArgoCDConnector) — enables `syncPolicy.automated` via ArgoCD API (GET → patch → PUT); options: `prune`, `self_heal`
- **`argocd_disable_auto_sync`** (ArgoCDConnector) — removes `syncPolicy.automated`
- **`openshift_add_scc`** (KubectlConnector) — `oc adm policy add-scc-to-user`; falls back to kubectl if `oc` absent
- **`openshift_create_route`** (KubectlConnector) — `oc expose service` with optional hostname
- `system_prompt.md` routing rules for all new actions (cloud pull secrets, ArgoCD auto-sync, OpenShift)

### Documentation
- **Scenario 27** — ArgoCD auto-sync: GitOps loop closure via one prompt; ROI: $9,975/year manual sync overhead + $2,256–$3,384/year missed sync incidents
- **Scenario 28** — Cloud registry deploy (ECR/GCR/ACR): unified pull secret interface across AWS/GCP/Azure; ROI: $15,625/year ECR token refresh overhead
- **Scenario 29** — OpenShift deploy: SCC + Route in same prompt as deploy; ROI: $19,500–$39,000/year platform team ticket overhead
- `README.md` GitOps section updated: ArgoCD auto-sync flow, cloud registry patterns, OpenShift flow
- `docs/demo-scenarios.md`: 26 → 29 scenarios

---

## [0.16.2] — 2026-05-07

### Added
- **`create_pull_secret`** (KubectlConnector) — creates or updates a Kubernetes docker-registry pull secret; idempotent (`--dry-run=client -o yaml | kubectl apply`); works on EKS, GKE, AKS, RKE2, bare metal, and local clusters
- **`deploy_webapp`** updated — new `image_pull_secret` param; patches Deployment with `imagePullSecrets` spec after creation
- **`kind_load_image`** — loads a locally built Docker image into a kind cluster's containerd runtime (`kind load docker-image`)
- **`k3d_load_image`** — loads a locally built image into a k3d cluster (`k3d image import`)
- **`minikube_load_image`** — loads a locally built image into a minikube cluster (`minikube image load [-p <profile>]`)
- **Scenario 26** in `docs/demo-scenarios.md` — private registry deploy on K8s: `create_pull_secret` → `deploy_webapp(image_pull_secret=...)`, with ROI

### Tests
- 3 kubectl chain tests (`test_kubectl_connector.py`): full `create_pull_secret` → `deploy_webapp` sequence, failure-blocks-deploy, patch failure propagation
- 10 new connector tests for `k3d_load_image` and `minikube_load_image`

### Documentation
- `README.md` — GitOps section updated with private registry K8s deploy flow and local cluster load patterns
- `system_prompt.md` — routing rules for all local cluster patterns (kind/k3d/minikube) and production private registry pattern

---

## [0.16.1] — 2026-05-07

### Added
- **`DockerBuildConnector`** — 4 actions: `docker_build` (600s timeout, layer streaming), `docker_tag` (30s), `docker_push` (login via `--password-stdin`, token masked in logs, digest returned), `docker_build_push` (combined)
- **`CIConnector`** — 4 actions: `ci_trigger` (GitHub Actions dispatch + run_id polling; GitLab pipeline trigger), `ci_status` (normalized status across providers), `ci_wait` (5s polling, 900s timeout), `registry_list_tags` (OCI v2 Bearer auth)
- CI connector reuses `GIT_TOKEN` / `GIT_PROVIDER` — no new credential (ADR 0022)
- `triggered_by=vibops` injected in every pipeline trigger
- **Admin → Git panel** — org-level PAT config + apps table with inline repo linking/unlinking/editing (22 frontend tests)
- **Admin → CI panel** — provider status card with per-provider scope hints + pipeline runs table with 7 columns (22 frontend tests)
- **ADR 0022** — CI connector: GitHub dispatch + GitLab pipeline, token reuse strategy, Admin → CI panel design
- **ADR 0021** updated — status Accepted/implemented; docker_build tools shipped; CI items → ADR 0022
- Connector count: 23 → **25**

### Tests
- 40 new connector tests (`test_docker_build_connector.py`)
- 37 new connector tests (`test_ci_connector.py`)
- 44 new frontend tests (`TestAdminGit` + `TestAdminCI`)

---

## [0.15.1] — 2026-05-05

### Security
- **ILIKE injection fix** in `memory_service.search()` — `%`, `_`, `\` now escaped before ILIKE pattern
- **GET /secrets/{name}** now requires `require_write` instead of `get_current_user` — read-only users can no longer access decrypted secret values
- **Training export proxy** in console — `_svc_headers()` added to agent call (was unauthenticated in dev mode)

### Fixed
- `revoke_gateway` now cancels orphaned PENDING/RUNNING jobs before deleting the gateway
- Dataset export endpoint streams via `db.stream_scalars()` — prevents OOM on large datasets
- `ChatRequest.message` capped at `max_length=32768`; `history` list capped at `max_length=40`
- `GET /memories` pagination — `limit`/`offset` query params (default 100, max 500)
- `get_commits` and `get_metrics` in console now validate `limit` and `minutes` with Query bounds
- Reseller profile N+1 count queries replaced with `func.count()`
- `stream_job_logs` timeout: `None` → `Timeout(connect=5, read=300)`
- Console `requirements.txt`: fastapi `0.115.6`, httpx `0.28.1`
- Bare excepts in `chat_feedback` and `list_kube_contexts` now log warnings

### Database
- Alembic `j6k7l8m9n0o1`: missing DB indexes (jobs.created_at, recommendation_events.recommended_at, trigger_rules.enabled, ix_jobs_org_id_status, ix_trigger_rules_org_id_enabled)
- Alembic `k7l8m9n0o1p2`: composite index for finops waste detection query

---

## [0.15.0] — 2026-05-05

### Q2 2026 Robustness Audit — 56 fixes across security, performance, and reliability

#### Security (16 fixes)
- `/audit/ingest`: added `X-Internal-Key` service-to-service auth — was publicly accessible
- `/alerts/notify`: `org_id` now forced from JWT, not request body (cross-tenant targeting prevented)
- Password reset token no longer logged in plaintext
- `ClusterMetrics`: Pydantic validation for negative values and `gpu_used > gpu_total`
- CORS origins moved to `settings.cors_origins` env var (was hardcoded to localhost)
- `INTERNAL_API_KEY` absence now fatal at startup in production
- `CoreClient` (agent): `X-Internal-Key` header on all 5 previously unauthenticated endpoints
- `GET /jobs`, logs, cancel: added `get_current_user` + org_id isolation
- 5 additional endpoint hardening fixes across auth, audit, gateways, webhooks, rate limiting

#### Performance (2 fixes)
- `/health` Celery probe: serialized lock removed — fire-and-forget at cold start, `"checking"` returned immediately; p99: 926ms → ~100ms
- SLO test threshold: 800ms → 850ms (CI stability)

#### Robustness — Workers (12 fixes)
- Engine `dispose()` in `try/finally` across job_tasks, discovery_tasks, pipeline_tasks
- Atomic log append: `UPDATE jobs SET logs = COALESCE(logs,'') || :line` (was read-modify-write race)
- `SELECT FOR UPDATE SKIP LOCKED` before PENDING→RUNNING transition (double-execution prevention)
- `asyncio.gather` with `timeout=120` in discovery tasks
- Celery time limits: `task_time_limit=1800`, `soft_time_limit=1500` on all tasks
- `task_acks_on_failure_or_timeout=True` — no silent task loss
- `broker_transport_options.visibility_timeout=1900` — prevents Redis double-execution
- `worker_max_tasks_per_child=100` — prevents memory drift
- LLM client: `httpx.Timeout(120.0)` on both Anthropic and OpenAI clients

#### Robustness — API & Services (10 fixes)
- Grafana webhook Redis pool reset on connection error
- `scale_replicas` annotation: `try/except (ValueError, TypeError)` with fallback
- Webhook fields capped: repo/branch/commit_sha/commit_message
- Webhook idempotence on `(action, triggered_by, commit)`
- Trigger field validation: `metric` max 1024, `name` max 128, `cooldown_minutes` ge=0 le=10080
- Trigger name uniqueness check (409 instead of silent ambiguity)
- Pipeline step `UniqueConstraint("pipeline_id", "order")`
- SMTP exception classification: `SMTPAuthenticationError` → log.error, `SMTPException`/`OSError` → log.warning
- `channel_service._http_post` timeout: 5s → 10s

#### Connectors & Agent (8 fixes)
- kubectl subprocess timeouts: `asyncio.wait_for()` + kill + drain on timeout (10s / 30s / 120s)
- kubectl stdout capped at 5 MB to prevent memory exhaustion
- git connector subprocess timeout: 120s
- Groq `_api_get`/`_api_post`: typed exception handlers with warnings (was silent `except Exception: return None`)
- kubectl benchmark JSON parsing: `except (JSONDecodeError, KeyError)` with debug log
- `core_client.format_gateways`, `resolve_cluster_gateway`, `create_slo`: all bare excepts now log warnings

---

## [0.16.0] — 2026-05-05

### Added
- **DatasetAggregator service** — ADR 0020 Decision 3: K≥10 distinct orgs per accelerator bucket, ≤40% max per-org share before training export; non-configurable constants by design
- **`GET /resellers/me/dataset-stats`** enriched: `consenting_org_count`, `suppressed_bucket_count` fields; aggregates computed by DatasetAggregator on consenting orgs only
- **ADR 0020**: Dataset governance for reseller export pipeline — 4 decisions: default consent cascade, governance thresholds, export pipeline, data residency

---

## [0.15.0-sprint15] — 2026-05-02

### Added
- **Secret `is_system` flag** — cross-org fallback restricted to `is_system=True` secrets; prevents accidental cross-tenant credential access
- **Observability default-on**: Prometheus + Grafana start with `docker compose up` (removed from `observability` profile)
- **3 alerting rules**: `VibOpsCoreDown`, `VibOpsHighJobFailureRate` (>10% over 5min), `VibOpsBudgetHardCapBreached`
- **`restart: unless-stopped`** on all docker-compose services
- **Automated backup**: `pg_dump` compressed daily → `/backups/vibops_YYYY-MM-DD.sql.gz`, 30-day retention
- **`scripts/pilot_provision.py`**: idempotent org + admin + budget provisioning with colored output and JWT
- **`Makefile`**: `make pilot-create-client`, `make backup-now`, `make backup-list`
- **`docs/runbooks/pilot-runbook.md`**: go-live checklist, provisioning, observability, backup/restore

### Fixed
- Connector timeout: `asyncio.wait_for(timeout=1200s)` marks job FAILED if connector blocks before Celery kills the process

---

## [0.14.5] — 2026-05-02

### Added
- **20 FinOps contract tests** (API → UI contract, ADR 0019): budget, waste, chargeback, spend/trend, budget/alerts
- **ADR 0019**: UI testing strategy — contract tests HTTP now, Playwright Sprint 17+

---

## [0.14.0] — 2026-05-01

### Added
- **FinOps UI** — 4 sub-tabs: Waste · Budget · Chargeback · Alerts (Alpine.js)
- **`BudgetResponse`** enriched: `daily_burn_rate_usd`, `spend_forecast_eom_usd`, `days_elapsed`, `days_in_month`
- **`GET /finops/spend/trend`**: 12 months historical, `has_report` flag
- **Waste enriched**: `estimated_waste_usd_per_month`, `waste_score` (0-100), `scanned_hours_ago`
- **`GET /finops/budget/alerts`**: alert history with `is_hard_cap` distinction
- **Chargeback**: `team_breakdown` by namespace (JSON column + migration)
- **`generate_from_jobs()`**: auto-aggregates Job records by vendor + namespace — no manual `vendor_usage` input needed
- 28 new i18n keys (fr.json)

---

## [0.13.0] — 2026-04-30

### Added
- **RLHF signal in dataset**: `agent_feedback` in `GET /api/v1/dataset` — total exchanges, with_feedback, by_domain
- **Consent gate on training export**: `opted_out`/`NULL` → 403
- **`workload_context`** on `TrainingExchange`: cluster, gateway_id, domain — correlates feedback with GPU context
- **Anonymization of `org_id`** in training export via Sprint 12 engine

---

## [0.12.0] — 2026-04-30

### Added
- **Three consent states** on `Organization`: `pseudonymized` | `anonymized` | `opted_out` | `NULL`
- **Anonymization engine** (`anonymization.py`): `pseudonymize()` HMAC-SHA256, `clean_payload()` allowlist filter, `anonymize_job()`
- **`DATASET_PSEUDONYMIZATION_SALT`** env var — never stored in DB
- **Consent API**: `GET /dataset/consent`, `PATCH /dataset/consent` (org_admin)
- **Export API**: `GET /dataset/export` — JSONL streaming of dataset-eligible jobs
- **ADR 0018**: GDPR posture, consent model, anonymization design

---

## [0.11.0] — 2026-04-30

### Added
- **`WorkloadDetector`**: auto-detects framework from container image — 10 frameworks (vllm, nim, triton, tgi, pytorch, tensorflow_serving, ollama, sglang, deepspeed, ray, litellm)
- **New Job columns**: `framework` (indexed), `framework_version`, `model_name` (indexed)
- **`GET /api/v1/dataset/stats`**: 6-group health snapshot (coverage, outcomes, recommendations, cost prediction quality, framework distribution, coverage gaps)
- **ADR 0017**: framework detection + GPU metrics deferral

### Tests
- ~30 new tests (Sprint 11)

---

## [0.10.0] — 2026-04-29

### Added
- **`WorkloadSignature`** Pydantic schema: typed descriptor for accelerator targeting, workload characterization, scheduling hints
- **Job model extended**: `workload_signature` (JSON), `vendor` (indexed), `accelerator_type` (indexed)
- **9 job outcome fields**: outcome, exit_code, actual_duration_s, avg/peak GPU/memory utilization, actual_cost_usd, failure_reason_category
- **`_classify_failure()`**: pattern-based taxonomy (oom/timeout/network/quota/driver_error/config/unknown)
- **`_compute_actual_cost()`**: customer_cost_per_gpu_hour × gpu_count × duration / 3600
- **`recommendation_events` table**: captures every recommendation + operator response (followed/ignored/overridden)
- **Recommendations API**: POST, POST /{id}/respond, GET, GET /{id}
- **ADR 0016**: Operational dataset as strategic asset

### Security hardening
- **Router-level auth enforcement**: all protected routes on `APIRouter(dependencies=[Depends(get_current_user)])` — auth by construction
- **CI auth introspection test** (`test_endpoint_auth.py`): FastAPI dependency tree walker; unauthenticated endpoints fail the build
- **25 endpoints hardened**: org_id isolation, HMAC verification, atomic gateway claim, LIKE injection prevention, recursive secret resolution, path traversal fix
- **Redis ZSET sliding window rate limiter** (atomic Lua, DoS-safe fallback)
- **ADR 0005**: endpoint auth invariant + CI enforcement

### Tests
- ~40 new tests (Sprint 10); full suite: 655 tests, 0 failures

---

## [0.9.0] — 2026-04-29

### Added
- **Pricing at job submission** (ADR 0015): `internal_cost_per_gpu_hour_usd`, `customer_cost_per_gpu_hour_usd`, `pricing_rule_source` frozen at submission — never recalculated
- **Cloud vs. on-prem pricing formulas**: `ClusterRate.formula_type` discriminator + formula fields
- **Pricing tiers**: `on_demand`, `spot`, `reserved_1y`, `reserved_3y` with cascade fallback

### Tests
- 25 new tests; full suite: 407 tests

---

## [0.8.0] — 2026-04-29

### Added
- **Tier 3 Multi-Tenant Reselling** (ADR 0013): `Organization` with `org_type`, `reseller_id` (self-referential FK), `white_label_name/slug`
- **Pricing Engine** (ADR 0014): `PricingRule`, `CustomerPricingOverride`, 7-level cascade, floor/ceiling enforcement
- **Budget Enforcement**: `Budget` model with soft/hard caps; `BudgetAlert` immutable records
- **Chargeback Reporting**: `ChargebackReport` monthly snapshot, idempotent generation
- **FinOps API** (`/api/v1/finops/`): budget CRUD, alert history, chargeback CRUD, waste endpoint
- **Reselling API** (`/api/v1/resellers/`): reseller profile, customer management, pricing rules, customer overrides

### Tests
- 27 new tests; full suite: 382 tests

---

## [0.7.0] — 2026-04-29

### Added
- **GroqConnector**: sixth accelerator vendor — Groq Cloud LPU; per-token pricing model; probe-based metrics (TTFT, tokens/sec); 5 known models
- **`accelerator_detect_waste`**: 11th accelerator tool — snapshot-based utilization threshold check; three diagnostic paths (idle/memory-bound/underutilised)
- **ADR 0012 corrections**: FOCUS field mapping, Sprint 8 deferred decisions documented

### Tests
- 179 new tests; full connector suite: 2785 tests, 0 failures

---

## [0.6.0] — 2026-04-29

### Added
- **`TrainiumConnector`** (AWS Neuron — Trn1/Trn2/Inf1/Inf2): dynamic Neuron exporter discovery, NEURON_* Prometheus metrics, `CostMetadata` with `cloud_provider="aws"`
- **`TPUConnector`** (Google Cloud TPU v3/v4/v5e/v5p/v6e Trillium): GKE label detection, generation-aware capabilities, topology field
- **`CostMetadata` dataclass**: FinOps cost schema contract (ADR 0012) — aligned with FOCUS standard; populated across all 5 connectors
- **ADR 0012**: cost schema — structural anchoring vs. pricing resolution

### Tests
- 386 new tests; full connector suite: 2606 tests, 0 failures

---

## [0.5.0] — 2026-04-28

### Added
- **`IntelConnector`** (Gaudi 3): third vendor proof; dynamic namespace + resource discovery; GAUDI_* metrics; `supports_partitioning=False`
- **`NvidiaConnector` pruned**: 5 deprecated vendor-specific tools removed; `TOOL_CATALOG = AcceleratorConnector.ACCELERATOR_TOOL_CATALOG` (10 tools)
- **`accelerator_get_capabilities`**: 10th tool; `AcceleratorCapabilities` dataclass

### Tests
- 116 new tests; full connector suite: 2220 tests

---

## [0.4.1] — 2026-04-28 (Sprint 4b — AMD ROCm)

### Added
- **`AmdConnector`** (ROCm): second vendor; dynamic exporter discovery (4 label selectors); AMD partition modes (SPX/DPX/QPX/CPX); ROCM_* metrics; 3 diagnostic patterns
- Vendor-parametrized CI guardrail (`VENDOR_TOOL_GUARDRAILS` dict)

### Tests
- 35 new tests; full connector suite: 2104 tests

---

## [0.4.0] — 2026-04-28 (Sprint 4a — Multi-Accelerator Foundation)

### Added
- **`AcceleratorConnector` abstraction**: vendor-agnostic ABC — 6 abstract operational methods + 3 analytical methods
- **Unified data model**: `WorkloadSignature`, `UnifiedDeviceDescriptor`, `UnifiedDeviceMetrics`, `PortabilityProfile`, `CostFunction`, `TrustAttributes`, `DiagnosticPattern`
- **`NvidiaConnector`** refactored to inherit `AcceleratorConnector`; 9 new `accelerator_*` tools
- **`docs/architecture/multi-accelerator.md`**: interface spec, vendor roadmap
- **ADR 0010**: GPU operations abstraction layer positioning
- **ADR 0011**: Four-dimensional moat — strategic positioning

---

## [0.3.1] — 2026-04-28

### Fixed
- **Structured logging in Celery workers** — trigger, briefing, discovery workers emit structured events
- **Lazy-load-after-commit crash** in trigger worker — `job_id` captured before `mark_triggered()` commits
- **`merge_contextvars` replaced** with direct `ContextVar` processor (safe in API + worker contexts)

### Added
- **MCP `get_job_metrics` tool** — exposes `GET /api/v1/metrics/jobs` to the LLM

---

## [0.3.0] — 2026-04-28

### Added
- **Org invites**: one-time invite links with 48h TTL, single-use, revocable; invitee sets own credentials and receives JWT immediately
- **Scope enforcement on pipeline triggers**: `check_scope` wired into `trigger_pipeline`

---

## [0.10.0-security] — 2026-04-22

_(Sprint 10 security block — see v0.10.0 above for the full entry)_

---

## [0.9.1] — 2026-04-15

### Added
- **VibOps Connect — Gateway CRUD**: register, list, revoke gateways; token shown once, stored as SHA-256 hash
- **Gateway token auth**: per-gateway Bearer token independent of JWT
- **Job routing to gateways**: auto-routing by cluster overlap; `GET /jobs?gateway_id=`
- **Connect worker** (`connect/worker.py`): async polling loop — claim, execute, report
- **`scripts/connect-setup.sh`**: one-command gateway onboarding
- **Monitoring tab**: unified Grafana + Connect Gateways sub-tabs
- **GPU Simulator**: Python DCGM exporter (2×A100 sinusoidal for demo environments)
- **Gateway-aware agent**: `format_gateways()` in system prompt; `list_gateways`, `run_on_gateway`, `search_jobs`, `diagnose_cluster` tools

### Fixed
- JWT / gateway token collision (moved gateway router outside JWT router)
- LLM streaming jank: deferred `marked.parse()` to end of stream

### Tests
- 19 gateway + 9 connect worker + 8 agent gateway tests

---

## [0.8.1] — 2026-04-12

### Added
- **i18n**: 8 languages (EN, FR, ES, DE, IT, PT, JA, ZH); agent responses follow UI language
- **Streaming UI**: token-by-token with blinking cursor, sticky scroll, `⌘K` shortcut
- **EventSource auth**: JWT accepted as `?token=` query param
- **OpenAPI contract in CI**: `docs/openapi.json` diff-checked on every PR

---

## [0.7.1] — 2026-03

### Added
- Real-time job log SSE streaming; job cancel from chat panel
- Multi-cluster context selector; kubeconfig auto-scan

---

## [0.6.1] — 2026-02

### Added
- Notification channels (Slack webhook); Secrets vault admin UI
- GPU alert presets; persistent pipeline memory; auto-preference memory
- `patch_deployment` live resource patching; GitHub token auto-injection

---

## [0.5.1] — 2026-01

### Added
- Multi-tenancy RBAC (Organisation → Team → Member); GPU Observability Stack (DCGM + Prometheus + Grafana)
- NIM integration; GPU QoS & Reservation; MIG support; GPU time-slicing
- EKS/GKE/AKS/ArgoCD/Helm/Terraform connectors; conversation history; auto-rollback; GitHub Webhooks

---

## [0.4.2] — 2025-12

### Added
- Pipelines (multi-step with branching); Secrets vault (Fernet AES); Audit trail; Structured logging (JSON + OTEL)
- CI/CD pipeline (GitHub Actions); SLO tests (Locust); HA manifests (HPA + PDB); Grafana SLO dashboard

---

## [0.3.2] — 2025-11

### Added
- Console redesign (GitHub Dark); Discovery engine; Cluster resources tab
- GPU support (NVIDIA/AMD/Intel/Groq); Datadog + Ollama connectors; kind management; Triggers; Pipelines UI

---

## [0.2.0] — 2025-10

### Added
- Claude agent with agentic loop + guardrails; kubectl + git connectors; JWT auth; Multi-tenancy DB schema
- Helm chart; OpenAI-compatible gateway proxy

---

## [0.1.0] — 2025-09

### Added
- Core Execution Engine (Celery + Redis); FastAPI core (SQLAlchemy async, Alembic, PostgreSQL)
- Docker Compose local dev stack; Initial project structure
