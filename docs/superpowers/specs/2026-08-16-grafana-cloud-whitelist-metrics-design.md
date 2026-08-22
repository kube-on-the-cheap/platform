# Grafana Cloud whitelist-based metrics collection

Date: 2026-08-16 (updated 2026-08-22 with post-rollout corrections)
Status: Implemented and merged (PR #54)

## Problem

After the Phase 1 cardinality cuts (see
`2026-06-09-grafana-cloud-cardinality-optimization-design.md`), Grafana Cloud
active series across `freeloader` and `understairs` sit near the top of the
tenant ceiling and periodically exceed it. The current configuration relies
on the k8s-monitoring chart's `useDefaultAllowList: true` for
`kube-state-metrics`, `cadvisor`, and `kubelet`. That default list is curated
for the Grafana Cloud Kubernetes integration dashboards - useful, but broader
than this homelab needs. It ships metrics that will never be queried here,
including full apiserver bucket histograms via
`apiserver_request_duration_seconds_count` and similar.

**Tenant limits observed during rollout (2026-08-17):**
- Active series ceiling: **15,000** (not 20,000 as previously assumed - Grafana
  Cloud free-tier limits for this stack are stricter than the public
  documentation implies).
- Ingestion rate: **1,500 samples/s** with a 15,000-sample burst
  (`err-mimir-tenant-max-ingestion-rate`).
- Out-of-order sample window: samples with timestamps older than the ingester
  window are rejected with 400 `err-mimir-sample-timestamp-too-old` - relevant
  after a config change when Alloy replays its WAL.

The goal is to move from "default allowlist minus a few excludes" to an
explicit per-source whitelist that enumerates every metric that ships. Baseline
monitoring and availability are the target - performance profiling is not.

## Goals

- Cut active series to well below the 15k tenant ceiling, with headroom to
  reintroduce a metric or two without approaching the limit. Target: 3-6k
  active series across both clusters combined.
- Express the whitelist in the chart's canonical `metricsTuning` idiom:
  `useDefaultAllowList: false` + `includeMetrics: [...]` per source.
- Preserve visibility into node health, pod state, and monitoring pipeline
  health (Alloy reaching Grafana Cloud, cert expiry, storage free space).
- Keep the inventory of allowed metrics as a first-class artifact in the repo
  so future review, addition, or removal is a diff against a single readable
  file.
- Correct two bugs in the current configuration surfaced during design:
  - `apiServer.metricsTuning.useDefaultAllowList: true` - the chart's
    `apiServer` source has no `useDefaultAllowList` field; this line is a
    no-op and should be removed.
  - `prometheusOperatorObjects` uses `excludeNamespaces: [kube-system]` -
    permissive by default. Flip to positive scoping (or disable, see below).

## Non-goals

- Ingress or gateway request-rate / error-rate metrics. Explicitly deferred;
  can be added later via the `annotationAutodiscovery` contract when needed.
- Application performance metrics (latency histograms, throughput
  distributions). Not part of baseline availability monitoring.
- Log volume changes. The Phase 1 log trims and exclude namespaces stand.
- Moving off the Grafana Cloud free tier.
- OpenWRT scrape reconfiguration on `understairs`. Already tight (3 targets,
  low-hundreds of series), lives in `extraConfig`, out of scope for this pass.
- Alerting rules. This spec defines what is collected; alerting is a separate
  concern.

## Approach

Approach A from brainstorming: per-source `includeMetrics` in each cluster
overlay, no shared Kustomize component. Two-cluster duplication is accepted
over premature abstraction. Both overlays carry the same lists; when a third
cluster shows up, the lists get promoted to a shared component.

The chart's own metrics-tuning examples use exactly this idiom (see
`charts/k8s-monitoring/docs/examples/metrics-tuning/README.md` in
`grafana/k8s-monitoring-helm`), so this is idiomatic and stable across chart
versions in the `~4.x` line.

Alloy self-monitoring gains three remote_write health metrics so silent
pipeline failure is detectable. OpenWRT scrape on `understairs` is unchanged.

## Whitelist inventory

The complete list ships as `apps/grafana/METRICS.md`, generated as part of
this change and kept in sync with the overlays. Summary by source:

### `clusterMetrics.kube-state-metrics`

Pod state and node conditions - the core of "is anything broken":

- `kube_node_info`
- `kube_node_status_condition`
- `kube_node_spec_unschedulable`
- `kube_pod_info`
- `kube_pod_status_phase`
- `kube_pod_status_ready`
- `kube_pod_container_status_restarts_total`
- `kube_pod_container_status_waiting_reason`
- `kube_pod_container_status_terminated_reason`
- `kube_deployment_status_replicas_available`
- `kube_deployment_status_replicas_unavailable`
- `kube_statefulset_status_replicas_ready`
- `kube_daemonset_status_number_unavailable`
- `kube_persistentvolume_status_phase`
- `kube_persistentvolumeclaim_status_phase`

### `clusterMetrics.cadvisor`

Minimum for "is a pod thrashing or eating all the memory". No histograms,
no per-filesystem breakdown:

- `container_cpu_usage_seconds_total`
- `container_memory_working_set_bytes`
- `container_oom_events_total`
- `container_last_seen`

### `clusterMetrics.kubelet`

Kubelet health plus volume free space (the storage signal):

- `kubelet_node_name`
- `kubelet_volume_stats_available_bytes`
- `kubelet_volume_stats_capacity_bytes`

### `clusterMetrics.apiServer`

Enough to notice the apiserver died or is erroring at high rate. No latency
histograms. Chart quirk: `apiServer.metricsTuning` has no `useDefaultAllowList`
field; an empty `includeMetrics` means "keep all", so the list itself is the
whitelist:

- `apiserver_request_total`
- `apiserver_current_inflight_requests`

### `hostMetrics.linuxHosts`

Keep `useIntegrationAllowList: true` - it powers the Grafana Cloud Linux Node
integration dashboard and is already tight. Add `excludeMetrics` for the
noisiest per-device series we don't need:

- `node_network_receive_.*`
- `node_network_transmit_.*`
- `node_disk_io_time_weighted_seconds_total`
- `node_scrape_collector_.*`

Estimated ~200-300 series per host after excludes.

### `annotationAutodiscovery`

Stays enabled - this is the opt-in contract for future app metrics
(`prometheus.io/scrape=true`). No app currently uses it, so cost today is
zero. Add:

```yaml
metricsTuning:
  excludeMetrics: ["go_.*", "process_.*", ".*_bucket", ".*_sum"]
```

### `prometheusOperatorObjects`

Disable (`enabled: false`). No app currently ships a ServiceMonitor or
PodMonitor we want consumed. The contract is: when an app needs one, add
its namespace here explicitly (`namespaces: [<ns>]`) rather than opening
the door globally. Loud failure over silent scraping.

### `integrations.alloy` self-monitoring

Extend the existing `alloy` instance's `includeMetrics` with:

- `prometheus_remote_storage_samples_total`
- `prometheus_remote_storage_samples_failed_total`
- `prometheus_remote_storage_sent_bytes_total`

Detects silent failure to reach Grafana Cloud.

## Expected outcome

| Source                                             | Per cluster | Notes                             |
| -------------------------------------------------- | ----------- | --------------------------------- |
| kube-state-metrics (15 metrics)                    | ~600        | Multiplied by pods/nodes/PVCs     |
| cAdvisor (4 metrics x containers)                  | ~800        | Bulk of cluster series            |
| kubelet (3 metrics)                                | ~50         | Volume stats scale with PVC count |
| apiServer (2 metrics)                              | ~150        | Per verb x resource               |
| hostMetrics.linuxHosts (integration list - excl.)  | ~250        | Per node                          |
| annotationAutodiscovery                            | 0           | No consumers today                |
| prometheusOperatorObjects                          | 0           | Disabled                          |
| Alloy self-monitoring                              | ~30         | Existing + 3 new                  |
| **Per cluster**                                    | **~1,900**  |                                   |
| **Both clusters**                                  | **~3,800**  |                                   |
| **Plus OpenWRT on understairs**                    | **~150**    |                                   |
| **Grand total**                                    | **~4,000**  | 11k headroom under 15k ceiling    |

## Rollout

Single PR, both overlays changed together. The change narrows what already
works; no schema migration, no new charts, no CRDs.

1. Land `apps/grafana/METRICS.md` and the two overlay changes together.
2. Flux reconciles; Alloy collectors reload their configs without restart
   loops.
3. Observe Grafana Cloud active series drop within one hour.

If freeloader misbehaves, revert the single PR - both overlays return to the
Phase 1 state.

## Verification

### Pre-merge

- Render each overlay:
  `helm template grafana/k8s-monitoring --version 4.1.4 -f apps/grafana/overlays/freeloader/alloy-values.yaml`
  and confirm it renders without error.
- Inspect the rendered Alloy config for each source - confirm the drop rule
  is present with the correct name list (the chart implements
  `useDefaultAllowList: false` + `includeMetrics` via a
  `prometheus.relabel` block with `action = "keep"`). Grep for one metric
  that should be kept and one that should be dropped to confirm.
- Diff `apps/grafana/METRICS.md` against the two overlay `includeMetrics`
  lists - they must match exactly.

### Post-deploy (within 1 hour)

- Grafana Cloud -> Billing -> Active Series across both clusters drops to
  ~4-5k. If it stays above 8k, a source is not being narrowed as expected.
- `topk(20, count by (__name__)({cluster=~"freeloader|understairs"}))` -
  the top 20 metric names should all appear in `METRICS.md`.
- `count(count by (__name__)({cluster=~"freeloader|understairs"}))` -
  distinct metric names should be roughly 25-30 (whitelist size + Alloy
  self-monitoring).

### Post-deploy (within 24 hours)

- Walk the Grafana Cloud Kubernetes integration dashboards. Any panel
  showing "No data" indicates a metric we cut that a dashboard needs.
  Two responses:
  1. Panel matters to you - add the metric to the whitelist AND
     `METRICS.md`, submit as a follow-up PR.
  2. Panel does not matter - leave it. That dashboard is not part of
     baseline monitoring for this homelab.
