# Prometheus query optimization for Kubernetes metrics

Quick reference for writing PromQL that returns fast in Grafana based on Prometheus 3.x docs and source.

**Prometheus indexes every label; cost is series × samples.** The opposite
of Loki. Finding series is cheap; loading their samples is what costs. A
query touching 50,000 series over 7 days at a 30s scrape interval loads ~1
billion samples. Everything below is a way to touch fewer series, load fewer
samples, or do the work once in a recording rule.

## Selectors

```promql
container_cpu_usage_seconds_total{namespace="payments", container="api"}
```

| Do | Avoid | Why |
|---|---|---|
| Start with the metric name | `{namespace="payments"}` | Selects every metric in the namespace. |
| `namespace="payments"` or `namespace=~"payments\|billing"` | `namespace=~"pay.*"` | Equality and pure literal alternations are direct index lookups. Any other regex is tested against every value of the label. |
| `pod=~"api-.*"` | `pod=~".*api.*"` | Regexes are fully anchored. A literal prefix makes each value check cheaper, but every value is still scanned. |
| `job="kubelet"` when names collide | Bare metric name | Same metric can come from several jobs, doubling series. |
| `{__name__=~"kube_pod_.*"}` only for exploration | In dashboards | Matches across every metric name. |

## Counters: `rate` before `sum`

```promql
# right: per-series rate, then aggregate
sum by (namespace) (rate(http_requests_total[5m]))

# wrong: summing raw counters hides resets; rate of a sum is meaningless
rate(sum by (namespace) (http_requests_total)[5m:])
```

| Function | Use for |
|---|---|
| `rate(x[r])` | Per-second average over the range. Dashboards and alerts. |
| `increase(x[r])` | Total count over the range. "Restarts in the last hour". |
| `irate(x[r])` | Last two samples only. Spiky; zoomed-in graphs, never alerts. |

**The range should cover at least 4 scrape intervals.** `rate` needs two
samples, and in 3.x ranges exclude the left boundary, so with 30s scrapes
`[1m]` holds at most two and one missed scrape empties it. In Grafana use
`$__rate_interval`, defined as `max($__interval + scrape_interval, 4 *
scrape_interval)`.

For stat panels on an Instant query, use `increase(x[$__range])` to get the
total over the dashboard's time range.

## Aggregate early, keep few labels

Every label left on the result is a series Grafana must draw and Prometheus
must return.

```promql
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[$__rate_interval]))
```

- `sum by (a, b)` keeps only what you name. Good for panels.
- `sum without (instance)` keeps everything else. The Prometheus docs
  recommend `without` in recording rules so `job` and similar labels
  survive; in panels it surprises you when a new label appears.
- `topk(10, ...)` still computes every series; it only trims the output. In
  a range query it picks the top 10 independently at each step, so the graph
  can show more than 10 lines.
- `count(up)` is cheap. `count by (__name__)({__name__=~".+"})` is not; use
  the TSDB status page instead (below).

## Histograms

```promql
histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket{namespace="payments"}[$__rate_interval]))
)
```

- Always `sum by (le, ...)`. Drop `le` and you get no result: series without
  it are silently ignored. Keep `pod` and you get one quantile per pod.
- Each bucket is a series, plus `_sum` and `_count`. A histogram with 12
  buckets on 200 pods is 2,800 series per label combination.
- Native histograms store all buckets in one series. Stable since 3.8: set
  `scrape_native_histograms: true` globally or per scrape config. Before 3.8
  it was `--enable-feature=native-histograms`, a no-op from 3.9. Query the
  base metric name with no `_bucket` and no `le`:

```promql
histogram_quantile(0.95, sum by (service) (rate(http_request_duration_seconds[5m])))
```

## Joins

Used to attach Kubernetes metadata (owner, node, labels) from
kube-state-metrics.

```promql
# CPU by owner (ReplicaSet, StatefulSet, DaemonSet, Job)
sum by (namespace, owner_kind, owner_name) (
  rate(container_cpu_usage_seconds_total{container!=""}[$__rate_interval])
  * on (namespace, pod) group_left (owner_kind, owner_name)
  max by (namespace, pod, owner_kind, owner_name) (kube_pod_owner)
)
```

- Always say `on (...)`. Default matching uses all labels and usually
  matches nothing.
- The `max by` on the right side dedupes. With two kube-state-metrics
  replicas every `kube_pod_owner` series exists twice and the join fails
  with `found duplicate series`.
