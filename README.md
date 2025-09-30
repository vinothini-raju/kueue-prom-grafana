# kueue-prom-grafana

Grafana dashboards for monitoring **Kueue scheduling** and **Slurm workloads** with Prometheus and kube-state-metrics.  
Track admissions, pending workloads, job/pod state, latency, evictions (Kueue) and cluster/job states (Slurm).

This repo provides ready-to-import Grafana dashboards that unify:

- **Kueue controller metrics** (`kueue_*`) + **Kubernetes state** (from kube-state-metrics)
- **Slurm cluster exporter metrics** (`slurm_*` resources, nodes, CPUs, memory)
- **Slurm job exporter metrics** (live job counts by state, partition, user, account)

> 🖼️ **Dashboard preview**  
![Kueue Overview Dashboard](/images/kueue-minimalist-dashboard-cluster-queue.png)

---

## What you get

### Kueue Dashboard
- **Kueue health** (is the controller scraped?)
- **Admitted workloads** and **pending workloads** (Kueue metrics)
- **Latency**: P95 admission attempt duration
- **Success ratio** of admissions
- **Kueue-managed Jobs & Pods** (from kube-state-metrics):
  - Jobs by status (now)
  - Active jobs by name (history-friendly)
  - Pending pods by job (Kueue-managed only)
- Variable filters:
  - `cluster_queue` (regex; default: all)
  - `kueue_ns` (namespaces with Kueue-managed jobs; default: all)

### Slurm Dashboard
- **Cluster state**:
  - CPU load, total CPUs, idle CPUs
  - Memory (real, allocated, free)
  - Nodes per state (`idle`, `alloc`, `mixed`, …)
  - CPUs per state
- **Job state**:
  - Jobs by state (pending, running, completing, failed, cancelled, timeout, completed)
  - Jobs by partition
  - Jobs by user
  - Jobs by account
- **Exporter health**:
  - Scrape duration for node & job exporters
- **Latest snapshot** of cluster and job states

---

## Requirements

### Common
- **Grafana** (any recent version; dashboards built with schema version 37 / Grafana 9+).
- **Prometheus** as Grafana datasource.
- **Prometheus Operator / kube-prometheus-stack** recommended:
  - The stack sets a label selector like  
    `spec.serviceMonitorSelector.matchLabels.release=<helm release>`
  - Ensure ServiceMonitors for Kueue and Slurm are labeled with this `release`.
- Prometheus must have RBAC permission to **GET `/metrics`** on the monitored components.
- Network access from Prometheus to metric endpoints (cluster-internal ports open).

---

### Kueue Dashboard prerequisites
- **Kueue controller metrics**:
  - `/metrics` on the controller’s HTTPS port (namespace: `kueue-system`)
  - ServiceMonitor for the controller with the correct `release` label
  - Prometheus service account bound with a ClusterRole allowing `get` on `/metrics`
- **kube-state-metrics** configured with:
  - `--metric-annotations-allowlist=jobs=[kueue.x-k8s.io/queue-name]`
  - `--metric-labels-allowlist=pods=[kueue.x-k8s.io/podset]`
- **Expected metrics must be present** in Prometheus:
  - `kueue_admission_attempts_total`
  - `kueue_admission_attempt_duration_seconds_bucket`
  - `kueue_pending_workloads`
  - `kube_job_status_active`, `kube_job_status_succeeded`, `kube_job_status_failed`
  - `kube_pod_status_phase`, `kube_pod_labels`, `kube_job_annotations`
- **Variables**:
  - `cluster_queue` → from `label_values(kueue_admission_attempts_total, cluster_queue)`
  - `kueue_ns` → from `label_values(kube_job_annotations{annotation_kueue_x_k8s_io_queue_name!=""}, namespace)`

---

### Slurm Dashboard prerequisites
- **Slurm control plane**:
  - Running `slurmctld` with `munge` started and valid `SLURM_CONF`
- **Two exporters scraped by Prometheus**:
  1. **Cluster/partition exporter** (`prometheus-slurm-exporter`)  
     - Provides CPU load, CPUs per state, memory usage, node counts  
     - Runs alongside `slurmctld`, exposes `:9092`  
     - Service + ServiceMonitor with `release` label
  2. **Job exporter** (Python-based)  
     - Runs `squeue` + `scontrol` to count jobs by state/partition/user/account  
     - Exposes `:9103`  
     - Service + ServiceMonitor with `release` label  
     - Needs `/etc/slurm/slurm.conf`, `squeue`, and `scontrol` available
- **Expected metrics must be present** in Prometheus:
  - Cluster exporter:
    - `slurm_cpu_load`
    - `slurm_cpus_total`, `slurm_cpus_idle`
    - `slurm_mem_real`, `slurm_mem_alloc`, `slurm_mem_free`
    - `slurm_cpus_per_state{state=...}`
    - `slurm_node_count_per_state{state=...}`
  - Job exporter:
    - `slurm_job_count_per_state{partition="...",state="...}`
    - `slurm_job_count_total{state="...}`
    - `slurm_user_state_total{user="...",state="...}`
    - `slurm_account_job_state_total{account="...",state="..."}`

- **Zero baseline requirement**:  
  The job exporter must publish **0 values** for common states (`PENDING, RUNNING, COMPLETING, SUSPENDED, CANCELLED, FAILED, TIMEOUT, COMPLETED`) and partitions even if no jobs exist.  
  This ensures Grafana panels don’t go blank when states are absent.

---

## Quick start (import)

1. Open Grafana → **Dashboards → New → Import**
2. Paste the content of `dashboards/kueue-overview.json` or `dashboards/slurm-overview.json`
3. Choose your **Prometheus** datasource → **Import**
4. Adjust variables at the top-left:
   - For Kueue: `cluster_queue` and `kueue_ns`
   - For Slurm: none required (panels derive from exporter metrics)

---

## Repo layout

kueue-prom-grafana/
└─ dashboards/
├─ kueue-overview.json
└─ slurm-overview.json

---

## Troubleshooting

See **[TROUBLESHOOT.md](TROUBLESHOOT.md)** for:
- Confirming Prometheus target discovery
- PromQL smoke tests (`kueue_*`, `slurm_*`)
- RBAC/TLS checks
- Common empty-panel causes (e.g. ServiceMonitor not labeled with `release`)

---

## Tested environment

- **MicroK8s v1.27** (single-node lab on AWS EC2 instance without GPUs)  
- **Slurm-on-K8s** lab cluster with `slurmctld`, `slurmd`, `munge`, and exporters sidecars  
  (behavior and label sets may vary slightly on other Kubernetes distributions/versions)

---

## AI-assisted content

Portions of these dashboards and documentation were **AI-generated** and then **validated and tested by a human** in the environment noted above. Please review and adapt queries/labels to match your cluster.

---

## License

This repo is licensed under `Business Source License 1.1`. See **[LICENSE.md](LICENSE.md)** for details.
