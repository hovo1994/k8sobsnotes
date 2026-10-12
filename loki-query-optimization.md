# Loki query optimization for Kubernetes logs

Quick reference for writing LogQL that returns fast in Grafana based on Loki 3.x docs and source.

**Loki indexes labels, not log content.** Every query is: use labels to pick
streams, then read and scan the chunks of those streams line by line. Query
cost is roughly bytes scanned. Everything below is a way to scan fewer bytes.

## Pipeline order

Write stages in this order. Each one shrinks the input for the next.

```logql
{namespace="payments", container="api"}   # 1. stream selector: picks chunks
  | detected_level="error"                 # 2. structured metadata filter
  |= "timeout"                             # 3. line filter: cheap substring scan
  != "healthz"
  | json msg, status                       # 4. parser: extract only what you need
  | status >= 500                          # 5. label filter on parsed fields
  | line_format "{{.msg}}"                 # 6. formatting: last
```

| Stage | Cost | Rule |
|---|---|---|
| Stream selector | Index lookup | Use the most specific label you have. |
| Structured metadata filter | Compare per line, no parsing | Before any parser. Required for bloom query acceleration. |
| Line filter `\|=` `!=` | Substring scan | Before any parser. Drops lines before parsing. |
| Regex filter `\|~` `!~` | Regex per line | Use only when `\|=` can't express it. |
| Parser `json` `logfmt` `pattern` `regexp` | Parse per line | Name the fields: `\| json msg, status`. |
| Label filter `\| status >= 500` | Compare per line | After a parser, ideally after a line filter too. |
| `line_format`, `label_format` | Template per line | Last. Display only. |

The common mistake is `{namespace="x"} | json | level="error"`. That parses
every line in the namespace. `| detected_level="error"` or `|= "error"`
before `| json` skips most of them without parsing.

## Stream selectors

| Do | Avoid | Why |
|---|---|---|
| `{app="checkout-api"}` | `{namespace=~".+"}` | Matches every stream in the tenant. |
| `{namespace="payments", container="api"}` | `{container="api"}` alone | Generic container names repeat across namespaces. |
| `namespace="a"` or `namespace=~"a\|b"` | `namespace=~"pay.*"` | Equality is cheapest. |
| Labels set by the collector | `\| json \| namespace="x"` | Parsed fields aren't indexed; it scans everything first. |

Adding a broader label to a specific one buys nothing: if `app` is unique,
`namespace` alongside it doesn't narrow the search. Add it when the name
isn't unique.

At least one matcher must be non-empty-matching; Loki rejects `{app=~".*"}`.

## Labels and cardinality

Each unique label set is a stream. Too many streams means a large index,
tiny chunks and slow queries for everyone on the tenant.

| Label | Verdict |
|---|---|
| `cluster`, `namespace`, `container`, `app`, `job` | Good. Low cardinality, used in every query. |
| `pod` | Discouraged. Every rollout creates new streams. Send it as structured metadata; `\| pod="x"` and `sum by (pod)` still work. |
| `level` | Avoid as a label. Loki 3.x fills `detected_level` structured metadata automatically (`discover_log_levels`, on by default). |
| `trace_id`, `user_id`, `request_id`, IPs, URLs | Never a label. Unbounded. |

**Structured metadata** is stored with the line, not in the index, and
filters like a label. It's on by default in 3.x but needs schema v13 and the
`tsdb` index. In Alloy or the OpenTelemetry Collector, map `k8s.pod.name`
and similar fields to structured metadata rather than labels.

```logql
{namespace="payments"} | trace_id="4bf92f3577b34da6"
```

If a field is both a stream label and parsed from the line, the parsed copy
is renamed `<name>_extracted`, and `| level="error"` silently filters the
stream label instead. Another reason not to make `level` a label.

Check stream counts per label:

```bash
logcli series '{namespace="payments"}' --since=1h --analyze-labels
```

## Parsers