- Aggregate both sides down before joining when you can; the join cost is
  the product of series on each side.
- `kube_pod_owner` points to the ReplicaSet, not the Deployment. To get the
  Deployment, join through `kube_replicaset_owner{owner_kind="Deployment"}`,
  or use the kube-prometheus-stack rule
  `namespace_workload_pod:kube_pod_owner:relabel`, which already does it.
  Stripping the hash from the ReplicaSet name with `label_replace` works
  but is fragile.

## Subqueries

`max_over_time(rate(x[5m])[1d:1m])` evaluates `rate()` at every 1m step
across the panel range plus 1d. On a 7-day panel that's (7d + 1d) / 1m ≈
11,500 evaluations over raw samples, repeated on every refresh. Replace with
a recording rule for `rate(x[5m])` and run `max_over_time` over that.

## Kubernetes gotchas

| Metric | Trap | Fix |
|---|---|---|
| `container_cpu_usage_seconds_total`, `container_memory_*` | cAdvisor also exports pod-level cgroup series with `container=""`. Summing everything double-counts. | `{container!=""}`. kubernetes-mixin uses `image!=""` for the same effect. Dockershim-era clusters also export `container="POD"` for the pause container; add `container!="POD"` there. |
| `container_memory_usage_bytes` | Includes page cache. Not what the OOM killer looks at. | `container_memory_working_set_bytes` |
| `kube_pod_container_status_restarts_total` | Counter; raw value is lifetime restarts. | `increase(...[1h])` |
| `kube_pod_status_phase` | One series per phase, value 0 or 1. | `sum by (phase) (kube_pod_status_phase{namespace="x"} == 1)` |
| Any `pod` label | Every rollout creates new series, and a long range touches all of them. cAdvisor series also linger up to 5 min after the pod is gone, because they carry explicit timestamps and skip staleness markers. | Aggregate to `namespace` / workload for long ranges. |

CPU throttling ratio, a common one to get wrong:

```promql
sum by (namespace, pod) (rate(container_cpu_cfs_throttled_periods_total{container!=""}[$__rate_interval]))
/
sum by (namespace, pod) (rate(container_cpu_cfs_periods_total{container!=""}[$__rate_interval]))
```

## Cardinality

High cardinality is the usual root cause of slow queries and Prometheus OOMs.

| Label | Verdict |
|---|---|
| `namespace`, `container`, `job`, `service` | Good. |
| `pod`, `instance` | Acceptable, but churns with every rollout. |
| `path` with IDs (`/users/8812`), `user_id`, `trace_id`, `error_message` | Never. Fix in the exporter or drop with `metric_relabel_configs`. |

Find the worst offenders:

- Prometheus UI: **Status → TSDB Status** lists top metrics and labels by series count.
- `curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'`
- `promtool tsdb analyze /prometheus` on the data directory.

Drop a bad label at scrape time:

```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: request_id
```

If that label was the only thing telling two series apart, they collide
after the drop and the scrape reports duplicate samples. Fix the exporter
instead when that happens.

## In Grafana

| Setting | Where | Effect |
|---|---|---|
| Time range | Picker | Samples loaded scale linearly with it. |
| `$__rate_interval` | Inside `rate()` | Correct range for any zoom level. Set the data source's scrape interval to match the real one. |
| Min interval (panel) / Min step (query) | Query options | Raise on long-range panels. Fewer steps, fewer evaluations. |
| Max data points | Query options | Caps steps per series. Default is panel width in pixels. |
| Type: Instant | Query editor options | Stat, gauge, table panels. One evaluation instead of hundreds. |

**Query inspector → Stats** shows series returned and query time. The
Prometheus API also returns sample counts with `stats=all`:

```bash
curl -s 'localhost:9090/api/v1/query?query=sum(rate(http_requests_total[5m]))&stats=all' | jq .data.stats
```

### Dashboard variables

`label_values(pod)` returns pod names from every metric in every namespace.
Scope it to one metric and the parent variable:

```
label_values(kube_pod_info{namespace="$namespace"}, pod)
```

Chain `cluster` → `namespace` → `workload` → `pod`. "On dashboard load"
refresh is cheapest; pod lists change over time, so pod variables usually
need "On time range change".

## Recording rules

Any expression used by more than one panel, used by alerts, or slow over a
long range should be a recording rule. Name them `level:metric:operations`:
`_total` is dropped when the operation is `rate`, and `sum` is omitted when
there are other operations.

