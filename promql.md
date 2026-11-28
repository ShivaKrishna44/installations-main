# PromQL Practice Guide — Expense App

Every query in this guide runs against the live expense app. Each topic explains the concept, gives queries to run with what you should see, and lists the mistakes people make most often. Practice questions with answers are at the end.

## Environment

| Component | Where | Port |
| --- | --- | --- |
| App (public) | `https://expense.daws88s.online` (ALB) | 443 |
| Frontend (nginx + exporter) | frontend VM | 9100, 9113 |
| Backend (Node.js app metrics) | backend VM | 9100, 9101 |
| MySQL (+ mysqld_exporter) | db VM | 9100, 9104 |
| Prometheus | monitoring VM | 9090 |

Each node target carries a `tier` label: `frontend`, `backend`, `db`, `monitoring`. All four VMs are t3.micro with 2 vCPUs. Add `{tier!="monitoring"}` when you want only the app VMs.

Run queries in the Prometheus UI (`http://<monitoring-ip>:9090`) under **Query**. Use **Table** to see current values and labels, **Graph** to see how they change over time.

## Before you start

**Check the metric names.** The names in this guide are the usual ones. Confirm yours once:

```bash
curl -s http://backend.daws88s.online:9101/metrics | grep -E '^# (TYPE|HELP)' | grep http
```

Look for `http_requests_total` (counter), `http_request_duration_seconds` (histogram), the in-flight gauge, and the labels `method`, `route`, `status_code`.

**Make sure real routes appear.** If every series shows `route="unmatched"`, Prometheus is only seeing 404s from internet scanners. Real API traffic shows route templates such as `route="/api/expenses"` and `route="/api/expenses/:id"`. That is why most app queries below filter `route!="unmatched"`.

**Generate traffic.** Rates and percentiles need steady traffic. Run the load test from the infra repo:

```bash
node loadtest/expense-load.mjs
```

It signs up a user, then sends a weighted mix of expense, category, budget, insight and auth requests (see `loadtest/README.md`). Give it about 5 minutes before you rely on `[5m]` windows.

---

## 1. The data model

A **time series** is a metric name plus a unique set of labels. Every different label combination is a separate series.

| Type | What it is | Example question |
| --- | --- | --- |
| Instant vector | One latest value per series | What is it now? |
| Range vector | Many samples per series over a window | What happened in the last minute? |
| Scalar | A plain number, no labels (`100`, `time()`) | Used in calculations; vector × scalar keeps all labels |
| String | Exists, practically never used | — |

```promql
http_requests_total
```
Table view shows many rows: one per method × route × status_code combination.

```promql
count(http_requests_total)
```
The number of **series** (label combinations), not the number of requests. This is how you check cardinality.

```promql
http_requests_total[1m]
```
Each series shows about 4 raw samples with timestamps, because Prometheus scrapes every 15 s.

**Common mistakes**
- Graphing a range vector (`http_requests_total[1m]`) fails with *"expected type instant vector … got range vector"*. Graphs need instant vectors.
- `irate(http_requests_total)` fails with *"expected type range vector in call to function irate"*. Rate functions need at least two samples, so they need a range.

## 2. Label matchers

| Matcher | Meaning |
| --- | --- |
| `=` | Exactly equal |
| `!=` | Not equal |
| `=~` | Matches regex |
| `!~` | Does not match regex |

Regex matches are **fully anchored**: the whole value must match.

```promql
http_requests_total{route="/api/expenses"}
http_requests_total{status_code=~"4.."}
http_requests_total{status_code=~"4..|5.."}
http_requests_total{status_code!~"2.."}
http_requests_total{route!~"/api/auth/.*"}
node_load1{tier="db"}
node_load1{tier!="monitoring"}
{__name__=~"node_memory_Mem.*"}
```

**Common mistakes**
- `{status_code=~"4"}` returns nothing. Anchored regex means "exactly 4", not "starts with 4"; use `"4.."`.
- `{route!~"/api/auth/*"}` still shows auth routes. In regex, `*` repeats the previous character (`/`); the shell's `*` is regex `.*`.
- `node_load1{tier="mysql"}` returns nothing. The label value is `db`. Check real label values before filtering.
- `{job=~".*"}` fails with *"must contain at least one non-empty matcher"*.

## 3. Counters: rate, increase, irate

A counter only goes up, and resets to 0 when the process restarts. Its raw value tells you little; the **speed** at which it grows is what matters.

| Function | Returns |
| --- | --- |
| `rate(x[5m])` | Average per-second increase over 5 minutes |
| `increase(x[5m])` | Total increase over 5 minutes (= rate × 300) |
| `irate(x[5m])` | Per-second rate from the last two samples only; the window only limits how far back to look |

