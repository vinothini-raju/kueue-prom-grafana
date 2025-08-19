# kueue-prom-grafana
Grafana dashboard for monitoring Kueue scheduling with Prometheus and kube-state-metrics—track admissions, pending workloads, job/pod state, latency, and evictions across cluster queues and namespaces.

This repo provides a ready-to-import Grafana dashboard that unifies Kueue controller metrics (`kueue_*`) and Kubernetes state (from kube-state-metrics) to give clear visibility into scheduling health and throughput. Panels cover controller scrape health, admitted/pending workloads, P95 admission latency, admission success ratio, Kueue-managed jobs by status/name, and pending pods by job. The dashboard is environment-agnostic and uses variables for `cluster_queue` and kueue_ns so you can filter without editing queries.

> 🖼️ **Dashboard preview**  
![Kueue Overview Dashboard](/images/kueue-minimalist-dashboard-cluster-queue.png)


## What you get

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

---

## Requirements

- **Grafana** (any recent version)
- **Prometheus** scraping:
  - Kueue controller metrics (`/metrics` on the Kueue controller’s HTTPS port)
  - **kube-state-metrics** (for `kube_job_*`, `kube_pod_*`, `kube_pod_labels`, `kube_job_annotations`)
- Prometheus must be configured to discover your **ServiceMonitor** for the Kueue metrics.

> The dashboard assumes:
> - Kueue-managed **Jobs** have annotation `kueue.x-k8s.io/queue-name`
> - Kueue-managed **Pods** have label `kueue.x-k8s.io/podset`

---

## Quick start (import)

1. Open Grafana → **Dashboards → New → Import**
2. Paste the content of `dashboards/kueue-overview.json` (or upload the file)
3. Choose your **Prometheus** datasource → **Import**
4. Adjust variables at the top-left:
   - `cluster_queue` → pick a specific queue or **All**
   - `kueue_ns` → select namespaces that run Kueue-managed jobs

---

## Repo layout

kueue-prom-grafana/
└─ dashboards/
└─ kueue-overview.json

---

## Troubleshooting

See **[TROUBLESHOOT.md](TROUBLESHOOT.md)** for:
- How to confirm Prometheus sees the Kueue target
- PromQL smoke tests for `kueue_*` metrics
- RBAC/TLS tips
- Common empty-panel causes & fixes

---

## Tested environment

- **MicroK8s v1.27** (single-node lab on AWS EC2 instance without GPUs).  
  Behavior and label sets may vary slightly on other Kubernetes distributions/versions.

---

## AI-assisted content

Portions of this dashboard and documentation were **AI-generated** and then **validated and tested by a human** in the environment noted above. Please review and adapt queries/labels to match your cluster.

---

## License

This repo is licensed under `Business Source License 1.1`.  See **[LICENSE.md](LICENSE.md)** for details.
