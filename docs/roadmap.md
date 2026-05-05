# VibOps — Technical Roadmap

_Last updated: 2026-05-03 · v0.15.0-sprint15_

## Principles

- Ship value iteratively — no big bang releases
- Each item is independent and can be prioritized separately
- Connect Gateway items are gated on having a first Connect client signed
- Enterprise segment features are as valid as CSP features — roadmap serves both

---

## Done — shipped ✓

### Core platform
- [x] Multi-cluster discovery and resource audit
- [x] Production safety guardrails (confirmation required for destructive ops)
- [x] Dry-run preview — `reversibility`, `resolved_params`, `estimated_cost_hourly_usd`
- [x] PolicyEngine default-deny — every action must be in TOOL_CATALOG (ADR 0001)
- [x] Two-tier authorization — `evaluate()` for API callers, `evaluate_system()` for Celery workers (ADR 0002)
- [x] Multi-tenant isolation — row-level `org_id` from JWT, never from request body (ADR 0003)
- [x] Rate limiting — sliding window, Redis ZSET, 60 req/60s/org (ADR 0004)
- [x] Endpoint auth invariant — router-level enforcement + CI test (ADR 0005)
- [x] Structured logging (structlog JSON, hot-path events on all critical paths)
- [x] Request correlation — `RequestIdMiddleware`, `X-Request-ID` propagated to workers
- [x] Job SLIs — `GET /api/v1/metrics/jobs` (throughput, success rate, p50/p95 by action)
- [x] Secrets vault — Fernet-encrypted, recursive payload resolution, never logged
- [x] Pipelines — multi-step with `on_failure: rollback` guards, `evaluate_system()` gate per step
- [x] Trigger rules — event-driven automation (GPU utilization, alert state, schedule), AND/OR logic
- [x] Org invites — one-time link, 48h TTL, single-use, revocable

### Multi-Accelerator Abstraction (Sprints 4a–7)
- [x] `AcceleratorConnector` ABC — vendor-agnostic interface over all GPU/accelerator vendors
- [x] Unified data model — `WorkloadSignature`, `UnifiedDeviceDescriptor`, `UnifiedDeviceMetrics`
- [x] 11 unified tools — `accelerator_diagnose`, `accelerator_detect_waste`, `accelerator_workload_match`, `accelerator_get_metrics`, `accelerator_deploy_workload`, `accelerator_portability_check`, `accelerator_cost_estimate`, `accelerator_install_operator`, `accelerator_list_devices`, `accelerator_partition_device`, `accelerator_get_capabilities`
- [x] NVIDIA connector refactored — inherits `AcceleratorConnector`, vendor-specific tools removed from public API
- [x] AMD ROCm connector — SPX/DPX/QPX/CPX partitioning, dynamic exporter discovery
- [x] Intel Gaudi connector — dynamic namespace/resource detection, GAUDI_* metrics
- [x] AWS Trainium/Inferentia connector — Neuron SDK, Trn1/Trn2/Inf1/Inf2
- [x] Google TPU connector — v3/v4/v5e/v5p/v6e, topology-aware, GKE label detection
- [x] Groq connector — LPU managed service, per-token cost model, probe-based metrics
- [x] CI guardrails — `test_no_new_nvidia_specific_tools_when_accelerator_equivalent_exists`
- [x] ADR 0010 — GPU operations abstraction layer positioning
- [x] ADR 0011 — Four-dimensional moat strategic positioning
- [x] ADR 0012 — Accelerator cost schema (structural anchoring vs pricing resolution)

### FinOps Engine (Sprints 8–9, 14–14.5)
- [x] Tier 3 reselling architecture — `org_type`, `reseller_id`, white-label name/slug (ADR 0013)
- [x] Pricing engine — 7-level specificity cascade, floor/ceiling, markup transparency (ADR 0014)
- [x] Budget enforcement — soft cap (alert) + hard cap (block), deduplication
- [x] Chargeback reporting — monthly snapshot per org, idempotent generation, vendor breakdown
- [x] At-submission pricing — prices frozen at job creation, never recalculated (ADR 0015)
- [x] Cloud vs. on-prem formulas — `formula_type` discriminator, TCO normalization
- [x] Pricing tiers — `on_demand`, `spot`, `reserved_1y`, `reserved_3y`
- [x] `accelerator_detect_waste` — snapshot-based idle GPU detection, vendor-agnostic
- [x] **Sprint 14** — FinOps UI 4 sous-onglets (Waste / Budget / Chargeback / Alerts)
- [x] **Sprint 14** — Budget live spend from jobs (`_get_monthly_spend` fallback on Job records)
- [x] **Sprint 14** — EOM forecast (`daily_burn_rate`, `spend_forecast_eom_usd`)
- [x] **Sprint 14** — `generate_from_jobs()` — chargeback auto depuis Job records, plus besoin de `vendor_usage`
- [x] **Sprint 14** — `team_breakdown` sur ChargebackReport — coûts par namespace
- [x] **Sprint 14** — `GET /finops/spend/trend` — 12 mois historiques + sparkline UI
- [x] **Sprint 14** — Waste enrichi : `waste_score` (0-100), `scanned_hours_ago`, `estimated_waste_usd_per_month`
- [x] **Sprint 14.5** — 20 contract tests HTTP couvrant tous les endpoints FinOps (ADR 0019)

