# Monitoring & Observability

The stack is monitored end to end with Prometheus and Grafana, deployed through
the same GitOps flow as everything else. Cluster, node, and application metrics
all flow into one place.

## What's deployed

- **kube-prometheus-stack** (Prometheus, Grafana, node-exporter,
  kube-state-metrics) as its own Argo CD Application. Alertmanager is disabled and
  resource requests are trimmed to fit a small node group.
- **Application metrics** from the Django backend via `django-prometheus`, exposed
  at `/metrics` and scraped through a `ServiceMonitor`.

## How it's wired into GitOps

Rather than a plain `helm install`, the monitoring stack is a small wrapper chart
in this repository that declares `kube-prometheus-stack` as a Helm dependency, so
its values live in git under the `kube-prometheus-stack:` key and it is deployed
by an Argo CD Application like every other component. Monitoring is declared in
git alongside the app.

The `ServiceMonitor` for the backend lives in `charts/backend`, so scraping is
described next to the service it scrapes. It selects the backend Service by label
and carries the `release: monitoring` label so the Prometheus Operator adopts it.

## Problems and solutions

### 1. The stack didn't fit on the node group

**Symptom:** A node-exporter pod stayed `Pending`.

**Cause:** Small instance types cap how many pods a node can hold (an ENI/IP
limit, not memory). Prometheus and its exporters, on top of Argo CD, the External
Secrets Operator, the load balancer controller, and the application, pushed one
node over its pod limit. `Pending` here meant "no pod slot," not "no memory."

**Fix:** Ran more nodes. In production the real answer is VPC CNI **prefix
delegation** (`ENABLE_PREFIX_DELEGATION`), which raises the pods-per-node limit
sharply, or larger instance types, or dedicated monitoring nodes. Monitoring is
not a free passenger; it needs its resources planned for.

### 2. Argo CD couldn't sync the Operator CRDs

**Symptom:** Sync failed on the Prometheus Operator custom resource definitions.

**Cause:** Those CRDs are very large and exceed the size limit of the
last-applied-configuration annotation that a client-side apply writes.

**Fix:** `ServerSideApply=true` in the Application's sync options, which applies
resources server-side and avoids that annotation entirely.

### 3. The application target was discovered but never scraped

**Symptom:** The backend showed up in Prometheus service discovery but never
appeared in the active targets, so no application metrics were collected. Hitting
the pod's `/metrics` by hand returned 200, so the app was fine.

**Cause:** A `ServiceMonitor`'s `endpoints.port` matches on the Service port's
**name**, which is distinct from the container's `targetPort`. The backend
Service defined the port but left it unnamed, so there was no name to match and
the endpoint was dropped.

**Fix:** Named the Service port `http`. The target scraped immediately.

**Takeaway:** "Discovered but dropped" in Prometheus is a matching problem, not a
connectivity one. When the endpoint answers by hand but the scrape never happens,
check whether the selector and port name actually resolve, not the network.

## Note on annotations

The older `prometheus.io/scrape` annotation style is a different, non-Operator
mechanism and is ignored by kube-prometheus-stack. With the Prometheus Operator,
a `ServiceMonitor` is the way to register a scrape target.
# Monitoring & Observability

The stack is monitored end to end with Prometheus and Grafana, deployed through
the same GitOps flow as everything else. Cluster, node, and application metrics
all flow into one place.

## What's deployed

- **kube-prometheus-stack** (Prometheus, Grafana, node-exporter,
  kube-state-metrics) as its own Argo CD Application. Alertmanager is disabled and
  resource requests are trimmed to fit a small node group.
- **Application metrics** from the Django backend via `django-prometheus`, exposed
  at `/metrics` and scraped through a `ServiceMonitor`.

## How it's wired into GitOps

Rather than a plain `helm install`, the monitoring stack is a small wrapper chart
in this repository that declares `kube-prometheus-stack` as a Helm dependency, so
its values live in git under the `kube-prometheus-stack:` key and it is deployed
by an Argo CD Application like every other component. Monitoring is declared in
git alongside the app.

The `ServiceMonitor` for the backend lives in `charts/backend`, so scraping is
described next to the service it scrapes. It selects the backend Service by label
and carries the `release: monitoring` label so the Prometheus Operator adopts it.

## Problems and solutions

### 1. The stack didn't fit on the node group

**Symptom:** A node-exporter pod stayed `Pending`.

**Cause:** Small instance types cap how many pods a node can hold (an ENI/IP
limit, not memory). Prometheus and its exporters, on top of Argo CD, the External
Secrets Operator, the load balancer controller, and the application, pushed one
node over its pod limit. `Pending` here meant "no pod slot," not "no memory."

**Fix:** Ran more nodes. In production the real answer is VPC CNI **prefix
delegation** (`ENABLE_PREFIX_DELEGATION`), which raises the pods-per-node limit
sharply, or larger instance types, or dedicated monitoring nodes. Monitoring is
not a free passenger; it needs its resources planned for.

### 2. Argo CD couldn't sync the Operator CRDs

**Symptom:** Sync failed on the Prometheus Operator custom resource definitions.

**Cause:** Those CRDs are very large and exceed the size limit of the
last-applied-configuration annotation that a client-side apply writes.

**Fix:** `ServerSideApply=true` in the Application's sync options, which applies
resources server-side and avoids that annotation entirely.

### 3. The application target was discovered but never scraped

**Symptom:** The backend showed up in Prometheus service discovery but never
appeared in the active targets, so no application metrics were collected. Hitting
the pod's `/metrics` by hand returned 200, so the app was fine.

**Cause:** A `ServiceMonitor`'s `endpoints.port` matches on the Service port's
**name**, which is distinct from the container's `targetPort`. The backend
Service defined the port but left it unnamed, so there was no name to match and
the endpoint was dropped.

**Fix:** Named the Service port `http`. The target scraped immediately.

**Takeaway:** "Discovered but dropped" in Prometheus is a matching problem, not a
connectivity one. When the endpoint answers by hand but the scrape never happens,
check whether the selector and port name actually resolve, not the network.

## Note on annotations

The older `prometheus.io/scrape` annotation style is a different, non-Operator
mechanism and is ignored by kube-prometheus-stack. With the Prometheus Operator,
a `ServiceMonitor` is the way to register a scrape target.
