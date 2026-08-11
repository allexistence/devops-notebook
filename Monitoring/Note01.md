# Complete Monitoring Stack Guide — Prometheus, Grafana, Loki, Thanos

Written in plain language, story-style where it helps. This is the full version covering install, logs, cross-cluster + long-term storage, security, dashboards, and real Day-2 operations.

---

# PART 1 — Install Prometheus + Grafana + Exporters (Helm)

The core team:
- **Prometheus** = walks around asking "what's your number right now?" and writes it down.
- **Exporters** = translators next to each machine/app, converting raw info into numbers.
- **Grafana** = draws graphs from Prometheus's notebook.
- **Alertmanager** = taps you on the shoulder when a number looks bad.
- **Loki** = same idea as Prometheus, but for *logs* instead of numbers (Part 5).

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

`values-monitoring.yaml`:
```yaml
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: portworx-sc
          resources: { requests: { storage: 100Gi } }
    externalLabels:
      cluster: cluster-1

grafana:
  adminPassword: "changeme"
  persistence: { enabled: true, size: 10Gi }

nodeExporter: { enabled: true }
kubeStateMetrics: { enabled: true }
```

```bash
helm upgrade --install kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f values-monitoring.yaml
```

Bundled automatically: Prometheus, Grafana, Alertmanager, node-exporter (per-node CPU/RAM/disk), kube-state-metrics (K8s object health), and kubelet/cAdvisor scraping (container-level cgroup CPU/memory).

Add separately as needed: DCGM exporter (GPU), blackbox-exporter (URL/ping checks), process-exporter (per-process detail), Portworx metrics (already exposed, just needs a ServiceMonitor).

---

# PART 2 — Loki: Log Aggregation Alongside Metrics

### Why you need it
Prometheus tells you a number went wrong ("memory spiked at 14:32"). It cannot tell you *why*. That's what logs are for. **Loki is "Prometheus but for logs"** — same label-based model, much cheaper to run than the ELK stack because it doesn't index the full text of every log line, only the labels.

### Install
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki-stack \
  -n monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set loki.persistence.storageClassName=portworx-sc
```

- **Promtail** (or Grafana Agent / Alloy in newer setups) is the collector — it runs as a DaemonSet, tails every container's log file on the node, and pushes lines to Loki, tagging each line with the same kind of labels Prometheus uses: `namespace`, `pod`, `container`, `node`.
- **Loki** stores and indexes only those labels, not the full text — so it stays lightweight even at huge log volume.

### Add Loki as a Grafana datasource
```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki.monitoring.svc.cluster.local:3100
      access: proxy
```

Now in Grafana, when a Prometheus graph shows a memory spike, you flip to the **Explore** view, pick Loki, filter `{namespace="prod", pod="my-app-xyz"}`, and read the actual application logs from that exact time window — this is called "metrics-to-logs correlation" and it's the single biggest reason to run both together.

---

# PART 3 — Thanos: Long-Term Storage + True Cross-Cluster Query

### The problem Thanos solves
A single Prometheus has two weaknesses: (1) local disk fills up, so you can't keep years of data, and (2) it only knows about its own cluster. Thanos fixes both by moving data to cheap object storage and adding a query layer that reads from *every* cluster at once.

### Components
| Component | Job |
|---|---|
| **Thanos Sidecar** | Runs next to each cluster's Prometheus, uploads its data blocks to object storage (S3/MinIO), and lets Thanos Query read recent data directly from that Prometheus |
| **Thanos Store Gateway** | Serves *old* data straight from object storage without needing Prometheus to hold it locally |
| **Thanos Query (Querier)** | The single entry point Grafana talks to — fans out to every Sidecar + Store Gateway, merges results, dedupes if you run HA Prometheus pairs |
| **Thanos Compactor** | Runs in the background, compacts and downsamples old blocks in object storage so old queries stay fast |
| **Thanos Receive** | Alternative to Sidecar — clusters push data in via `remote_write` instead of Thanos pulling it. Simpler when clusters are behind NAT/firewalls and can't be reached inbound |

### Minimal setup per cluster (Sidecar model)
```yaml
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: cluster-1
    thanos:
      image: quay.io/thanos/thanos:v0.35.0
      objectStorageConfig:
        key: thanos.yaml
        name: thanos-objstore-secret
```
```yaml
# thanos-objstore-secret (as a K8s secret, referenced above)
type: S3
config:
  bucket: monitoring-thanos
  endpoint: s3.amazonaws.com
  access_key: <key>
  secret_key: <secret>