| Parser | Use for | Note |
|---|---|---|
| `json` | JSON logs | `\| json a, b` extracts two fields; bare `\| json` extracts all. |
| `logfmt` | `key=value` logs | `\| logfmt a, b` likewise. |
| `pattern` | Fixed-layout lines (nginx, apache) | Faster than `regexp`. |
| `regexp` | Anything else | Slowest. Last resort. |
| `unpack` | Lines packed by Alloy `stage.pack` | |

```logql
{app="ingress-nginx"}
  | pattern `<ip> - - <_> "<method> <uri> <_>" <status> <size> <_>`
  | status >= 500
```

Drop parsed fields you don't need with `| drop` or keep a few with `| keep`.
Fewer labels on the output means fewer series in metric queries.

## Metric queries

Grafana dashboards should use metric queries, not raw log panels, wherever a
number will do.

```logql
# error rate per app, one series per app
sum by (app) (
  rate({namespace="payments"} |= "error" [$__auto])
)

# p95 latency from a parsed field
quantile_over_time(0.95,
  {app="api"} | json duration_ms | unwrap duration_ms | __error__="" [$__auto]
) by (namespace)

# top 10 noisiest pods by bytes (run as Instant)
topk(10, sum by (pod) (bytes_over_time({namespace="payments"}[$__auto])))
```

- Always aggregate with `sum by (...)`. Without it, every stream, every
  parsed label and every structured metadata field becomes its own series,
  and you hit `max_query_series`.
- After `unwrap`, add `| __error__=""` to drop lines that failed to parse.
  Without it, one non-JSON line fails the whole query. It must come after
  the `unwrap`, not before.
- Use `$__auto` for the range. It's the step on range queries and the whole
  time range on instant queries.
- `topk` in a range query picks the top 10 independently at each step, so
  the graph can show more than 10 series. Use the Instant type for "top N
  over this period".

## In Grafana

| Setting | Where | Effect |
|---|---|---|
| Time range | Picker | Biggest single lever. 1h scans ~1/24 of 1d. |
| Line limit | Query options | Default 1000. Lower for log panels you only skim. |
| Query type: Instant | Query options | Stat, gauge, table panels. One evaluation instead of one per step. Metric queries only. |
| Min interval | Query options | Raise to cut the number of steps on long ranges. |
| Logs volume | Explore | Runs an extra metric query over the full range on every search. |

**Query inspector → Stats** shows timing, and for Loki usually a "total
bytes processed" summary. Check it before and after a change.

```bash
logcli query --stats --since=1h '{namespace="payments"} |= "error"'
```

### Dashboard variables

An unscoped "Label values" variable for `app` returns every value in the
tenant over the dashboard time range. Scope it with the stream selector
field:

| Field | Value |
|---|---|
| Query type | Label values |
| Label | `app` |
| Stream selector | `{namespace="$namespace"}` |

Older dashboards write the same thing as
`label_values({namespace="$namespace"}, app)`.

Chain variables so each one narrows the next: `cluster` → `namespace` → `app`.

### Recording rules

A dashboard panel that re-runs the same expensive metric query every 30s
should be a recording rule. The Loki ruler evaluates it on a schedule and
remote-writes the result to Prometheus/Mimir; the panel then queries
PromQL, which is cheap.

```yaml
groups:
  - name: payments
    interval: 1m
    rules:
      - record: namespace_app:log_errors:rate1m
        expr: sum by (namespace, app) (rate({namespace="payments"} |= "error" [1m]))
```

The ruler needs remote write enabled, and the receiving Prometheus needs
`--web.enable-remote-write-receiver`:

```yaml
ruler:
  remote_write:
    enabled: true
    clients:
      prometheus:
        url: http://prometheus:9090/api/v1/write
```

## Server-side settings

For whoever runs Loki. Values are starting points, not tuned numbers.