### Operational Dataset & RLHF (Sprints 10–13)
- [x] `WorkloadSignature` — typed descriptor for accelerator targeting + data governance
- [x] Job outcome tracking — `outcome`, `failure_reason_category`, `actual_cost_usd`, duration
- [x] `_classify_failure()` — typed taxonomy: oom / timeout / network / quota / driver_error / config
- [x] `RecommendationEvents` — captures followed/ignored/overridden signals
- [x] Dataset stats API — `GET /api/v1/dataset/stats` (6-group health snapshot)
- [x] Framework auto-detection — `WorkloadDetector` on container image (10+ frameworks)
- [x] Consent model — `pseudonymized` / `anonymized` / `opted_out` per org (ADR 0018)
- [x] Anonymization engine — HMAC-SHA256 pseudonymization, allowlist payload filter
- [x] Export API — JSONL jobs + training exchanges (alpaca/sharegpt/chatml formats)
- [x] RLHF feedback loop — thumbs up/down per agent response, wired into training export
- [x] `workload_context` on `TrainingExchange` — cluster/gateway/domain correlation

### Agent Behavioral Model
- [x] 13 mandatory rules in system prompt (ADR 0007) — act directly, parallel execution, anti-loop, confirmation flow, etc.
- [x] Language policy — agent responds in user's language (ADR 0008)
- [x] 3-layer testing stack — L1 form, L2 coherence, L3 behavioral (ADR 0009)
- [x] Incident response workflow — `correlate_incident` → `analyze_pod_failure` → `remediate_incident`
- [x] Epistemic honesty — `confidence` field, "not observed ≠ not present"
- [x] Multi-accelerator routing — `accelerator_*` tools mandatory, vendor-specific tools FORBIDDEN
- [x] GitOps workflow — `git_clone` → `git_patch_yaml` → `git_commit_push` → `git_create_pr`
- [x] NIM workflow — `nim_list_catalog` → `nim_profiles` → GPU audit → `nim_deploy`
- [x] Memory system — `save_memory` / `recall_memory`, proactive saving after incidents and decisions

### Observability
- [x] Prometheus webhook integration (`POST /api/v1/webhook/grafana`)
- [x] Alert JSON → natural language → agent pipeline
- [x] Multi-source incident correlation (`correlate_incident` — logs + events + metrics + deployment)
- [x] Prometheus missing detection + auto-install offer (helm kube-prometheus-stack)
- [x] SLO monitoring — `create_slo`, breach triggers remediation or trigger rule

### Multi-tenancy & Auth
- [x] Organisation → Team → Member RBAC model
- [x] JWT enriched with org + teams + scope
- [x] Admin panel: Teams / Users / Audit / Memories / Notifications / Secrets / Integrations / Licence
- [x] Password change, refresh token (7 days)
- [x] **Sprint 15** — Tier 3 secret isolation (`is_system` gate — cross-org fallback restreint aux shared credentials explicites)

### Licence & Packaging
- [x] RS256 JWT licence — vendor holds private key, clients cannot forge
- [x] Trial 14 days, plan enforcement (users_max, clusters_max, gpu_max)
- [x] Helm chart `helm/vibops` — HPA, PDB, ServiceMonitor, TLS
- [x] `scripts/package-delivery.sh` — air-gapped delivery package
- [x] `scripts/onboard-client.sh` — CSP / Enterprise segment-aware (Helm/K8s)
- [x] `scripts/update-docs.sh` — post-sprint doc bump (version, date, counts, Known limitations)
- [x] **Sprint 15** — `make pilot-create-client` — provisioning Docker Compose pilot en une commande
- [x] **Sprint 15** — `docs/runbooks/pilot-runbook.md` — checklist go-live complète

### Résilience & Observabilité (Sprint 15)
- [x] `restart: unless-stopped` sur tous les services docker-compose
- [x] Service `backup` — `pg_dump` quotidien compressé, rétention 30 jours
- [x] Timeout connecteur `asyncio.wait_for(1200s)` — FAILED propre avant kill Celery
- [x] Prometheus + Grafana default-on (suppression du profil `observability`)
- [x] `GET /health` enrichi avec check worker Celery
- [x] 3 alerting rules : service down, failure rate >10%, budget hard cap
- [x] Counter `vibops_jobs_total{action, status}` dans le worker