```

### Central query layer (in your hub/Grafana cluster)
```bash
helm upgrade --install thanos-query bitnami/thanos \
  -n monitoring \
  --set query.enabled=true \
  --set query.stores="{prometheus-cluster1.example.com:10901,prometheus-cluster2.example.com:10901}"
```

Grafana then points at **one URL** — Thanos Query — and can graph a metric across all clusters at once, filtered by the `cluster` label you set earlier.

**Rule of thumb:** 2–3 clusters, short retention → plain `remote_write` (Part covered before) is enough. Long retention, many clusters, or clusters behind firewalls → Thanos.

---

# PART 4 — Cross-Cluster Monitoring, Detailed

### Scenario: Grafana lives in Cluster A (hub), Cluster B is a spoke you want fully monitored

**Step 1** — Cluster B gets its own Prometheus (it must — nothing can scrape a cluster's internals from outside the cluster).

**Step 2** — Cluster B's Prometheus either:
- pushes data out via `remote_write` to Cluster A, or
- runs a Thanos Sidecar/Receive so Cluster A's Thanos Query can pull/receive it.

**Step 3** — Cluster B sets `externalLabels: { cluster: cluster-2 }` so its data never gets confused with Cluster A's own metrics once merged.

**Step 4** — Grafana in Cluster A never talks to Cluster B directly. It only ever talks to its **local** Prometheus (remote_write model) or **local** Thanos Query (Thanos model) — both of which already contain Cluster B's data.

This is the important mental model: **data flows outward from the spoke cluster to the hub — Grafana never reaches inward into a remote cluster.** That keeps the spoke clusters' API/network surface closed to the outside, which is also a security win (Part 8).

---

# PART 5 — Storage Management

### What's actually being stored where
| Data | Stored in | Grows with |
|---|---|---|
| Recent metrics (Prometheus TSDB) | Local PVC on Prometheus pod | retention days × scrape frequency × number of unique label combinations (cardinality) |
| Long-term metrics | Object storage (S3/MinIO) via Thanos | same, but cheap and effectively unlimited |
| Logs | Loki's PVC or object storage backend | log volume × retention |
| Dashboards/config | Grafana's own PVC (or a database if using external DB) | small, rarely an issue |

### Sizing Prometheus disk (rule of thumb)
```
disk needed ≈ retention_days × ingested_samples_per_day × bytes_per_sample (~1-2 bytes compressed)
```
In practice: watch `prometheus_tsdb_storage_blocks_bytes` and set an alert at 80% of the PVC.

### Keeping storage under control
1. **Set retention explicitly** (`retention: 15d`) — never leave it unset.
2. **Watch cardinality** — a single label like `user_id` or `request_id` on a metric can create millions of unique series and blow up disk/memory. Use `prometheus_tsdb_symbol_table_size_bytes` and `topk(10, count by (__name__)(...))` to find offenders.
3. **Move old data to Thanos** instead of growing local retention — local disk should only hold a few days' "hot" data; Thanos Store Gateway serves the rest from object storage.
4. **Downsample** — Thanos Compactor automatically creates 5m/1h resolution rollups of old data so a 1-year graph doesn't have to scan raw 15-second samples.
5. **Loki**: set `retention_period` and enable the compactor so old log chunks get deleted/archived instead of growing forever.

---

# PART 6 — Troubleshooting (Deep Dive)

### A structured way to debug "I don't see my metric"
1. **Is the exporter even running?** `kubectl get pods -n <ns> -l app=<exporter>`
2. **Is the exporter's `/metrics` endpoint reachable?** `kubectl port-forward` to it and `curl localhost:PORT/metrics` — if this fails, the problem is the app, not Prometheus.
3. **Does a ServiceMonitor/PodMonitor exist pointing at it?** `kubectl get servicemonitor -A`
4. **Does Prometheus actually see it?** Open Prometheus UI → **Status → Targets**. If it's not listed at all, the ServiceMonitor's label selector doesn't match Prometheus's `serviceMonitorSelector`. If it's listed but "down", it's a network/auth/TLS issue — read the error text shown there.
5. **Is the metric actually being scraped but just named differently?** Use Prometheus UI → Graph → type the exporter name prefix and autocomplete to browse what's really there.

### A structured way to debug "my alert didn't fire"
1. Check the rule itself in **Status → Rules** — is it in `firing`, `pending`, or `inactive` state?
2. If `pending`, the `for:` duration hasn't elapsed yet.
3. If `inactive`, the expression currently evaluates false — test the raw PromQL manually in the Graph tab.
4. If `firing` but you got no notification, the problem is in Alertmanager — check **Alertmanager UI → Silences** (someone may have muted it) and its routing tree.

### A structured way to debug "Grafana panel shows No Data"
1. Check the datasource health: Grafana → Connections → Data sources → Test.
2. Check the exact PromQL query the panel runs (panel → Edit → Query inspector) — copy it into Prometheus's own Graph tab and see if data comes back there.
3. Check dashboard template variables (like `$cluster` or `$namespace`) — a wrong default value silently filters everything out.

---

# PART 7 — Real-Time Monitoring

"Real-time" in this stack really means: **short scrape interval + live dashboards + alerting with short `for:` windows.**

- Default scrape interval is usually 30s–60s; for real-time dashboards (e.g. an incident war-room view) drop specific ServiceMonitors to `interval: 5s` or `10s` — but only for what truly needs it, since this multiplies storage and CPU cost.
- Grafana dashboards auto-refresh (top-right refresh picker) — set to 5s/10s during an incident.
- For genuinely real-time system stats beyond what scraping intervals give you, `kubectl top` / `metrics-server` gives instantaneous CPU/memory (used by HPA), separate from the Prometheus stack.
- Loki + Promtail can tail logs live — Grafana's Explore view has a "Live" toggle for logs, streaming new lines as they're written, similar to `kubectl logs -f`.

---

# PART 8 — Upgrade with Rollback

### Before every upgrade
```bash
helm get values kube-prom-stack -n monitoring -o yaml > backup-values-$(date +%F).yaml
helm history kube-prom-stack -n monitoring
```

### Doing the upgrade safely
```bash
# Preview the diff first (requires helm-diff plugin)
helm diff upgrade kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-monitoring.yaml