```promql
http_requests_total{route="/api/expenses", method="GET", status_code="200"}
```
A line that only climbs. From this alone you can't tell whether traffic is high or low.

```promql
rate(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
A fairly flat line: requests per second for that route. Compare it with the load test's own numbers.

```promql
increase(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
Requests in the last 5 minutes. It is rarely a whole number, because Prometheus extrapolates to the edges of the window.

```promql
irate(http_requests_total{route="/api/expenses", method="GET", status_code="200"}[5m])
```
Much spikier than `rate`. Use `irate` when zoomed in on a short graph range for debugging; use `rate` for dashboards and alerts.

### Counter resets

When the backend restarts, every counter drops back to near 0. The raw counter graph shows a cliff, but `rate()` carries on almost unchanged: it detects the drop as a reset and treats it as a new start. You can see resets directly:

```promql
resets(http_requests_total{route="/api/expenses"}[15m])
```

A series only appears again after its first request following the restart, so a series that hasn't received traffic yet is missing, not 0.

The old samples are still in Prometheus even though the app forgot them. Look back with `offset` or `@`:

```promql
http_requests_total{route="/api/expenses"} offset 10m
```

**Common mistake:** `rate(http_requests_total[15s])` returns nothing. The window must hold at least two samples. Rule of thumb: at least 4 × the scrape interval, so `[1m]` minimum and `[5m]` as the default.

## 4. Aggregation

Aggregation operators combine many series into fewer. Removing a label means "treat these as one".

| Operator | Answers |
| --- | --- |
| `sum` | Total |
| `avg` | Average |
| `min` / `max` | Smallest / largest |
| `count` | How many series (ignores the values) |
| `topk` / `bottomk` | Top / bottom N, labels kept |

`by (labels)` keeps only those labels; `without (labels)` drops only those. **Always `rate()` first, then `sum`.**

### sum: request totals

```promql
sum(rate(http_requests_total[5m]))                         # total requests per second
sum by (route) (rate(http_requests_total[5m]))             # per route
sum by (status_code) (rate(http_requests_total[5m]))       # per status code
sum by (method, route) (rate(http_requests_total[5m]))
sum without (instance, job) (rate(http_requests_total[5m]))
sum by (route) (increase(http_requests_total[5m]))         # request COUNT per route
topk(3, sum by (route) (rate(http_requests_total[5m])))    # 3 busiest routes
```

With one backend VM, `without (instance, job)` only removes those labels. `sum by (route)` does merge rows, e.g. GET and POST on `/api/expenses`. Note that `without` keeps every other label: with two backend VMs, `instance_id` and `az` would still keep them apart.

### avg, min, max, topk: CPU across VMs

CPU % per VM:

```promql
100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Reading it inside out:
1. `node_cpu_seconds_total{mode="idle"}` is a counter: seconds each core has spent idle since boot.
2. `rate(...[5m])` gives idle seconds per second, i.e. the idle fraction of each core (0 to 1). One row per core.
3. `avg by (tier)` averages the two cores into one value per VM.
4. `* 100` gives idle %; `100 - ...` gives **CPU used %**.

Across all VMs:

```promql
avg(100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))    # average VM
max(100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))    # busiest VM, number only
min(100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))    # least busy VM
topk(1, 100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) # busiest VM with its tier
```

Why the choice matters: if one VM runs at 50% and the other three at 3%, `avg` says about 15% (looks fine), `max` says 50% (something is wrong), and `topk(1, …)` tells you it's `tier="db"`.

| You need | Use |
| --- | --- |
| How bad is the worst? (one number for a stat panel) | `max()` |
| Which VM is the worst? | `topk(1, …)` |
| Worst VM in each group (several VMs per tier) | `max by (tier) (…)` |
| One result for every VM over 80% | The per-VM query `> 80` (labels kept) |

### count

```promql
count(http_requests_total)                                # number of series, not requests
count by (tier) (up{job="node"})                          # targets per tier, up OR down
sum by (tier) (up{job="node"})                            # targets per tier that are UP
count by (tier) (node_cpu_seconds_total{mode="idle"})     # vCPUs per VM (= 2)
```

`count` ignores values: a target that is down still has an `up` series (value 0), so `count by (tier)` still shows 1 while `sum by (tier) (up)` shows 0.

**Common mistakes**
- **Sum before rate.** `rate(sum(http_requests_total)[5m:])` is wrong: once counters are summed, `rate` can't see each series' reset, so a restart shows up as a fake dip or spike. `sum(rate(http_requests_total[5m]))` is right.
- `avg(rate(http_requests_total[5m]))` averages per series, not per request. Use `sum` for request rates.
- `node_load1` is one value per VM (the system load average), not per CPU. Per-core data is in `node_cpu_seconds_total` (`cpu` label).

## 5. Gauges

A gauge goes up and down: free memory, load, requests in flight. Use its value directly, or with `*_over_time` functions. **Never `rate()` a gauge.**

```promql
node_memory_MemAvailable_bytes / 1024 / 1024                                     # MB free
node_load1                                                                       # one value per VM
max_over_time(node_load1[30m])                                                   # peak load in 30 min
avg_over_time(node_load1[30m])
http_requests_in_flight                                                          # use your gauge's name
max_over_time(http_requests_in_flight[10m])
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600)          # free bytes 4 h from now
```

`max(x)` is the largest across series **now**; `max_over_time(x[30m])` is the largest value of **each series over time**.

`node_load1` is the average number of processes running or waiting to run over the last minute. On a 2-vCPU VM, below about 2 means the CPUs are keeping up. Load per CPU (uses `on()`, explained in section 8):

```promql
node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})
```

Above 1 means saturated.

**Common mistake:** `rate(node_memory_MemAvailable_bytes[5m])` returns numbers, but they mean nothing, and every drop is treated as a counter reset. Prometheus won't stop you. The metric name doesn't tell you the type: check `# TYPE` in `/metrics`.

