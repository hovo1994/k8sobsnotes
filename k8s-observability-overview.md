# Kubernetes observability stack overview

How metrics and logs get from a pod to a Grafana panel in a generic
Kubernetes cluster running Prometheus, Loki, Alloy and Grafana. High level;
the query guides cover what to do once the data is there:

- [prometheus-query-optimization.md](prometheus-query-optimization.md)
- [loki-query-optimization.md](loki-query-optimization.md)

**Diagram:** [k8s-observability-stack.drawio](k8s-observability-stack.drawio).
Open it in [app.diagrams.net](https://app.diagrams.net) or the draw.io VS
Code extension. The Mermaid version below renders inline on GitHub and
GitLab.


## Components

| Component | Runs as | Default port | Job |
|---|---|---|---|
| kubelet + cAdvisor | Every node, part of Kubernetes | 10250 | Container CPU, memory, network, filesystem per pod (`container_*`). Writes container stdout to `/var/log/pods`. |
| node-exporter | DaemonSet | 9100 | Host metrics: CPU, memory, disk, network per node (`node_*`). |
| kube-state-metrics | Deployment | 8080 | Object state from the API server: pod phase, restarts, owners, replicas (`kube_*`). No resource usage. |
| Prometheus | StatefulSet, usually via Prometheus Operator | 9090 | Scrapes everything above, stores it in its TSDB, evaluates recording and alert rules. |
| Alertmanager | StatefulSet | 9093 | Dedupes, groups, silences and routes alerts from Prometheus and the Loki ruler. |
| Grafana Alloy | DaemonSet | 12345 (UI) | Tails pod logs on its node, attaches Kubernetes labels, pushes to Loki. Can also scrape metrics and remote-write them. |
| Loki | Deployment / StatefulSets | 3100 | Stores logs. Indexes labels only; chunks go to object storage. |
| Grafana | Deployment | 3000 | Dashboards, Explore, alerting UI. Queries Prometheus and Loki; stores no telemetry itself. |
| Object storage | Outside the cluster | | Loki chunks and index. Optionally Thanos/Mimir blocks. |

## How data flows

### Metrics: pull

Prometheus discovers targets through the API server (pods, services,
endpoints, nodes) and scrapes each one's `/metrics` endpoint on an interval,
usually 30s. Every scrape adds one sample per series. With the Prometheus
Operator, targets are declared as `ServiceMonitor` and `PodMonitor`
resources rather than in `prometheus.yml`.

The three Kubernetes sources answer different questions:

| Question | Source | Example metric |
|---|---|---|
| How much CPU is this container using? | cAdvisor (kubelet) | `container_cpu_usage_seconds_total` |
| Is the node's disk filling up? | node-exporter | `node_filesystem_avail_bytes` |
| How many times has this pod restarted? Which Deployment owns it? | kube-state-metrics | `kube_pod_container_status_restarts_total`, `kube_pod_owner` |

Joining them, e.g. CPU usage by Deployment, means joining cAdvisor series
to kube-state-metrics series on `namespace` and `pod`. See the Joins section
of the Prometheus guide.

### Logs: push

Containers write to stdout/stderr. The container runtime writes that to
`/var/log/pods/<namespace>_<pod>_<uid>/<container>/` on the node. Alloy, one
per node, tails those files, looks up the pod in the API server, attaches
labels (`namespace`, `container`, `app`, and `pod` as structured metadata),
and pushes batches to the Loki distributor.

Inside Loki:

1. **Distributor** validates, applies rate limits, and hashes each stream to
   ingesters.
2. **Ingester** buffers lines in memory, compresses them into chunks, and
   flushes chunks and index to object storage.
3. **Query-frontend** splits a query by time, queues it and caches results.
4. **Querier** reads recent data from ingesters and older data from object
   storage, then runs the LogQL pipeline.
5. **Compactor** compacts the index and enforces retention.
6. **Ruler** evaluates LogQL alert and recording rules.

Small installs run all of these in one process (monolithic or "simple
scalable" mode). The components are the same.

The labels Alloy attaches are the stream labels you select on in LogQL.
Collector config decides query performance before anyone writes a query:
a high-cardinality label added here slows every query on the tenant.

### Queries

Engineers reach Grafana through the cluster's Ingress or Gateway. Grafana
has a Prometheus data source pointed at Prometheus (or Thanos/Mimir), and a
Loki data source pointed at the Loki query-frontend. Each panel query is
sent as PromQL or LogQL; Grafana stores dashboards, not data.

Correlating the two depends on shared labels. If metrics and logs both carry
`namespace`, `pod` and `container` with the same values, Grafana can jump
from a CPU spike on a panel to that pod's logs in Explore.

### Alerts and rules

- Prometheus evaluates alert rules and sends firing alerts to Alertmanager.
- Prometheus also evaluates recording rules and writes the results back as
  new series.
- The Loki ruler does the same for LogQL. Alerts go to the same
  Alertmanager; recording rule results are remote-written into Prometheus,
  so a log-derived number can be graphed and alerted on with PromQL.
- Alertmanager groups, dedupes and silences, then routes to Slack,
  PagerDuty or email.

## Typical install

| Helm chart | Installs |
|---|---|
| `prometheus-community/kube-prometheus-stack` | Prometheus Operator, Prometheus, Alertmanager, node-exporter, kube-state-metrics, Grafana, and the kubernetes-mixin dashboards and rules. |
| `grafana/loki` | Loki in monolithic, simple scalable or distributed mode. |
| `grafana/alloy` | Alloy as a DaemonSet. Or `grafana/k8s-monitoring`, which wraps Alloy with Kubernetes defaults. |

## Where query performance is decided

| Layer | Decision | Effect on queries |
|---|---|---|
| Exporters and apps | Which labels exist on metrics | Prometheus series count. |
| Alloy / scrape config | Which labels become Loki stream labels; `metric_relabel_configs` | Loki stream count; dropped Prometheus labels. |
| Prometheus / Loki config | Retention, limits, caching, query splitting | What can be queried and how fast. |
| Recording rules | Precomputed aggregates | Dashboards read a few series instead of thousands. |
| Grafana panels | Time range, step, instant vs range | Samples and bytes read per refresh. |

## References

- [Prometheus overview and architecture](https://prometheus.io/docs/introduction/overview/)
- [Prometheus Kubernetes service discovery](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)
- [Prometheus Operator: ServiceMonitor and PodMonitor](https://prometheus-operator.dev/docs/developer/getting-started/)
- [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
- [node-exporter](https://github.com/prometheus/node_exporter)
- [Kubernetes: logging architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Kubernetes: resource metrics pipeline (cAdvisor)](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Loki architecture and components](https://grafana.com/docs/loki/latest/get-started/components/)
- [Loki deployment modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
- [Grafana Alloy: collect Kubernetes logs](https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/)
- [Grafana Kubernetes Monitoring Helm chart](https://github.com/grafana/k8s-monitoring-helm)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