# Then apply
helm upgrade kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-monitoring.yaml
```

### Rolling back
```bash
helm history kube-prom-stack -n monitoring          # find the revision number you want
helm rollback kube-prom-stack <REVISION> -n monitoring
```
Helm rollback reverts the release's manifests, but **not** any data already written to PVCs (that data just stays as-is — rollback is about config, not about metrics/dashboard data). CRDs (like `ServiceMonitor`, `PrometheusRule` definitions) are usually **not** rolled back automatically by Helm — check `helm upgrade --set installCRDs=...` behavior on that chart version, or manage CRDs separately with `kubectl apply` if the chart warns about it.

### Good habits
- Never change things with `kubectl edit` — always change `values-monitoring.yaml` and re-run `helm upgrade`, or the next Helm run will silently overwrite your manual edit.
- Pin the chart version explicitly (`--version 62.x.x`) rather than always pulling latest, so upgrades are deliberate.
- Test upgrades in a staging cluster first if you have one — Prometheus Operator major version bumps occasionally change CRD schemas.

---

# PART 9 — Security: Tooling, Ingress, and Login

### Grafana login
- Never leave the default `admin/prom-operator` or a plaintext password in values.yaml. Reference an existing secret instead:
```yaml
grafana:
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password
```
- For real teams, wire up **OAuth/OIDC** (Azure AD, Okta, Keycloak, GitHub) instead of local accounts:
```yaml
grafana:
  grafana.ini:
    auth.generic_oauth:
      enabled: true
      client_id: <id>
      client_secret: <secret>
      scopes: openid profile email
      auth_url: https://login.microsoftonline.com/<tenant>/oauth2/v2.0/authorize
      token_url: https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
```

### Ingress
```yaml
grafana:
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
    hosts: ["grafana.internal.example.com"]
    tls:
      - secretName: grafana-tls
        hosts: ["grafana.internal.example.com"]
```
Do the same pattern for Prometheus/Alertmanager UIs if you expose them — but strongly consider putting them **behind an internal-only ingress class or a VPN**, since Prometheus's UI has no built-in auth by default (add `nginx.ingress.kubernetes.io/auth-type: basic` or put an OAuth2-proxy in front if you must expose it).

### Tool-to-tool security
- **Prometheus → kubelet**: uses TLS + a ServiceAccount token with RBAC (`nodes/metrics`, `nodes/proxy`) — already handled by the chart's ClusterRole, just don't strip it.
- **Prometheus → remote_write target**: use `basicAuth` or `bearerTokenSecret`, referencing a K8s Secret, never inline credentials.
- **Thanos → object storage**: IAM role (IRSA on EKS, workload identity on AKS/GKE) is safer than static access keys where the platform supports it.
- **NeuVector / network policies**: restrict which namespaces can reach `monitoring` namespace pods, and restrict `monitoring` namespace egress to only what it needs (kube-apiserver, object storage endpoint, remote_write target).

---

# PART 10 — Dashboard JSON, In Detail

### Getting a dashboard JSON
- From grafana.com/dashboards — search, copy the numeric ID (e.g. `1860` for node-exporter-full).
- Or build/export your own: Dashboard → Settings → JSON Model → copy.

### Loading it into Helm-managed Grafana
**Option A — by ID (auto-downloaded at install time):**
```yaml
grafana:
  dashboards:
    default:
      node-exporter-full:
        gnetId: 1860
        revision: 32
        datasource: Prometheus