## 6. Binary operators

Arithmetic `+ - * / %`, comparison `> < == !=`, and logical `and or unless`. Between two vectors, Prometheus pairs series with **identical labels**. A vector combined with a scalar keeps all its labels.

```promql
# Memory used % (both sides have the same labels, so they pair up)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Error ratio: 5xx / total
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

On a healthy app the error ratio is **empty**, because no 5xx series exist yet. Section 9 explains why and how to fix it. When MySQL is down, the backend starts returning 5xx, those series appear, and the ratio shows a value.

Comparisons filter by default; `bool` returns 1 or 0 instead:

```promql
sum by (route) (rate(http_requests_total[5m])) > 1          # keeps only routes above 1 req/s
sum by (route) (rate(http_requests_total[5m])) > bool 1     # 1 or 0 for every route
```

## 7. Histogram queries

Histogram buckets are cumulative counts of requests that took `≤ le` seconds. `histogram_quantile` estimates a percentile by interpolating inside the bucket it falls in.

```promql
# p99 overall (use 0.5 / 0.95 for p50 / p95)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# p99 per route
histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))

# Average latency
sum(rate(http_request_duration_seconds_sum[5m]))
/ sum(rate(http_request_duration_seconds_count[5m]))

# % of requests that finished in ≤ 500 ms (exact, no estimation)
sum(rate(http_request_duration_seconds_bucket{le="0.5", route!="unmatched"}[5m]))
/ sum(rate(http_request_duration_seconds_count{route!="unmatched"}[5m])) * 100
```

The last query is exact because it reads a real bucket boundary; it works only for `le` values that exist as buckets. "Percentage of requests faster than X" is how latency SLOs are written.

Under heavier load on a t3.micro, p99 climbs while p50 barely moves: most requests stay fast, and only the slowest ones queue.

**Common mistakes**
- `sum(rate(..._bucket[5m]))` without `by (le)` returns NaN or nothing: `histogram_quantile` needs the `le` label.
- Leaving in `route="unmatched"` 404s makes p99 look better than reality, because those fast 404s pad the distribution.

## 8. USE and RED: reference queries

These are the queries dashboards and alerts are built from.

### USE: per VM (node_exporter)

| Signal | Query |
| --- | --- |
| CPU % | `100 - (avg by (tier) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |
| Disk % | `(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100` |
| Saturation | `node_load1` (2 vCPUs, so above 2 is saturated) |
| Network | `rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8` (bits/s) |

### RED: backend

| Signal | Query |
| --- | --- |
| Rate | `sum by (route) (rate(http_requests_total{route!="unmatched"}[5m]))` |
| Errors % | `(sum(rate(http_requests_total{status_code=~"5.."}[5m])) or vector(0)) / sum(rate(http_requests_total[5m])) * 100` |
| Duration p99 | `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{route!="unmatched"}[5m])))` |

### Edge: outside-in

| Signal | Query |
| --- | --- |
| nginx requests/s | `sum(rate(nginx_http_requests_total[5m]))` |
| Active connections | `nginx_connections_active` |
| Site up (blackbox) | `probe_success` |
| Certificate days left | `(probe_ssl_earliest_cert_expiry - time()) / 86400` |

Inside-out and outside-in signals react differently. With MySQL down: error %, and `mysql_up` move; p99 may even drop, because errors return fast; CPU barely changes; and `probe_success` stays 1 if the probed homepage is served by nginx without the database.
