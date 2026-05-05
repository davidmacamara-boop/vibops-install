# VibOps — User Guide

## Table of contents

1. [Introduction](#1-introduction)
2. [Quick start — from zero to operational](#2-quick-start--from-zero-to-operational)
   - [Step 1 — Log in and change your password](#step-1--log-in-and-change-your-password)
   - [Step 2 — Connect your first cluster](#step-2--connect-your-first-cluster)
   - [Step 3 — Configure your secrets](#step-3--configure-your-secrets)
   - [Step 4 — Connect your Git provider](#step-4--connect-your-git-provider-for-gitops)
   - [Step 5 — Configure your organization (admin)](#step-5--configure-your-organization-if-you-are-an-admin)
   - [Step 6 — Set up a notification](#step-6--set-up-a-notification-optional-but-recommended)
   - [Step 7 — First test with the agent](#step-7--first-test-with-the-agent)
3. [Login and first session](#3-login-and-first-session)
4. [Console overview](#4-console-overview)
5. [The Morning Brief](#5-the-morning-brief)
6. [Talking to the agent](#6-talking-to-the-agent)
   - [How to phrase a request](#how-to-phrase-a-request)
   - [What the agent can do](#what-the-agent-can-do)
   - [Production safety guardrails](#production-safety-guardrails)
   - [Cross-session memory](#cross-session-memory)
7. [Kubernetes cluster management](#7-kubernetes-cluster-management)
8. [Inference workloads and GPU](#8-inference-workloads-and-gpu)
9. [MLOps workflows](#9-mlops-workflows)
10. [Incident diagnosis and remediation](#10-incident-diagnosis-and-remediation)
11. [Capacity planning and benchmarking](#11-capacity-planning-and-benchmarking)
12. [SLOs and alerts](#12-slos-and-alerts)
13. [GitOps](#13-gitops)
14. [Dashboard tab](#14-dashboard-tab)
15. [Cluster tab](#15-cluster-tab)
16. [Monitoring tab](#16-monitoring-tab)
17. [Admin tab](#17-admin-tab)
    - [Organization](#organization)
    - [Teams](#teams)
    - [Members](#members)
    - [Per-cluster roles (overrides)](#per-cluster-roles-overrides)
    - [Secrets](#secrets)
    - [Notifications](#notifications)
    - [Integrations](#integrations)
    - [Audit](#audit)
    - [Memories](#memories)
    - [License](#license)
18. [Connect Gateway](#18-connect-gateway)
19. [FinOps tab](#19-finops-tab)
    - [Waste — GPU waste detection](#waste-sub-tab--gpu-waste-detection)
    - [Budget](#budget-sub-tab)
    - [Chargeback](#chargeback-sub-tab)
    - [Alerts](#alerts-sub-tab)
20. [Dataset & RLHF](#20-dataset--rlhf)
21. [MCP Server](#21-mcp-server)
22. [Quick reference](#22-quick-reference)

---

## 1. Introduction

VibOps is the **AI agent for GPU/AI infrastructure operations**. It interfaces directly with your Kubernetes clusters, your NVIDIA stack, your Git repositories, and your monitoring tooling — responding to natural language instructions.

**What VibOps is not:**
- Not a general-purpose LLM — it knows GPU infrastructure and Kubernetes, not cooking or law
- Not a visualization tool — it is an agent that acts, not just displays
- Not an agent framework — it is a vertical solution, not a development kit

**What VibOps is:**
- A single interface for operating multi-environment Kubernetes clusters
- An on-call SRE capable of diagnosing, correlating, and remediating incidents in seconds
- A production guardrail: no destructive action without explicit confirmation
- An operational memory: the context of each cluster persists between sessions

---

## 2. Quick start — from zero to operational

This section is for you if you have just accessed VibOps for the first time and do not know where to begin. Follow these steps in order — allow **15 to 30 minutes** to be fully operational.

---

### Step 1 — Log in and change your password

1. Open the console URL provided by your administrator
2. Enter the username and temporary password communicated by your admin
3. Once logged in, click your **name in the top right** → **Change password**
4. Enter the temporary password in "Current password", then your new password (minimum 8 characters) twice
5. Click **Save**

> **Forgot or lost your password?** On the login page, click **Forgot password?**, enter your email or username, and follow the link received by email. In dev mode (without SMTP configured), the code appears directly on screen.

---

### Step 2 — Connect your first cluster

Without a connected cluster, the agent cannot do anything. This is the prerequisite for everything else.

**Option A — You have an existing Kubernetes cluster (most common)**

VibOps connects via a lightweight gateway deployed in your cluster:

1. Admin → **Integrations** sub-tab → **Connect Gateway** section
   — or directly: navigation bar → **Connect**
2. Click **Register gateway**
3. Give the gateway a name (e.g. `prod-gpu`, `eks-staging`) and an optional description
4. Click **Register** → a token is displayed. **Copy it now** — it will not be shown again.
5. On your cluster, deploy the VibOps worker with this token:
   ```bash
   # Copy the command shown in the interface, it looks like:
   docker run -d \
     -e VIBOPS_TOKEN=<your_token> \
     -e VIBOPS_CORE_URL=https://your-instance.vibops.ai \
     vibopsai/vibops-worker:latest
   ```
   — or via Helm if you are in production (see §17 Connect Gateway for details)
6. Wait a few seconds → the gateway switches to **online** status ✅ in the interface

> **The gateway does not require opening any inbound port** on your infrastructure. It establishes an outbound connection to VibOps.

**Option B — You want to test without a real cluster**

Use the built-in demo cluster (`vibops-dev` or `kind-vibops-dev`) if it is already shown in the Connect tab. These clusters are pre-configured for demonstration.

---

### Step 3 — Configure your secrets

Secrets allow the agent to act on your behalf on third-party services — without credentials ever appearing in the chat.

**Essential secrets to configure based on your use case:**

| You want to… | Secret to create | Value |
|--------------------|---------------|--------|
| Clone/push Git repos (GitHub) | `git_token` | GitHub Personal Access Token (scopes: `repo`, `workflow`) |
| Clone/push Git repos (GitLab) | `git_token` | GitLab Personal Access Token (scope: `read_api`) |
| Deploy NVIDIA NIMs | `ngc_api_key` | NGC API key (`ngc.nvidia.com` → Setup → API Key) |
| Access a private Docker registry | `registry_password` | Registry password or token |

**How to create a secret:**
1. Admin → **Secrets** sub-tab → **+ New secret**
2. **Name**: short identifier without spaces (e.g. `git_token`)
3. **Value**: paste the token/credential in plain text — it will be encrypted immediately
4. Click **Create**

Once created, use it in the chat with `@secret:name`:
```
Clone the repo github.com/myorg/infra with token=@secret:git_token
```

---

### Step 4 — Connect your Git provider (for GitOps)

If you want the agent to clone repos, push commits, and open Pull Requests:

**GitHub — Create a Personal Access Token:**
1. Go to [github.com](https://github.com) → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Give it a name (e.g. `vibops-prod`)
4. Check the scopes: **`repo`** (required) + **`workflow`** (if you use GitHub Actions)
5. Click **Generate token** → copy the value (`ghp_...`)

**GitLab — Create a Personal Access Token:**
1. Go to your GitLab instance → **User settings** → **Access tokens**
2. Give it a name (e.g. `vibops-prod`)
3. Check the scope: **`read_api`** (minimum) + **`write_repository`** (if you push commits)
4. Click **Create personal access token** → copy the value (`glpat-...`)

**Register the token in VibOps:**
1. Admin → **Secrets** → **+ New secret**
2. Name: `git_token` / Value: paste the token
3. Click **Create**

**Declare the integration:**
1. Admin → **Integrations** → click the **Git Provider** tile
2. Select your provider (`github` or `gitlab`)
3. Enter the token; for self-hosted GitLab, also fill in the **GitLab URL**
4. Click **Save**

**Test:**
```
Clone the repo github.com/myorg/my-repo
```
If the agent clones successfully → Git provider is configured ✅

---

### Step 5 — Configure your organization (if you are an admin)

This step is for `org_admin` users who want to invite their team and define access. If you are the only user on the instance, skip to step 6.

**Create a team:**

A team defines what its members are allowed to do (on which clusters, which namespaces, which environments).

1. Admin → **Teams** sub-tab → **New team**
2. Give it a name (e.g. `Platform`, `Dev`, `ReadOnly`)
3. Define the scope:
   - **Allowed namespaces** — e.g. `default, gpu-prod` (empty = all)
   - **Allowed environments** — e.g. `dev, staging` (empty = all)
   - **Allowed clusters** — e.g. `eks-prod` (empty = all)
4. Click **Create**

> **Getting started tip:** create at least two teams — one `Ops` with no restrictions and one `Dev` limited to `dev, staging`. Adjust from there.

**Invite a user:**

1. Admin → **Users** sub-tab → **+ New user**
2. Fill in:
   - **Username** — login identifier
   - **Email** — for password recovery
   - **Initial password** — communicate via a separate channel
   - **Role** — `readonly` / `developer` / `org_admin`
3. Click **Create**
4. Expand the user panel → assign them to a team

**Role summary:**

| Role | Can do |
|------|-----------|
| `readonly` | View clusters, logs, dashboard — no actions |
| `developer` | Deploy, scale, rollback — within their team's scope |
| `org_admin` | Everything — full organization management |

---

### Step 6 — Set up a notification (optional but recommended)

To receive VibOps alerts (detected incidents, budget exceeded, triggers fired) on Slack or by email:

**Slack (fastest):**
1. In Slack: target channel → **Integrations** → **Add an app** → **Incoming Webhooks**
2. Copy the generated URL (`https://hooks.slack.com/services/...`)
3. In VibOps: Admin → **Notifications** → **+ New channel** → type **Slack Incoming Webhook**
4. Paste the URL → click **Create**

**Email:**
1. Admin → **Notifications** → **+ New channel** → type **Email (SMTP)**
2. Fill in the SMTP fields (see §16 Notifications for Gmail/SMTP details)

---

### Step 7 — First test with the agent

Your cluster is connected and your secrets are in place. Verify everything works:

```
List the deployments on cluster <your-cluster-name>
```

The agent should respond with the list of active pods/deployments. If so: **you are operational** ✅

**Other useful verification commands:**
```
What is the health status of my infrastructure?
List the connected clusters and their gateways
Are there any GPUs available on my clusters?
```

---

### Summary — Getting started checklist

```
☐ Login OK + password changed
☐ At least one cluster connected (gateway online)
☐ git_token secret created (if GitOps)
☐ Git provider integration configured (if GitOps)
☐ ngc_api_key secret created (if NVIDIA NIM)
☐ Teams created and users invited (if admin)
☐ Notification channel created (Slack or email)
☐ First agent test successful (deployment list)
```

Once this checklist is complete, you can explore the rest of the guide according to your needs.

---

## 3. Login and first session

### Logging in

Open the console URL provided by your administrator. Enter your username and password. The session token is stored locally in the browser — you remain logged in until you click **Logout** or the token expires (24 hours by default).

### Changing your password

In the user menu (top right corner) → **Change password**. A temporary password is assigned to you when your account is created by your administrator — change it at first login.

### The Morning Brief

When opening the first session of the day, the agent automatically generates a **morning brief**: cluster health status, open incidents, anomalies detected since the previous day. It appears in the chat window without any action on your part.

See section [4 — The Morning Brief](#4-the-morning-brief) for details.

---

## 4. Console overview

```
┌─────────────────────────────────────────────────────────────────┐
│  [Dashboard] [Cluster] [Monitoring] [Git]  [⚙ Admin]  [FR/EN]  │
├──────────────────────┬──────────────────────────────────────────┤
│                      │                                          │
│   Sidebar            │   Main area                              │
│   (applications)     │   (active tab)                           │
│                      │                                          │
│                      ├──────────────────────────────────────────┤
│                      │   Chat panel (resizable)                 │
└──────────────────────┴──────────────────────────────────────────┘
```

### Navigation bar

- **Dashboard** — real-time view of the infrastructure (jobs, deployments, health)
- **Cluster** — CPU / RAM / GPU resources of the selected cluster
- **Monitoring** — live GPU metrics (temperature, utilization, VRAM, power)
- **Git** — GitOps status of the selected application
- **⚙ Admin** — administration panel (visible to `org_admin` only)
- **Language selector** — FR / EN / ES / DE / IT / PT / JA / ZH

A **red badge** on the ⚙ icon indicates the license expires in 7 days or less, or has expired.

### Sidebar

Lists all applications discovered on the cluster. Clicking an application selects it as the active context for chat and tabs.

### Cluster selector

At the top of the main area: dropdown menu to switch between the Kubernetes clusters you have access to. Changing the cluster automatically updates the sidebar, the Dashboard, and the Cluster tab.

### Chat panel

Always visible at the bottom of the screen. Drag the top edge to expand or collapse it. It retains the history of the current conversation.

---

## 5. The Morning Brief

At the first session of the day, the agent automatically generates a morning summary without you needing to ask a question. This brief includes:

- **Cluster health status** — pods in error, OOMKills, active CrashLoops
- **Recent jobs** — operations from the last 24 hours (success / failure / in progress)
- **Detected anomalies** — configuration drift, overloaded resources, received Prometheus alerts
- **Reminders** — in-progress tasks memorized from previous sessions (e.g. "staging to prod migration planned")
- **GPU metrics** — if a GPU cluster is connected: utilization, temperature, ECC errors

The brief is displayed directly in the chat window. You can ask follow-up questions immediately:

```
> Give me more details about last night's OOMKills
> Which pod has the most restarts?
> What is the status of the migration I started yesterday?
```

---

## 6. Talking to the agent

### How to phrase a request

The agent understands natural language instructions — no need to know kubectl syntax, exact resource names, or namespaces.

**Examples of valid phrasings:**

```
Give me the status of all our LLM inference services on vibops-dev
The open-webui pod is crashing. Diagnose what is happening.
Deploy nginx:latest on the apalacha cluster, namespace demo, 2 replicas
Scale llama3 in prod to 5 replicas and confirm they are all Running
Is there enough GPU capacity for a llama3-70b deploy with 4 GPUs?
What has changed on our clusters in the last 2 hours?
```

**Best practices:**

| Practice | Example |
|----------|---------|
| Specify the cluster if you have several | `"on vibops-dev"`, `"on the prod cluster"` |
| Specify the namespace for actions | `"in the ai namespace"`, `"in prod"` |
| Provide context for diagnostics | `"since this morning"`, `"since the last deployment"` |
| Confirm interactive actions | The agent asks for confirmation for destructive actions — reply "yes" or "cancel" |

**The agent always calls tools before responding** — it never guesses from its model memory. Every response reflects the actual state of the cluster at the time of the question.

### What the agent can do

| Category | Example actions |
|-----------|-------------------|
| **Discovery & audit** | List all clusters, map workloads, audit configurations |
| **Kubernetes** | Deploy, scale, restart, rollback, patch deployments |
| **GPU** | Check GPU availability, configure MIG, manage time-slicing |
| **Inference** | Deploy Ollama models, NVIDIA NIMs, manage endpoints |
| **Helm** | Install, update, rollback, diff charts |
| **ArgoCD** | Sync apps, view history, rollback |
| **Incident** | Diagnose, correlate logs + events + metrics, remediate |
| **Capacity** | Benchmark, plan capacity, detect underutilized GPUs |
| **GitOps** | Clone, patch YAML, commit, open PRs |
| **SLO & alerts** | Define SLOs, create triggers, configure Slack alerts |
| **Audit** | View the full history of all operations with who/what/when |

### Production safety guardrails

Certain actions are automatically blocked until explicit confirmation:

- Restarting a service in production
- Scaling a node group (cost and availability impact)
- Uninstalling a Helm chart
- Rolling back a deployment
- MIG reconfiguration (affects running pods)
- Any action in a `prod` namespace

When the agent proposes a blocked action, it displays:

```
⚠  This action will scale gpu-ng from 3 to 0 nodes.
   Impact: all GPU workloads will stop, cost suspended.
   Current state: 3 pods Running on gpu-ng.
   [Confirm]  [Cancel]
```

**These guardrails are enforced at the engine level** — not an interface convention, not a configuration flag. They cannot be disabled by the user.

### Cross-session memory

The agent memorizes operational context between sessions:
- Namespaces and clusters preferred by your team
- Models currently being experimented with
- Interrupted tasks (e.g. "migration planned tomorrow morning")
- Recurring anomalies you have already worked on

You can instruct the agent explicitly:
```
Remember that the ai namespace is reserved for research team experiments
Remember that llama3-70b is validated in staging until Friday
```

Memories can be viewed and deleted in **Admin → Memories**.

---

## 7. Kubernetes cluster management

### Multi-cluster discovery

```
Give me a complete view of our Kubernetes infrastructure — all clusters,
CPU/RAM/GPU resources per node, what is running in each namespace.
```

The agent automatically discovers all configured clusters, maps workloads in parallel, and surfaces operational risks (pods without limits, overloaded nodes, recent OOMKills).

### Creating a test cluster

```
Create a kind cluster "feature-test" with 2 nodes for the team, without touching vibops-dev.
```

The cluster is provisioned, kubeconfig configured automatically, and it appears in the console's cluster selector. To delete it:

```
Delete the feature-test cluster.
```

### Deploying a workload from scratch

```
Deploy nginx:latest on the apalacha cluster, namespace demo, 2 replicas, port 80.
```

The agent creates the namespace if needed, the Deployment, the Service, and confirms the pods are Running before responding.

### Port-forward to localhost

```
Port-forward nginx from the demo namespace on apalacha to localhost:8080.
```

### Patching resources

```
The open-webui pod is OOMKilling. Increase its memory limit to 2Gi on vibops-dev.
```

The agent checks the current state, applies the strategic merge patch, and confirms the rollout.

### Detecting configuration drift

```
Compare llama3 deployments in staging and prod on vibops-dev.
Are they running on the same image and the same config?
```

The agent queries both environments in parallel and produces a full diff: image, replicas, pull policy, resource limits — with an explanation of what each discrepancy means operationally.

### Auditing production workloads

```
Audit all our LLM workloads on vibops-dev.
Surface anything running without resource limits, with an untagged image,
or with fewer replicas than expected in prod.
```

### Operations history

```
Show me everything that happened on the clusters in the last 2 hours.
Any failures to flag?
```

---

## 8. Inference workloads and GPU

### Checking GPU availability

```
We need 4 A100s for a llama3-70b deployment.
Do we have the capacity on the prod cluster?
```

The agent inspects each GPU node in parallel, calculates free vs allocated GPUs, and delivers a verdict with the list of what is currently consuming GPUs.

### Deploying an Ollama model

```
Deploy an Ollama container for mistral in the ai namespace on vibops-dev,
1 replica for the research team.
```

### Deploying a NVIDIA NIM

```
Deploy the NVIDIA NIM for llama-3.1-8b-instruct on the prod GPU cluster with 2 GPUs.
```

The agent performs in order: GPU audit, operator audit, VRAM profile selection, Helm deployment with the NGC secret, waiting for pod availability, and a live inference test.

### Bootstrapping the GPU Operator

```
I just added a GPU node on vibops-dev.
Configure the NVIDIA GPU Operator so the node is ready for inference workloads.
```

The agent installs the GPU Operator via Helm, waits for driver + device-plugin + DCGM exporter to all be Running, and confirms the GPU capacity visible by the scheduler.

### Configuring MIG (multi-instance GPU)

```
Partition the A100s on gpu-node-1 into 3g.40gb slices so
three teams can share the GPU.
```

**Note:** MIG reconfiguration requires explicit confirmation — the agent explains the impact (node drain, affected pods) before acting.

### Configuring GPU time-slicing

```
Configure time-slicing on underutilized GPUs to multiply
workloads on the same hardware.
```

### GPU cost report

```
How much are our GPUs costing this week? Break it down by team
and surface anything that looks like waste.
```

### Detecting idle GPUs

```
Are any of our GPU workloads underutilizing their allocation?
I want to know what we can scale down or move to time-slicing.
```

---

## 9. MLOps workflows

### Inference fleet status (start of day)

```
Give me the complete status of all our LLM inference services on vibops-dev —
models, namespaces, replicas, health.
```

### Promoting a model from staging to prod

Step 1 — verify staging is ready:
```
Is llama3 in staging on vibops-dev healthy and running on the right image?
I want to promote it to prod.
```

The agent compares staging and prod in parallel and confirms whether the promotion is safe.

Step 2 — promote:
```
OK. Move prod llama3 to the same image as staging and confirm the rollout.
```

### Updating a model with validation

```
Update llama3 in prod on vibops-dev to ollama/ollama:0.6.5,
then check the logs to confirm the model loaded correctly.
```

The agent patches, waits for the rollout, reads the logs of the new pod, and confirms the model is operational — or reports a startup error.

### Emergency rollback

```
The last deployment broke the llama3 API in prod on vibops-dev.
Immediate rollback to the previous version.
```

The agent checks the deployment state, executes the rollback, and confirms the pod is Running again on the previous version.

### Scale-up for a traffic spike

```
We expect a traffic spike tonight on prod llama3 on vibops-dev.
Scale to 5 replicas and confirm they are all Running.
```

### Scale-down after the traffic spike

```
Traffic has dropped. Scale llama3 in prod to 2 replicas.
```

---

## 10. Incident diagnosis and remediation

### Initial diagnosis

```
We have latency spikes in the ai namespace since this morning.
Diagnose what is happening, check the cluster resources,
and tell me if open-webui is healthy.
```

The agent automatically correlates: recent logs, Kubernetes events, pod status, resource utilization, restart history. It surfaces the root cause and a concrete recommendation.

### Full remediation in two steps

Step 1:
```
The open-webui pod in the ai namespace on vibops-dev is crash-looping.
Diagnose what is wrong and tell me what to do.
```

The agent uses `analyze_pod_failure` to correlate events + describe + logs and returns a structured diagnosis (cause, current limit, last log lines, recommendation).

Step 2:
```
Fix it. Increase the memory limit to 2Gi.
```

The agent applies the patch and confirms the pod is Running with the restart count reset to zero.

### Multi-source correlation

For complex incidents involving multiple services:

```
Correlate the incident on the inference namespace: logs + Kubernetes events +
Prometheus metrics + deployment status. I want the full timeline.
```

The agent queries all four sources in parallel (`asyncio.gather`) and produces a correlated timeline with identified causal relationships.

### Prometheus alerts

When a Prometheus alert arrives via webhook, the agent receives it automatically, launches the diagnosis pipeline, and posts an analysis in the active chat (or opens a new conversation) without any action on your part.

### Auto-healing: creating a trigger

```
Set up a rule: if a pod has more than 3 restarts in 10 minutes,
automatically restart the deployment and notify on Slack.
```

For destructive actions (rollback, scale-down), the trigger will pause and ask for confirmation — the same guardrail as for manual actions.

---

## 11. Capacity planning and benchmarking

### Benchmarking an inference service

```
Benchmark the ollama service in the ai namespace on vibops-dev.
Run 5 requests against llama3 and give me p50/p95/p99 latencies.
```

The benchmark runs inside the pod (not from your workstation), giving real latencies as your users experience them.

### Planning capacity

```
Based on this benchmark (p99: 3200ms, avg: 2500ms), plan capacity for llama3
to handle 50 RPS at under 2000ms p99. Each replica needs 4 CPU and 8GB RAM.
```

The agent calculates the recommended number of replicas (with 20% headroom), total resources needed, and an HPA min/max range.

Then directly:
```
Scale llama3 in the ai namespace to the recommended number of replicas.
```

### Finding the right cluster for a new deployment

```
I need to deploy a new inference service: 4 CPU, 8GB RAM, 2 replicas.
Which of my clusters has the capacity?
```

The agent scans all clusters in parallel and returns a ranked list with fit score (available headroom).

### Detecting GPU waste

```
Are any of our GPU workloads underutilizing their allocation?
I want to know what we can optimize.
```

---

## 12. SLOs and alerts

### Defining an SLO

```
Define an SLO for llama3 in the prod namespace:
p99 latency below 2000ms, 99.9% of the time.
Alert on Slack if we exit the error budget.
```

The agent creates the SLO, configures a Prometheus trigger, and wires up the Slack notification.

### Checking SLO compliance

```
Are we currently meeting the llama3 SLO?
```

The agent queries Prometheus and returns real-time compliance: current p99 vs threshold, remaining error budget.

### Configuring Prometheus alerts

```
Configure a Slack alert if a node's CPU exceeds 90% for more than 5 minutes.
```

### Checking available notification channels

```
What notification channels are configured?
```

---

## 13. GitOps

### Modifying a configuration in a repository

```
Increase the replicas of inference-server to 3 in the infra repo
and open a PR for review.
```

The agent clones the repository (with the token configured in Secrets), patches the YAML, generates the diff, commits, and opens the PR — all in a single operation.

### Viewing the GitOps status of an application

**Git** tab → select the application in the sidebar.

Displays: current branch, last commit, diff vs main, open PRs associated with VibOps commits.

### Using a Git token from Secrets

```
Clone the infra repo using token=@secret:git_token
```

Secrets are injected at runtime — the raw value is never exposed in the chat.

---

## 14. Dashboard tab

Real-time view of operational activity:

- **Recent jobs** — all operations with status (in progress / success / failure), environment badge (dev/staging/prod), and timestamp. Click a job to see its full logs.
- **Counters** — number of jobs in progress / succeeded / failed for the period.
- **Deployments** — list of all discovered applications with replica count and health status.
- **⟳ Discover button** — triggers a full cluster scan.
- **Last discovered** — timestamp of the last automatic scan.

The Dashboard refreshes automatically every 30 seconds.

---

## 15. Cluster tab

Live resource view of the selected cluster:

- **GPU** — total / used / free with utilization rate bar
- **CPU** — requested vs allocatable across all nodes
- **Memory** — same
- **Nodes** — status per node, CPU/RAM/GPU capacity and requests
- **Pods** — top consumers by resource request

Use the **context selector** (top left) to switch between clusters. **Refresh** button to relaunch the scan manually.

---

## 16. Monitoring tab

Real-time GPU metrics collected by DCGM Exporter:

- Temperature per GPU (°C)
- GPU utilization rate (%)
- VRAM memory used / free (MiB)
- Power consumed (W)
- ECC error counters

**Prerequisites:** DCGM Exporter must be deployed on the cluster and Prometheus configured. If not, the tab displays installation instructions.

To detect and install automatically:
```
Is Prometheus installed on my cluster? If not, install it.
```

---

## 17. Admin tab

**Who can access it?** Only users with the `org_admin` role. If you do not see the ⚙ icon in the sidebar, your account does not have this role — ask an admin to assign it to you.

**How to access it?** Click the ⚙ icon at the bottom of the left sidebar. A panel opens over the console. Click **Close Admin** or press **Escape** to close it.

> **Red badge on the ⚙ icon** — Indicates a license issue to resolve: quota exceeded, license expired, or trial expired. Click ⚙ then the **License** tab to see the details.

The Admin is organized into sub-tabs. Here is each one in turn.

---

### Organization

This is your organization's identity profile in VibOps.

**What you can view and modify:**
- **Name** of the organization (displayed in the header and reports)
- **Slug** (short identifier, used in URLs — for example `acme-corp`)
- **Logo URL** — if you have a hosted logo, enter its URL here. The URL is auto-detected from your org email domain name, but you can replace it.
- **Dataset consent** — controls whether your org's operational data contributes to VibOps improvement (see §19 Dataset & RLHF)

**How to modify:**
1. Click in the field to modify
2. Type the new value
3. Click **Save**

> **The slug matters**: if you change it after creating rules or pipelines that use it, you will need to update them manually.

---

### Teams

A **team** is a group of users with a shared action scope. It defines: on which clusters, which namespaces, and which environments members can act.

**Why teams exist:** Imagine you have a junior developer. You want them to be able to deploy in `dev` but absolutely not touch `prod`. You create a `Dev team`, give it access to the `dev, staging` namespaces and `dev` environment only. The junior developer is a member. They will never be able to send an action to `prod`, even if they try.

**Team parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| **Name** | Internal team name | `Platform Team` |
| **Allowed namespaces** | List of accessible K8s namespaces (comma-separated). Empty = all. | `default, prod, gpu-workloads` |
| **Allowed environments** | `dev`, `staging`, `prod` — comma-separated. Empty = all. | `dev, staging` |
| **Allowed clusters** | Accessible cluster names. Empty = all. | `kind-dev, eks-staging` |
| **GPU quota** | Max number of GPUs the team can request per job. `0` = unlimited. | `8` |

**Create a team:**
1. Admin → **Teams** sub-tab
2. Click **New team**
3. Fill in the fields. **Important**: if you leave a field empty, there is no restriction for that dimension (e.g. empty namespaces = access to all namespaces).
4. Click **Create**

**Modify a team:**
1. Click the team name in the list
2. Modify the fields
3. Click **Update**

**Delete a team:**
- Click the trash icon next to the team
- Members of this team lose the permissions it gave them — verify they have other teams or roles before deleting

> **Tip:** Start by creating 2-3 teams (e.g. `Ops`, `Dev`, `ReadOnly`) and adjust as you go rather than creating one team per person.

---

### Members (Users)

This is where you manage user accounts: creation, teams, role, password, activation/deactivation.

**Understanding roles:**

| Role | Can do | Cannot do |
|------|-----------|------------------|
| `readonly` | View clusters, logs, dashboard, read-only chat | Launch jobs, modify resources, access Admin |
| `developer` | Everything `readonly` + deploy, scale, rollback, launch jobs | Manage users, secrets, licenses. Prod actions subject to team restrictions. |
| `org_admin` | Everything — manage teams, members, secrets, notifications, license | Nothing — full access |

> **Important:** The role is global to the organization. Fine-grained scope (which clusters, which namespaces) is controlled by teams. A `developer` without an assigned team can act on everything.

**Create a user:**
1. Admin → **Users** sub-tab
2. Click **+ New user**
3. Fill in:
   - **Username** — login identifier (no spaces, no uppercase)
   - **Email** — for password recovery (optional if no SMTP)
   - **Initial password** — communicate to the user via a separate channel
   - **Role** — choose `readonly`, `developer`, or `org_admin`
   - **Admin?** — check if `org_admin`
4. Click **Create**

> **After creation**, the user will need to change their password. You can generate a reset link from the login page ("Forgot password" section).

**Modify a user:**
- Click on the user's row to expand their detail panel
- Modify the role, teams, email
- Changes take effect immediately

**Deactivate an account:**
- Expand the user panel → **Deactivate** button
- The user can no longer log in but their history is preserved
- You can reactivate them later

**Assign a team:**
- In the user's detail panel, select teams from the list
- A user can belong to multiple teams — the broadest permissions apply

---

### Per-cluster roles (overrides)

**Why this feature exists:** Sometimes a user needs a different access level depending on the cluster, without you wanting to create an entire team just for that. Example: Alice is a `developer` in the general team, but on the `prod-gpu` cluster, you want her to be `readonly` — she can view but not modify.

**How it works — priority rule:**

```
Effective access on a cluster =
  cluster override defined for this user → that role applies
  otherwise → user's team role
  otherwise → user's global role
```

In other words: the cluster override ALWAYS takes precedence over the team role, for that specific cluster only.

**How to add an override:**
1. Admin → **Users** sub-tab
2. Find the user in the list
3. Click the **⛭ Cluster roles** button on their row (the numeric badge shows the number of active overrides)
4. The panel opens below the user card
5. In the **Cluster name** field: type the exact name of the cluster, as it appears in the Connect tab (e.g. `kind-vibops-dev`, `eks-prod-us-east-1`)
6. In the **Role** menu: choose `readonly`, `developer`, or `admin`
7. Click **+ Add override**

> **The cluster name must be exact.** If you type `prod` but the cluster is named `eks-prod`, the override will not apply. Check the exact name in the **Connect → Gateways** tab.

**How to remove an override:**
- In the user's Cluster roles panel, click the trash icon next to the override
- You will be asked to confirm
- The override is removed immediately — the user reverts to their team role on that cluster

**Concrete examples:**

| Situation | Solution |
|-----------|---------|
| Alice is dev everywhere, but readonly on prod | Override `prod` → `readonly` for Alice |
| Bob is readonly everywhere, but needs to act on the test cluster | Override `kind-test` → `developer` for Bob |
| Carol manages only the H100 cluster, not the others | Global role `readonly` + override `gpu-h100` → `admin` |
| The Platform team should be admin on all clusters except the client cluster | `developer` role in the team + override `client-cluster` → `readonly` |

> **Technical note:** Overrides are read on every API request. Deletion is effective immediately for API tokens. For users connected via JWT (browser session), the new role applies at the next token issuance (within 2 hours at most). If you need the effect to be immediate: ask the user to log out and log back in.

---

### Secrets

The VibOps secrets vault stores your sensitive credentials (API tokens, passwords, private keys) in encrypted form, and injects them into jobs on demand — without ever exposing them in plain text in the chat or logs.

**Why not just type the token in the chat?** If you type `ngc_api_key=abc123` in the chat, that value appears in the history, logs, and potentially exports. Vault secrets are never logged.

**Create a secret:**
1. Admin → **Secrets** sub-tab
2. Click **+ New secret**
3. Fill in:
   - **Name** — short identifier without spaces (e.g. `git_token`, `ngc_api_key`)
   - **Value** — the secret in plain text. Displayed masked after entry.
   - **Description** — optional, to remind you what this secret is for
4. Click **Create**

> Once created, **the value is no longer displayed** in the interface. If you have lost it, delete and recreate the secret.

**Use a secret in the chat:**
```
Clone the infra repo with token=@secret:git_token
Deploy the llama3 NIM with NGC key=@secret:ngc_api_key
Push the config with credentials=@secret:registry_password
```
The agent automatically resolves `@secret:<name>` before sending the request to the cluster. The token is never visible in the response or in the logs.

**Modify / delete a secret:**
- Click the pencil ✏ to modify the value
- Click the trash 🗑 to delete (irreversible — jobs that reference this secret will fail afterwards)

> **System secrets**: some secrets are marked `system` — they are shared between organizations by the VibOps team for reselling scenarios. Do not modify them unless explicitly instructed.

---

### Notifications

Configure where VibOps sends its alerts: detected incidents, fired triggers, completed deployments, license expiration.

**Available channels:**

| Channel | What to configure | When to use |
|-------|------------------------|-----------------|
| **Slack Incoming Webhook** | Slack webhook URL (e.g. `https://hooks.slack.com/...`) | Alerts in a Slack channel |
| **HTTP Webhook** | URL + optionally a secret for signing requests | Custom integration (PagerDuty, Opsgenie, etc.) |
| **Email (SMTP)** | SMTP server, port, user, password, recipients | Email alerts |
| **PagerDuty** | PagerDuty Integration Key | Incident escalation |

**Create a notification channel:**
1. Admin → **Notifications** sub-tab
2. Click **+ New channel**
3. Choose the type from the dropdown
4. Fill in the fields according to the type (see table below)
5. Click **Create**

**Slack configuration (simplest):**
1. In Slack: go to your workspace → Apps → Incoming Webhooks → Add to Slack
2. Choose the destination channel → click "Add Incoming Webhooks integration"
3. Copy the generated URL (e.g. `https://hooks.slack.com/services/T.../B.../xxx`)
4. In VibOps: paste this URL in the **Webhook URL** field
5. Give the channel a name (e.g. `Slack #alerts-gpu`)

**Email SMTP configuration:**

| Field | Description | Example |
|-------|-------------|---------|
| SMTP server | Mail server hostname | `smtp.gmail.com` |
| SMTP port | Port (587 = TLS, 465 = SSL, 25 = unencrypted) | `587` |
| Username | Email or SMTP login | `alerts@mycompany.com` |
| Password | SMTP password or App Password | `xxxx xxxx xxxx xxxx` |
| Recipients | Addresses separated by comma | `ops@mycompany.com, cto@...` |

> **Gmail**: use an "App Password" (Google account → Security → App passwords), not your regular Google password.

**Test a channel:**
After creation, manually trigger a trigger or a Morning Brief to verify the notification arrives.

**Delete a channel:**
Click the trash icon next to the channel. Triggers that use it will no longer be notified.

---

### Integrations

Connect VibOps to your external tools so the agent can use them automatically.

**Available integrations:**

| Integration | What it does | What to provide |
|-------------|---------------|------------------------|
| **Git Provider** (GitHub or GitLab) | Clone repos, push configs, create PRs/MRs (GitOps workflow) | `GIT_PROVIDER` + Personal Access Token (`repo`, `workflow` for GitHub — `read_api`, `write_repository` for GitLab) + `GIT_URL` for self-hosted GitLab |
| **Prometheus** | Real-time metrics, SLOs, incident correlation | URL (e.g. `http://prometheus:9090`) + credentials if auth is enabled |
| **Grafana** | Receive Grafana/AlertManager alerts in VibOps | The webhook URL to configure in Grafana — VibOps side is automatic |
| **NGC (NVIDIA)** | Deploy NIMs from the NVIDIA catalog | NGC API key (obtained at `ngc.nvidia.com`) |
| **ArgoCD** | GitOps synchronization, application status | ArgoCD URL + admin token |

**How to configure an integration:**
1. Admin → **Integrations** sub-tab
2. Click on the tile of the desired integration
3. Fill in the fields (they vary by integration)
4. Click **Save**

> **Integrations are stored in the secrets vault**. You will never see the token in plain text after the first entry.

**Verify an integration is working:**
- **Git Provider**: ask the agent to clone a repo: `"Clone the repo github.com/myorg/infra"` (or your GitLab URL). If it works, the token is valid.
- **Prometheus**: the Monitoring → Fleet tab should display metrics. If you see "Prometheus not detected", the URL is incorrect or unreachable.
- **NGC**: ask the agent: `"List the available models in the NGC catalog"`.

---

### Audit

The audit log is the **complete and immutable** history of everything that happened in VibOps: every job launched, every action denied, every configuration change.

**What it is for:**
- Find out who did what and when (essential after an incident)
- Verify that an action was successfully executed (or understand why it failed)
- SOC2 compliance / security audit

**Log columns:**

| Column | Description |
|---------|-------------|
| **Date** | Precise timestamp of the action |
| **User** | Who triggered the action (`claude-agent` = the AI, `github-webhook` = a GitHub trigger, or your username) |
| **Action** | Technical name of the action (e.g. `scale_cluster`, `deploy_model`) |
| **Parameters** | What was passed to the action (cluster, namespace, number of replicas…) |
| **Status** | `success` (green), `failed` (red), `pending` (grey) |
| **Duration** | Execution time |

**Filter logs:**
- **By action**: type `scale_cluster` to see only scaling actions
- **By status**: filter on `failed` to see only errors
- **By user**: type `claude-agent` to see AI actions, or your username for your own actions

**Typical use cases:**

_"A deployment crashed last night, I want to know what happened"_
→ Filter by status `failed`, look at the **Error detail** column on the corresponding row

_"Someone scaled the prod cluster to 0 replicas, who was it?"_
→ Filter by action `scale_cluster`, look at the **User** column

_"The agent took an action I didn't request"_
→ Filter by user `claude-agent`, check the **Parameters** column to see exactly what it did

> **Pagination**: the `←` and `→` buttons load in batches of 100 entries. The most recent appear first.

---

### Memories

The VibOps agent has **persistent memory** between sessions. Without it, it would start from scratch with each conversation — it would not know that it already diagnosed an incident on prod last week, or that you always want confirmation before scaling.

**Why is the panel empty at first?**
Memories are created over the course of real conversations. On a fresh instance, without chat history, the panel is empty — this is normal. They accumulate naturally as you use the agent.

**How it works in practice:**
1. You ask the agent a question or request an action
2. The agent acts, then at the end of the turn it decides whether something deserves to be memorized
3. If so, it calls `save_memory` — an entry appears in this panel
4. **At the next session**, before even reading your first message, the agent re-reads all saved memories and integrates them into its context — it already "knows" what it learned before

**The 6 memory types and when they are created:**

| Type | Created automatically when… | Example |
|------|-----------------------------|---------|
| `incident` | The agent diagnoses a failure with an identified cause | OOMKill on llama3, fixed on 2026-05-03 |
| `slo` | An SLO is created or modified via the agent | llama3: p99 < 500ms at 99.5% |
| `pipeline` | A multi-step pipeline is interrupted mid-way | deploy-to-prod blocked at step 3 |
| `preference` | You express an explicit preference | "Always ask for confirmation before scaling prod" |
| `fact` | The agent learns a structural fact about your infra | "vibops-dev is a kind cluster without real GPUs" |
| `action` | A recurring operation is defined | "Every Monday, scale down the dev cluster" |

The `incident`, `slo`, and `pipeline` types are saved **automatically**. The `preference`, `fact`, and `action` types generally require you to ask explicitly.

**Force a save manually:**
```
Remember that the prod namespace is frozen until the 25th of the month.
Remember that vibops-dev is our test cluster without GPUs.
Remember that I always want a dry-run before any destructive action.
```

**Agent behavior rules:**
- Maximum 2 memories saved per conversation turn
- Never a duplicate on an existing key (it updates instead)
- Never during an action sequence — only after the final result

**What the agent CANNOT do:**
The agent only has access to memories. It cannot modify users, teams, secrets, notifications, budget, license, or anything else in the Admin. All of that is reserved for `org_admin` users via the interface or API.

**Managing memories from the panel:**
1. Admin → **Memories** sub-tab
2. Filter by type (`fact`, `incident`, `preference`…) via the dropdown
3. **Delete**: click the trash icon — the agent will no longer remember it from the next session
4. **Refresh**: click ↺ to reload the list

**When to delete a memory:**
- The information has become false or outdated (e.g. the namespace freeze is over)
- The agent is drawing incorrect conclusions because of a stale memory
- You want to start fresh on a specific topic

> **Memories are per organization.** All `org_admin` users can view and delete them. Non-admin users do not see this panel.

---

### License

The License tab displays the status of your VibOps subscription and the associated quotas.

**What you see:**

| Information | Description |
|-------------|-------------|
| **Plan** | Trial (14 days) / Starter / Growth / Scale |
| **GPUs under management** | Number of GPUs currently declared vs maximum allowed by your plan |
| **Users** | Active accounts vs maximum |
| **Clusters** | Connected clusters vs maximum |
| **Expiration** | Expiration date or number of days remaining |

**Visual indicators:**

The **trial banner** at the top of the console (visible to admins only) changes color based on urgency:
- ⬜ **Grey**: more than 7 days remaining — nothing to do
- 🟡 **Amber**: 7 days or less — consider renewing
- 🔴 **Red**: expired or quota exceeded — some functions are blocked

**What happens when the license expires?**
- In-progress jobs complete normally
- New jobs are blocked
- The console switches to read-only mode
- The agent can answer questions but can no longer act

**What happens when a quota is exceeded?**
- GPU quota exceeded → new GPU jobs are rejected (existing ones continue)
- User quota exceeded → you can no longer create new accounts
- Cluster quota exceeded → you can no longer connect new clusters

To renew or upgrade to a higher plan: contact **`david@vibops.ai`** or your VibOps sales representative.

---

## 18. Connect Gateway

The Connect Gateway allows connecting remote clusters (on-premise, multi-cloud, isolated environments) to your central VibOps instance, via a secure outbound connection — without opening any inbound port on your infrastructure.

### Architecture

```
VibOps Core (your datacenter)
        ↑  Encrypted WebSocket (TLS)
        │
vibops-worker (in the target cluster)
        │
        ↓  kubectl / Helm / GPU Operator
Remote GPU cluster
```

The worker receives only the necessary instructions, sends back results, and pushes metrics (heartbeat every 60 seconds).

### Connecting a cluster via the console

1. Admin → **Gateways** tab → **New Gateway**
2. Enter the cluster name and URL
3. Copy the generated Helm command
4. Execute it on the target cluster

### Connecting a cluster via the script

```bash
./connect-setup.sh --name prod-gpu-1 --cluster kind-vibops-dev --start
```

Options:
- `--name` — gateway name in the console
- `--cluster` — target Kubernetes context name
- `--start` — starts the worker immediately after installation

### Connecting a cluster via Helm (production)

```bash
helm install vibops-worker charts/vibops-connect \
  --namespace vibops-connect \
  --create-namespace \
  --set worker.core.url=https://vibops.yourcompany.com \
  --set worker.core.token=<gateway-token>
```

### Gateway status

In Admin → Gateways, each gateway displays:
- **Status** — connected / disconnected / degraded
- **Last heartbeat** — X seconds ago
- **Clusters** — number of clusters exposed
- **GPUs** — total GPUs declared by the worker

A gateway disconnected for more than 5 minutes switches to `degraded` state and generates an alert.

---

## 19. FinOps tab

**What is it for?** The FinOps tab gives you a centralized view of what your GPUs cost, where the money goes, and how to control it. Four sub-tabs: **Waste**, **Budget**, **Chargeback**, **Alerts** (history of overruns).

**Who should use it:** admins, infrastructure leads, and anyone who needs to justify GPU costs to a finance director.

---

### Waste sub-tab — GPU waste detection

**What is GPU waste?** A GPU that is running but doing nothing (model loaded in memory but zero requests, pod waiting, job finished but worker not released) consumes energy and money. The Waste sub-tab shows you exactly that.

**How to read the Waste table:**

| Column | Description |
|---------|-------------|
| **Cluster** | Cluster where the idle GPU is detected |
| **GPU / Device** | GPU identifier (e.g. `nvidia.com/gpu` on node `gpu-node-1`) |
| **Utilization %** | Average utilization percentage measured at the last scan |
| **Memory used** | GPU memory occupied (if 0 MB → pod exists but uses nothing) |
| **Score** | Waste score from 0 to 100 — the higher, the more urgent |
| **Est. waste/month** | Estimated waste cost in USD for the full month |
| **Scanned** | How long ago the last measurement was taken |

**Item severity:**
- 🔴 **High** — utilization < 5% or memory occupied but no measured activity
- 🟡 **Medium** — utilization between 5% and 20%
- 🟢 **Low** — utilization between 20% and the configured threshold

**Available actions on each row:**

- **Click the row** → expands the detail by device (GPU ID, node, precise metrics)
- **Diagnose button** → automatically pre-fills the chat with scan data and starts an analysis: the agent tells you why this GPU is idle and what to do. No copy-pasting required.
- **Scale down button** → opens the scaling form directly in the chat to reduce replicas of the offending workload

**Triggering a new scan:**

The Waste scan is not continuous (it requires cluster resources). To launch a fresh analysis:
```
Detect idle GPUs on the prod cluster
```
or via the agent:
```
Analyze GPU waste across all clusters
```

> **The scan takes a few seconds.** After the agent responds, go back to the Waste tab and click ↺ to reload. Fresh data appears with a timestamp of "Scanned < 1 minute ago".

**Typical case: idle GPU overnight**
```
It is 8am, the scan shows 4 GPUs idle since the night:
→ Click Diagnose on the row → the agent confirms they are last night's test pods
→ Click Scale down → scale to 0
→ Budget saved: ~$200/night depending on GPU type
```

---

### Budget sub-tab

**What is a VibOps budget?** It is a monthly GPU spending cap for your organization. VibOps monitors accumulated consumption in real time (calculated from jobs executed this month) and alerts or blocks you when you approach or exceed the limit.

**Understanding the two types of cap:**

| Type | Behavior | When to use |
|------|-------------|-----------------|
| **Alert threshold (soft cap)** | Sends an alert (Slack, email…) but blocks nothing | Notify the team at 80% consumption |
| **Blocking cap (hard cap)** | Blocks creation of new GPU jobs when reached | Never exceed an absolute contractual or budget limit |

> **Tip:** Set the soft cap at 80% and the hard cap at 100% (or 110% if you want a margin). Only activate the hard cap if you have a real financial constraint — it can block production workloads.

**What you see on the Budget tab:**

- **Cap ($)** — configured monthly budget
- **Spent ($)** — current month consumption (calculated from jobs)
- **Burn rate / day** — current spending speed (last 7 days spend / 7)
- **End-of-month forecast** — at this rate, how much you will spend by the 31st. In red if it exceeds the cap.
- **Progress bar** — visual fill with soft cap and hard cap markers

**Create or modify a budget:**
1. FinOps → **Budget** sub-tab
2. If no budget is configured: click **Set budget**
3. If a budget already exists: click **Edit**
4. Fill in:
   - **Limit (USD)** — monthly cap in dollars (e.g. `5000`)
   - **Alert threshold (%)** — percentage at which to send an alert (e.g. `80`)
   - **Hard cap (%)** — percentage at which to block new jobs (e.g. `100`). Leave at 100 by default.
5. Click **Save**

> The budget is calculated on the calendar month (1st to last day of the month). It resets to zero on the 1st of each month.

**Or via the chat:**
```
Set the monthly GPU budget to 5000 dollars with an alert at 80% and a hard cap at 100%
```

**Delete the budget:**
- **Delete** button in the Budget form
- Without a configured budget, no blocking is applied (unlimited spending)

**Budget alerts:** when the soft cap is reached, a notification is sent to the channels configured in Admin → Notifications. Verify you have at least one channel configured to avoid missing the alert.

---

### Chargeback sub-tab

**What is chargeback?** It is the monthly report that details: how much did the GPU infrastructure cost this month, broken down by team, by cluster, and by workload type. Useful for charging GPU costs back to business units or for justifying spending internally.

**How to generate a report:**
1. FinOps → **Chargeback** sub-tab
2. Select the **year** and **month** from the selectors at the top
3. Click **Load** — if a report exists for this period, it is displayed
4. If no report exists → click **Generate** (or **Generate from jobs** to build the report directly from VibOps jobs)
5. The report appears within a few seconds

**Reading the report:**

| Section | Description |
|---------|-------------|
| **Total cost** | Total billed cost (customer price with markup if applicable) |
| **Internal cost** | Actual infrastructure cost (without markup) |
| **GPU hours** | Total GPU hours consumed |
| **Total jobs** | Number of jobs that contributed to the cost |
| **Markup applied** | If you are a reseller, the configured markup |
| **Breakdown by team** | Cost breakdown by team (K8s namespace) |
| **Vendor breakdown** | Distribution by GPU vendor (NVIDIA / AMD / Google TPU / etc.) |

**Export to CSV:**
- **Export CSV** button at the top of the report
- The downloaded file contains all detailed rows: date, action, cluster, namespace, unit cost, total cost

**12-month history:**
- The chart at the bottom of the tab displays your GPU spending evolution over the last 12 months
- Useful for spotting abnormal months and anticipating future budgets

**When "Generate" vs "Generate from jobs"?**
- **Generate** — creates the standard report from recorded cost data. Use this normally.
- **Generate from jobs** — rebuilds the report by re-reading VibOps jobs directly. Use this if the standard report is empty while jobs ran this month.

> A generated report is frozen at the time of generation. If new jobs arrive afterwards, you can regenerate — the report will be overwritten with up-to-date data.

---

### Alerts sub-tab

The history of all budget overruns: when the soft cap or hard cap was reached, what the spend was at that moment, and what the cap was.

**Columns:**

| Column | Description |
|---------|-------------|
| **Type** | `Soft alert` (alert threshold) or `Hard cap` (blocking cap) |
| **Threshold** | The percentage that triggered the alert (e.g. 80%) |
| **Spend at trigger** | Amount spent at the time of triggering |
| **Limit** | Cap configured at the time of triggering |
| **Date** | Date and time of triggering |

> Alerts are listed in reverse chronological order (most recent first). If the list is empty: either no budget is configured, or you have never exceeded the threshold — that is a good thing!

---

## 20. Dataset & RLHF

VibOps builds an operational dataset from real production actions — the foundation for fine-tuning specialized GPU models.

### Collected signals

| Signal | Source |
|--------|--------|
| WorkloadSignature (vendor, accelerator, framework) | Job submission |
| Job results (success/oom/timeout/failure) | Worker end-of-execution |
| Recommendation events (followed/ignored/overridden) | Operator response |
| Automatic framework detection from image | WorkloadDetector |
| Agent feedback (👍 / 👎 per response) | Chat interface |

### Consent model

Configurable per organization (Admin → Organization → Dataset consent):

| Mode | Behavior |
|------|--------------|
| `pseudonymized` | Stable hashed identifiers — cross-job correlation preserved |
| `anonymized` | Identifiers removed — full anonymity |
| `opted_out` | Excluded from all exports |

### Export the dataset

```bash
# Jobs (JSONL)
GET /api/v1/dataset/export

# Agent exchanges + feedback (alpaca / sharegpt / chatml formats)
GET /api/v1/training/export?format=alpaca
```

---

## 21. MCP Server

VibOps exposes its tools via the **Model Context Protocol** (MCP) — allowing any MCP client (Claude Desktop, Cursor, IDE) to operate GPU infrastructure directly from its own context.

### Connection

```json
{
  "mcpServers": {
    "vibops": {
      "command": "npx",
      "args": ["-y", "vibops-mcp"],
      "env": {
        "VIBOPS_API_URL": "https://vibops.yourcompany.com",
        "VIBOPS_API_TOKEN": "vbops_xxxxxxxx"
      }
    }
  }
}
```

### What you can do via MCP

The same tools as the VibOps agent are available: model deployment, scaling, incident diagnosis, GitOps, FinOps. The difference: you use them from your own LLM or IDE, without going through the VibOps console.

```
"Scale llama3 to 3 replicas in prod"     → patch_deployment
"Show idle GPUs on the GPU cluster"      → accelerator_detect_waste
"Deploy NIM for mistral-7b"              → nim_deploy (with guardrails)
```

The same guardrails apply: PolicyEngine, dry-run preview, confirmation required for destructive actions.

See the [`vibops-mcp`](https://github.com/VibOpsai/vibops-mcp) repository for the full installation guide.

---

## 22. Quick reference

### Keyboard shortcuts

| Shortcut | Action |
|-----------|--------|
| `⌘K` / `Ctrl+K` | Focus the chat input field |
| `Enter` | Send message |
| `Shift+Enter` | New line in message |
| `Escape` | Close the open modal |

### Language selector

Click the language selector (top right of the navigation bar) to change the interface language and agent response language.

Available languages: Français, English, Español, Deutsch, Italiano, Português, 日本語, 中文

The preference is saved locally and applied to both interface labels and agent responses.

### Personas and typical use cases

| Persona | Typical questions |
|---------|-------------------|
| **SRE / On-call** | Incident diagnosis, emergency rollback, last-hours audit, auto-healing |
| **MLOps engineer** | Inference fleet status, staging→prod promotion, benchmarking, capacity planning |
| **DevOps / Platform** | Deploying new services, creating environments, configuring clusters |
| **Engineering manager** | Audit trail, SLO compliance, GPU cost report, overall health status |
| **FinOps** | GPU cost report, idle GPU detection, rightsizing |

### Common error messages

| Message | Cause | Action |
|---------|-------|--------|
| `Cluster unreachable` | The kubeconfig context is no longer valid | Reconnect the gateway or update the kubeconfig |
| `Confirmation required` | Action blocked by production guardrails | Reply "yes" to confirm or "cancel" |
| `GPU limit exceeded` | License GPU quota reached | Contact david@vibops.ai to upgrade |
| `User limit reached` | Maximum number of users reached for your plan | Upgrade the plan or delete inactive accounts |
| `Licence expired` | License expired | Contact david@vibops.ai |
| `No Prometheus configured` | Metrics not available | See Admin → Integrations → Prometheus |

### Support

- Documentation: this file and `docs/installation.md`
- API: `docs/openapi.json`
- Issues: open a ticket with your VibOps contact
- License and pricing: `david@vibops.ai`