```

**Option B — your own JSON pasted inline:**
```yaml
grafana:
  dashboards:
    default:
      my-app-dashboard:
        json: |
          {
            "title": "My App Overview",
            "panels": [ ... ]
          }
```

**Option C — sidecar ConfigMap (recommended, GitOps-friendly, no Helm upgrade needed to add/change a dashboard):**
```bash
kubectl create configmap my-app-dashboard --from-file=my-app-dashboard.json -n monitoring
kubectl label configmap my-app-dashboard grafana_dashboard=1 -n monitoring
```
Requires:
```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      searchNamespace: ALL
```
The sidecar container inside the Grafana pod watches the K8s API (again, via a ServiceAccount with RBAC to `list/watch` ConfigMaps) for anything with that label, downloads the JSON, and writes it into Grafana's dashboard folder live.

### Templating a dashboard for multi-cluster
Add a `cluster` dashboard variable so one dashboard JSON works everywhere:
```json
"templating": {
  "list": [
    {
      "name": "cluster",
      "type": "query",
      "query": "label_values(up, cluster)"
    }
  ]
}
```
Then every panel's query includes `{cluster="$cluster"}` so switching the dropdown re-filters the whole dashboard.

---

# PART 11 — Integrating Multiple Prometheus Instances into One Centralized Grafana (as Datasources)

Simplest form of centralization — before you even need Thanos:

```yaml
grafana:
  additionalDataSources:
    - name: Prometheus-Cluster1
      type: prometheus
      url: https://prometheus-cluster1.example.com
      access: proxy
      isDefault: true
      jsonData:
        httpHeaderName1: Authorization
      secureJsonData:
        httpHeaderValue1: "Bearer <token>"
    - name: Prometheus-Cluster2
      type: prometheus
      url: https://prometheus-cluster2.example.com
      access: proxy
      jsonData:
        httpHeaderName1: Authorization
      secureJsonData:
        httpHeaderValue1: "Bearer <token>"
```

Now Grafana has **two separate datasources**, one per cluster. You pick which one a panel queries via a datasource variable:
```json
"templating": { "list": [ { "name": "DS_PROM", "type": "datasource", "query": "prometheus" } ] }
```
and set the panel's datasource to `${DS_PROM}`.

**Difference vs. Thanos:** with separate datasources, you can view Cluster 1 or Cluster 2, but you **cannot** sum/merge a metric across both in a single query — each panel talks to one datasource at a time. Thanos Query (Part 3) solves that by presenting *all* clusters as if they were one datasource. Use plain multi-datasource setup when clusters are meant to be viewed separately (e.g. different customers/environments); use Thanos when you need true merged views (e.g. "total GPU usage across all clusters").

Exposing each remote Prometheus safely for this requires an Ingress + TLS + auth token on each cluster's Prometheus (see Part 9) — you're now allowing inbound access into each spoke cluster, which is the trade-off against the "data flows outward" model in Part 4.

---

# PART 12 — Federating Metrics via Kubernetes RBAC over the kube-apiserver (Central Prometheus Model)

This is a different pattern: instead of each cluster running its *own full* Prometheus, one **central Prometheus** reaches into other clusters' kube-apiservers directly to discover and scrape targets. Less common, more advanced, used when you want one single Prometheus brain instead of many.

### How it works
1. On the **remote cluster**, create a dedicated ServiceAccount with only read permissions:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: central-prometheus-reader
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: central-prometheus-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes/proxy"]
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: central-prometheus-reader
subjects:
  - kind: ServiceAccount
    name: central-prometheus-reader
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: central-prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

2. Generate a long-lived token for that ServiceAccount (K8s 1.24+ needs an explicit `Secret` of type `kubernetes.io/service-account-token`, since tokens aren't auto-created anymore):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: central-prometheus-reader-token
  namespace: monitoring
  annotations:
    kubernetes.io/service-account.name: central-prometheus-reader
type: kubernetes.io/service-account-token
```
```bash
kubectl get secret central-prometheus-reader-token -n monitoring -o jsonpath='{.data.token}' | base64 -d
```