```yaml
server:
  http_server_read_timeout: 5m       # default 30s; caps query_timeout
  http_server_write_timeout: 5m      # default 30s

limits_config:
  query_timeout: 5m                  # default 1m
  split_queries_by_interval: 1h      # default; frontend splits into parallel sub-queries
  max_query_series: 500              # default; raise cautiously
  max_entries_limit_per_query: 5000  # default
  max_query_length: 721h             # default
  max_query_bytes_read: 0B           # default unlimited; set to cap runaway queries

query_range:
  cache_results: true                # default false
```

Raising `query_timeout` alone does nothing past 30s; the HTTP server
timeouts cut the request first. The Grafana data source timeout and any
proxy or load balancer in between must also allow 5m.

Shorter `split_queries_by_interval` means more parallelism but more
per-query overhead. The results cache only helps repeated queries over the
same window, which is what dashboards do. With no cache backend configured,
Loki turns on an in-process embedded cache automatically.

## Error messages and what they mean

| Error | Cause | Fix |
|---|---|---|
| `maximum number of series (500) reached for a single query` (≤3.5: `maximum of series (500) reached`) | Metric query without enough aggregation | Add `sum by (...)`, or `\| drop` parsed labels and structured metadata. |
| `max entries limit per query exceeded` | Line limit above server cap | Lower the line limit, narrow the range. |
| `request timed out, decrease the duration of the request or add more label matchers` / `context deadline exceeded` / 504 | Too many bytes scanned, or HTTP timeouts below `query_timeout` | Narrow selector, add line filter, shorten range. Check server timeouts. |
| `would read too many bytes` | Over `max_query_bytes_read` | Narrow selector, shorten range. |
| `the query time range exceeds the limit` | Over `max_query_length` | Shorter range, or a recording rule. |
| `queries require at least one regexp or equality matcher that does not have an empty-compatible value` | `{app=~".*"}` | Add a real matcher. |
| `Log queries not supported as instant query type` | Instant type on a log query | Switch to Range, or wrap in a metric query. |
| `__error__="JSONParserErr"` in results | Non-JSON lines hit `\| json` | Line-filter first, or `\| __error__=""`. |

## Checklist

1. Selector uses the most specific label, plus `namespace` if that label isn't unique.
2. Structured metadata filters and `|=` line filters come before any parser.
3. Parser names the fields it extracts.
4. Metric queries have `sum by (...)`; `unwrap` is followed by `| __error__=""`.
5. Time range is the shortest that answers the question.
6. No `pod`, `level` or unbounded fields as stream labels.
7. Query inspector shows bytes processed dropped after the change.

## References

Loki:

- [Query best practices](https://grafana.com/docs/loki/latest/query/bp-query/)
- [Log queries: line filters, parsers, `_extracted` suffix](https://grafana.com/docs/loki/latest/query/log_queries/)
- [Metric queries: `unwrap`, range aggregations](https://grafana.com/docs/loki/latest/query/metric_queries/)
- [LogQL reference: pipeline errors in metric queries](https://grafana.com/docs/loki/latest/query/query_reference/)
- [Query acceleration (blooms)](https://grafana.com/docs/loki/latest/query/query_acceleration/)
- [Troubleshoot queries: error messages and timeouts](https://grafana.com/docs/loki/latest/query/troubleshoot-query/)
- [Labels overview: pod names as structured metadata](https://grafana.com/docs/loki/latest/get-started/labels/)
- [Label best practices](https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/)
- [Cardinality](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/)
- [Structured metadata](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/)
- [Configuration reference: `limits_config`, `query_range`, `server`](https://grafana.com/docs/loki/latest/configure/)
- [Recording rules](https://grafana.com/docs/loki/latest/operations/recording-rules/)
- [Ruler and remote write](https://grafana.com/docs/loki/latest/alert/)
- [logcli](https://grafana.com/docs/loki/latest/query/logcli/)

Grafana:

- [Loki query editor: query type, line limit](https://grafana.com/docs/grafana/latest/datasources/loki/query-editor/)
- [Loki template variables: label values, `$__auto`](https://grafana.com/docs/grafana/latest/datasources/loki/template-variables/)