### MCP Server (`VibOpsai/vibops-mcp`)
- [x] 16 observation tools — clusters, deployments, jobs, GPU metrics, MTTR, cost, gateways, alerts, pipelines…
- [x] 8 action tools — scale, deploy, helm upgrade/uninstall, kubectl, git clone, create secret, trigger pipeline
- [x] 3 config tools — set cluster rate, register/delete gateway

### Connect Gateway
- [x] `vibops-worker` standalone image + Helm chart `charts/vibops-connect`
- [x] Gateway heartbeat + cluster metrics push
- [x] Bearer token auth, atomic job claim
- [x] Onboarding wizard in console

---

## P1 — Backlog (prioritized)

### FinOps maturity
- ~~[ ] `accelerator_detect_waste` — time-series mode: sustained underutilisation over N hours (not just snapshot)~~ ✓ Sprint 15
- ~~[ ] Chargeback generation — automated monthly Celery Beat task (currently admin-triggered)~~ ✓ Sprint 15
- [ ] Cloud pricing API integration — live AWS/GCP/Azure GPU rates (currently manual ClusterRate)
- [ ] Currency conversion — multi-currency support (currently USD only)
- [ ] White-label routing — custom domains per CSP via `white_label_slug`
- [ ] FinOps UI — consent management + dataset export controls in console (ADR 0018 — Decision 5)
- ~~[ ] Budget enforcement on pre-Sprint 9 jobs — sum Job records instead of ChargebackReport~~ ✓ Sprint 14

### Dataset & RLHF maturity
- [ ] Dataset UI — consent management and export controls in console
- [ ] GPU utilization per-job — per-pod DCGM via gateway (attribution currently impossible with concurrent workloads)
- ~~[ ] Salt rotation migration plan for `DATASET_PSEUDONYMIZATION_SALT`~~ ✓ Sprint 15
- ~~[ ] Reseller consent ownership — which org sets consent for reseller_customer orgs~~ ✓ Sprint 15 (ADR 0020 Decision 2)

### Platform
- [x] Per-cluster role assignments — user X = operator on prod, readonly on dev
- [ ] Spot preemption enforcement — scheduling-level, not just metadata
- [ ] Committed billing enforcement — reserved_1y/3y tiers beyond metadata
- [ ] Vendor/accelerator_type explicit in payload schema (currently heuristic extraction)
- [ ] `scale_cluster` up/down split (ADR 0002 deferred)

### Intelligence
- [ ] Proactive incident detection — agent monitors metrics autonomously between sessions
- [ ] Alert correlation across multiple services
- [ ] Predictive GPU failure — temperature trends + DCGM error patterns → warn before incident
- [ ] L2 LLM-as-judge scanner — non-blocking, catches subtle prompt↔schema contradictions

### Connect Gateway
- [ ] Gateway capability discovery endpoint — declares which connectors it exposes
- [ ] mTLS option between gateway and core

### Image build & CI connector
- [ ] `docker_build` tool — build a Docker image from a Git repo or local context, push to registry (kaniko or Docker-in-Docker)
- [ ] `docker_tag` / `docker_push` — tag and push an existing image to a registry
- [ ] CI connector — trigger and monitor a GitHub Actions / GitLab CI pipeline, wait for result before next step
- [ ] Full build→push→deploy pipeline in one conversation: `git clone` → `docker_build` → `docker_push` → `helm upgrade`
- [ ] Registry connector — list images, inspect tags, detect untagged `latest` across a private registry

### Onboarding
- [ ] Step 3 GitHub: functional PR webhook setup (not just skip)
- [ ] Post-onboarding checklist: Prometheus, alert rules, first team
- [ ] UI wizard — pilot client provisioning in console (currently CLI only)

---

## P2 — Medium term

### Enterprise
- [ ] SSO / SAML integration (required for large enterprise procurement)
- [ ] LDAP/AD user provisioning
- [ ] Audit log export → SIEM (Splunk, Datadog, S3)
- [ ] Custom GPU alert thresholds per team
- [ ] Multi-org licence (enterprise with multiple BUs on one instance)

### Accelerator vendors
- [ ] Intel Habana Gaudi 3 (next-gen, different driver stack)
- [ ] Additional cloud TPU generations as they release
- [ ] Accelerator vendor SDK version matrix — tested compatibility table

### MCP Server
- [ ] MCP tool coverage parity with full agent tool set
- [ ] MCP server Helm chart for self-hosted deployment
- [ ] SDK: typed Python client generated from OpenAPI spec

---

## Out of scope for now

- **Managed service** — operating client infrastructure on their behalf. Requires dedicated ops team. Revisit at scale.
- **VibOps SaaS Cloud** — hosted VibOps as SaaS. Target markets (sovereignty-constrained CSPs and enterprises) require on-premise. Revisit for smaller clients without own infra.
- **Native charts** (without Grafana dependency) — revisit when monitoring tab proves insufficient.