```yaml
groups:
  - name: k8s-cpu
    interval: 1m
    rules:
      - record: namespace_pod:container_cpu_usage_seconds:rate5m
        expr: |
          sum by (namespace, pod) (
            rate(container_cpu_usage_seconds_total{container!=""}[5m])
          )
      - record: namespace:container_cpu_usage_seconds:rate5m
        expr: sum by (namespace) (namespace_pod:container_cpu_usage_seconds:rate5m)
```

kube-prometheus-stack already ships many of these from kubernetes-mixin
(`node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m`,
`namespace_workload_pod:kube_pod_owner:relabel`). Check
`kubectl get prometheusrules -A` before writing your own.

## Server-side settings

For whoever runs Prometheus. Values shown are the defaults.

```bash
prometheus \
  --query.timeout=2m \
  --query.max-concurrency=20 \
  --query.max-samples=50000000 \
  --query.lookback-delta=5m
```

Long-range dashboards over a single Prometheus will stay slow however good
the query is. Thanos, Mimir and Cortex add a query frontend that splits
queries by day and caches results; Thanos can also downsample old data in
its compactor. That's the structural fix.

## Error messages and what they mean

| Error | Cause | Fix |
|---|---|---|
| `query processing would load too many samples into memory` | Over `--query.max-samples` | Narrow selector, shorter range, bigger step, or a recording rule. |
| `exceeded maximum resolution of 11,000 points per timeseries` | Range / step too fine | Raise min step, or let Grafana pick it. |
| `query timed out in expression evaluation` | Over `--query.timeout` | Same as above. Check for subqueries. |
| `multiple matches for labels: many-to-one matching must be explicit (group_left/group_right)` | Many side has more than one series per `on` key | Add `group_left` / `group_right`. |
| `multiple matches for labels: grouping labels must ensure unique matches` | Output of a `group_left` join isn't unique | Add labels to `on`, or aggregate a side first. |
| `found duplicate series for the match group ... many-to-many matching not allowed` | The "one" side isn't unique on the `on` labels | Aggregate it with `max by (...)`. |
| `vector cannot contain metrics with the same labelset` | Function dropped `__name__` and two series collided | `sum by` first, or `label_replace` to keep them apart. |
| Empty graph with gaps | `rate` range shorter than 2 scrapes | `$__rate_interval`. |

## Checklist

1. Selector has a metric name and `namespace`.
2. Equality matchers wherever a regex isn't needed.
3. `rate` / `increase` inside, `sum by (...)` outside.
4. Range is `$__rate_interval`, not a hardcoded `[1m]`.
5. cAdvisor queries exclude `container=""`.
6. Histograms aggregate `by (le, ...)`.
7. Anything slow or reused is a recording rule.
8. Stat and table panels use Type: Instant.

## References

Prometheus:

- [Querying basics: selectors, staleness](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Operators: aggregation, vector matching](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [Functions: `rate`, `histogram_quantile`](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [HTTP API: `stats`, TSDB status](https://prometheus.io/docs/prometheus/latest/querying/api/)
- [Configuration: `metric_relabel_configs`, `scrape_native_histograms`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [`prometheus` command-line flags](https://prometheus.io/docs/prometheus/latest/command-line/prometheus/)
- [`promtool tsdb analyze`](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)
- [Recording rule naming](https://prometheus.io/docs/practices/rules/)
- [Histograms and summaries](https://prometheus.io/docs/practices/histograms/)
- [Instrumentation: do not overuse labels](https://prometheus.io/docs/practices/instrumentation/#do-not-overuse-labels)
- [CHANGELOG: native histograms stable in 3.8, regex set-matcher speedups](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md)

Grafana:

- [Prometheus query editor: Type, Min step, Format](https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/)
- [Prometheus template variables: `label_values`, `$__rate_interval`](https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/)
- [Mimir query-frontend: splitting and results cache](https://grafana.com/docs/mimir/latest/references/architecture/components/query-frontend/)

Kubernetes:

- [kube-state-metrics metric docs: `kube_pod_owner`, `kube_replicaset_owner`](https://github.com/kubernetes/kube-state-metrics/tree/main/docs)
- [kubernetes-mixin recording rules (`rules/apps.libsonnet`)](https://github.com/kubernetes-monitoring/kubernetes-mixin/blob/master/rules/apps.libsonnet)
- [Thanos compactor: downsampling](https://thanos.io/tip/components/compact.md/)