3. On the **central Prometheus**, add a scrape config that talks straight to the remote cluster's kube-apiserver, using that token as a bearer credential, with `kubernetes_sd_configs` pointing at the remote API server URL:
```yaml
scrape_configs:
  - job_name: 'remote-cluster-2-pods'
    kubernetes_sd_configs:
      - role: endpoints
        api_server: https://cluster-2-apiserver.example.com:6443
        bearer_token_file: /etc/prometheus/secrets/cluster2-token
        tls_config:
          ca_file: /etc/prometheus/secrets/cluster2-ca.crt
    bearer_token_file: /etc/prometheus/secrets/cluster2-token
    tls_config:
      ca_file: /etc/prometheus/secrets/cluster2-ca.crt
```

This is exactly the same `kubernetes_sd_configs` mechanism Prometheus normally uses against its *own* local cluster — the only difference is `api_server` now points at a different cluster's API endpoint, authenticated with that cluster's own RBAC token instead of the local in-pod ServiceAccount token.

**Trade-off vs. remote_write/Thanos:** this requires the central Prometheus to have *inbound network reachability* to every remote cluster's API server (opposite of the "data flows outward" model), and RBAC tokens for every remote cluster need careful rotation. Most teams prefer remote_write/Thanos for this reason — federation-via-kube-API is mainly used when clusters are small, few, and already network-reachable (e.g. all in one VPC).

---

# PART 13 — The Full Metric Flow, In Detail (Exporters → `__meta_kubernetes_*` → Storage → Dashboard)

Walking one metric end-to-end, this time including exactly what happens at each hop:

**1. The exporter exposes a number.**
Every exporter (node-exporter, kube-state-metrics, DCGM, your own app) runs an HTTP server with a `/metrics` path that prints plain text like:
```
node_memory_MemAvailable_bytes 8321536000
```
This is the Prometheus exposition format — just a metric name, optional labels, and a number.

**2. Prometheus needs to find that endpoint — this is Service Discovery.**
Instead of you hardcoding IPs, Prometheus asks the Kubernetes API: "give me every Pod/Service/Endpoint/Node object right now, and keep me updated." For each object it gets back, Kubernetes SD attaches temporary `__meta_kubernetes_*` labels — e.g. `__meta_kubernetes_pod_name`, `__meta_kubernetes_namespace`, `__meta_kubernetes_pod_label_app`, `__meta_kubernetes_pod_annotation_prometheus_io_scrape`. These exist **only during discovery** — they are the raw material, not the final labels you'll query with.

**3. Relabeling decides who gets scraped, and what their final labels are.**
A `ServiceMonitor` (or raw `relabel_configs`) is the rulebook:
```yaml
relabel_configs:
  - action: keep
    source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    regex: "true"
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
```
Anything not explicitly copied over is discarded once scraping begins — that's why you'll never see `__meta_kubernetes_*` labels on an actual stored metric, only on the "Discovered Labels" debug view in the Prometheus UI.

**4. Prometheus scrapes the endpoint via RBAC-authenticated HTTP.**
Using its ServiceAccount token, Prometheus calls the exporter's `/metrics` URL (or, for kubelet/cAdvisor, `https://<node-ip>:10250/metrics/cadvisor`, requiring `nodes/metrics` and `nodes/proxy` RBAC verbs).

**5. The sample is stored in the TSDB with real labels attached.**
```
node_memory_MemAvailable_bytes{namespace="monitoring", pod="node-exporter-abc12", node="worker-3", cluster="cluster-1"} 8321536000  @1691740800
```

**6. If cross-cluster, the sample also travels via remote_write / Thanos** to the central store, carrying the `cluster` label so it never collides with another cluster's series.

**7. A `PrometheusRule` continuously evaluates PromQL against stored samples**, and fires alerts through Alertmanager if conditions hold for the configured `for:` duration.

**8. Grafana queries the same stored samples** (via its Prometheus/Thanos datasource) whenever you open a dashboard, using the exact labels that survived Step 3 — which is why *what you templated in relabeling* directly determines *what you can filter/group by in Grafana* later. If a label wasn't promoted out of `__meta_kubernetes_*` in Step 3, it simply doesn't exist for querying — Grafana can never show you a label Prometheus never kept.

### One-line summary of the whole chain
```
exporter /metrics → K8s SD discovers target (__meta_kubernetes_* labels) →
relabel_configs pick target + real labels → RBAC-authenticated scrape →
stored in TSDB (+ shipped cross-cluster via remote_write/Thanos) →
PromQL rules alert, Grafana visualizes
```