- Bookmark query: `prometheus_remote_storage_samples_failed_total > 0`.
  If this fires, the monitoring pipeline itself is failing and needs
  attention regardless of anything else on the dashboards.

### Success criteria

- Active series between 3k and 6k across both clusters, stable for 24
  hours.
- Node health, pod state, storage, and monitoring-pipeline signals all
  queryable in Grafana Cloud.
- `apps/grafana/METRICS.md` accurately reflects what ships.

## Risks and mitigations

| Risk                                                            | Mitigation                                                                                                       |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| A dashboard panel breaks because its metric is not on the list  | Iterate: identify metric via cardinality view, add it to overlays + `METRICS.md` in a follow-up PR. Non-urgent.  |
| Overlays and `METRICS.md` drift over time                       | Convention: any PR that changes `includeMetrics` in an overlay must also update `METRICS.md`. Add to PR checklist. |
| App later needs a ServiceMonitor consumed but discovery is off  | Documented on `METRICS.md`. Re-enable `prometheusOperatorObjects` with `namespaces: [<the-ns>]`, not globally.   |
| Alloy self-monitoring metrics themselves get dropped by mistake | Explicit `includeMetrics` block on the `alloy` self-monitoring instance covers the three health metrics.         |
| kube-state-metrics list misses a namespace-scoped resource type | Add via `includeMetrics`. The list is a floor, not a ceiling; grow it deliberately.                              |
| Free-tier limit changes                                         | Headroom of ~11k series under the observed 15k ceiling means a limit reduction to 5k would still be safe.        |
| Config-reload WAL replay hits ingestion-rate limit              | Alloy replays its on-disk WAL on config change; large pre-change backlogs (tens of MB) will saturate the 1.5k samples/s limit and produce sustained 429s. Mitigation: `kubectl rollout restart statefulset alloy-alloy-metrics` after the reload settles - drops the stale WAL, cuts cleanly to the new configuration. Cost: brief gap in samples during the pod restart. |

## File-level scope

- `apps/grafana/overlays/freeloader/alloy-values.yaml`
  - Flip `kube-state-metrics`, `cadvisor`, `kubelet` to
    `useDefaultAllowList: false` + explicit `includeMetrics`.
  - Remove the invalid `useDefaultAllowList: true` from `apiServer.metricsTuning`.
  - Populate `apiServer.metricsTuning.includeMetrics`.
  - Replace `hostMetrics.linuxHosts` excludes with the four listed above.
  - Add `annotationAutodiscovery.metricsTuning.excludeMetrics`.
  - Set `prometheusOperatorObjects.enabled: false`; leave the
    `serviceMonitors` / `podMonitors` blocks as commented stubs so the
    re-enable pattern is discoverable.
  - Extend `integrations.alloy` `alloy` instance `includeMetrics` with the
    three remote_write health metrics.
  - Header comment pointing at `apps/grafana/METRICS.md`.
- `apps/grafana/overlays/understairs/alloy-values.yaml`
  - Same changes as freeloader. Syslog receiver and OpenWRT scrape blocks
    under `collectors.alloy-singleton.extraConfig` untouched.
  - Header comment pointing at `apps/grafana/METRICS.md`.
- `apps/grafana/METRICS.md` (new)
  - One table per source with columns: metric, signal (node / pod /
    storage / monitoring-health), rationale.
  - Convention paragraph at the top: "This file is the canonical inventory
    of metrics shipped to Grafana Cloud. Any PR that changes an overlay's
    `includeMetrics` must update this file."
- `apps/grafana/base/` - unchanged.

## Post-rollout notes (2026-08-17)

Merged as PR #54 (rebase, three commits: `1440fbb`, `d9e0bb9`, `6ab71b0`).

**Deploy observations:**

- Flux reconciled cleanly on both clusters; HelmRelease upgraded to
  `k8s-monitoring@4.1.7` without pod restart loops.
- Live Alloy config verified: whitelist keep-rules present, dropped metrics
  absent, `prometheus.operator.*` discovery components correctly torn down.
- Initial reload produced sustained 429s (`err-mimir-tenant-max-ingestion-rate`)
  because Alloy replayed a large pre-change WAL (74 MB on understairs, 48 MB on
  freeloader) at 1,500 samples/s. Also produced 400
  `err-mimir-sample-timestamp-too-old` for buffered samples of metrics we no
  longer collect (e.g. `envoy_cluster_upstream_rq_pending_overflow`,
  `node_network_transmit_bytes_total`).
- Resolved by `kubectl rollout restart statefulset alloy-alloy-metrics` on both
  clusters. Post-restart WAL size dropped to ~4 MB (understairs) and ~2 MB
  (freeloader). Steady state reached within ~50 minutes: zero remote_write
  errors over a 10-minute window on both clusters.

**Lessons folded into this spec:**

- The observed 15k active-series ceiling replaces the previously assumed 20k
  throughout.
- The WAL-flush recipe (`rollout restart`) is now called out in the risks
  table as the standard mitigation for reload-time backpressure.
