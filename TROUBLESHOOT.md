## Prometheus targets show Kueue is being scraped

1. **Open Prometheus UI → _Status_ → _Targets_** and search for “kueue”.
2. You should see a target similar to:
   - `serviceMonitor/kueue-system/kueue-controller-manager-metrics-monitor/0`, **or**
   - a job like `kueue-controller-manager-metrics-service`.
3. The target **State must be _up_**. Confirm:
   - **Last scrape** is recent (within your scrape interval),
   - **Scrape duration** is reasonable (e.g., < 1s),
   - **Labels** show `namespace="kueue-system"`, `instance="IP:8443"`, `__scheme__="https"`, `__metrics_path__="/metrics"`.

> **Tip:** **Status → Service discovery** shows what Prometheus discovered before relabeling; use it when the target is missing.

### Common target errors (and how to fix)

| Error (Targets page) | What it means | Fix |
| --- | --- | --- |
| `401 Unauthorized` | Prometheus SA lacks permission to read `/metrics`. | Bind a Role/ClusterRole that allows `GET` on the Kueue metrics endpoint (or authorize via token). |
| `x509: certificate signed by unknown authority` | Self-signed/unknown CA on HTTPS. | In the `ServiceMonitor`, configure a CA or set `tlsConfig.insecureSkipVerify: true` (lab/dev only). |
| `context deadline exceeded` / `connection refused` | Network/Service/port wrong or Pod not listening. | Verify `Service` port/name (`8443`/`https`), `Endpoints` IP/port, Pod health. |
| **No target at all** | Label mismatch between `ServiceMonitor` and Prometheus selector. | Ensure Prometheus `.spec.serviceMonitorSelector.matchLabels` matches your ServiceMonitor’s labels (often `release=...`). |
| `404` / `503` | Wrong path or backend error. | Ensure `spec.endpoints[].path: /metrics` and the controller exposes it. |

---

## PromQL smoke tests (Prometheus → _Graph_)

Run these in Prometheus’ **Graph** tab to verify scraping and metric presence.

### 1) Target health

```promql
up{job=~".*kueue.*"}
up{namespace="kueue-system", pod=~"kueue-controller-manager-.*"}
```

Expect `1` for at least one series.

### 2) Kueue metric presence

```promql
count({__name__=~"kueue_.*"})  # > 0 means Kueue metrics are being ingested
kueue_build_info
kueue_pending_workloads
sum by (result) (increase(kueue_admission_attempts_total[5m]))
histogram_quantile(0.95, sum by (le) (rate(kueue_admission_attempt_duration_seconds_bucket[5m])))
label_values(kueue_pending_workloads, cluster_queue)  # which ClusterQueues exist
```

> If `up==1` but `kueue_*` is empty, you’re likely scraping the wrong port/path/scheme or dropping via relabel rules.

### 3) kube-state-metrics (required for Jobs/Pods panels)

```promql
up{job=~".*kube-state-metrics.*"}                 # should be 1
count({__name__=~"kube_job_.*|kube_pod_.*"})      # expect > 0
kube_job_annotations{annotation_kueue_x_k8s_io_queue_name!=""}
kube_pod_labels{label_kueue_x_k8s_io_podset!=""}
kube_pod_owner{owner_kind="Job"}
```

If these are empty, kube-state-metrics isn’t running, not scraped, or is too old to expose annotations/labels.

---

## Curl the endpoint directly (RBAC/TLS sanity check)

1) **Port-forward** the metrics service:

```sh
kubectl -n kueue-system port-forward svc/kueue-controller-manager-metrics-service 8443:8443 &
```

2) **Get a Prometheus SA token** (adjust namespace/SA names as needed):

```sh
NS=observability
SA=kube-prom-stack-kube-prome-prometheus
TOKEN=$(kubectl -n "$NS" create token "$SA")
```

3) **Curl with Bearer token** (ignore TLS for labs):

```sh
curl -ksH "Authorization: Bearer $TOKEN" https://127.0.0.1:8443/metrics | grep '^kueue_' | head
```

- Output shows `kueue_*` lines → endpoint & RBAC are OK.  
- `Unauthorized` → grant RBAC to the Prometheus SA.  
- TLS errors without `-k` → add CA in the ServiceMonitor or keep `insecureSkipVerify: true` (dev only).

---

## If targets are up but dashboard panels are empty

- **Time range too narrow**: panels using `max_over_time()` only show activity within the selected window.
- **Variables too restrictive**: set **`$cluster_queue`** and **`$kueue_ns`** to **All** to sanity-check.
- **Jobs finished/GC’d**: use queries that include `max_over_time()` on both job status and annotations to retain history.
- **Label/annotation mismatch**:
  - Jobs must have `kueue.x-k8s.io/queue-name` (checked via `kube_job_annotations`).
  - Pods must have `kueue.x-k8s.io/podset` (checked via `kube_pod_labels`).
- **kube-state-metrics missing/outdated**: verify with the PromQL in the previous section.
