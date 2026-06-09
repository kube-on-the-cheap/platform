# Grafana Cloud cardinality optimization and k8s-monitoring chart upgrade

Date: 2026-06-09
Status: Draft - pending user review

## Problem

Grafana Cloud ingestion from the `freeloader` and `understairs` clusters is
exhausting the 20,000 active-series free-tier limit. Alloy has been disabled
in the meantime, so there is no fresh cardinality data to target. The goal is
to resume ingestion with a deliberately small baseline, return to and remain
within the free tier, and retain enough visibility to detect that something
is broken in either cluster.

The currently pinned chart version (`grafana/k8s-monitoring ~2.0`) is two
majors behind the latest (`4.1.4`). Both `2 -> 3` and `3 -> 4` introduced
breaking schema changes. Applying cardinality cuts against the v2 schema
would be wasted work; the upgrade is folded into the same effort.

## Goals

- Restore metric and log ingestion to Grafana Cloud from both clusters,
  staying under 20k active series with comfortable headroom (target: 12k).
- Upgrade `grafana/k8s-monitoring` from `~2.0` to `~4.1` with behavior
  preserved during the upgrade step.
- Keep node and workload health visibility plus a minimal control-plane
  signal (apiserver request rate, error rate, latency without histograms).
- Maintain the opt-in annotation contract (`prometheus.io/scrape=true`)
  as the discovery mechanism for application metrics.
- Trim log volume by excluding the noisiest namespaces and dropping INFO
  noise from known-chatty workloads.

## Non-goals

- Adding profiling (Pyroscope), application observability (OTLP receiver),
  or auto-instrumentation (Beyla). The chart supports these in v4; they are
  out of scope for the budget recovery effort.
- Moving off the Grafana Cloud free tier. The 20k series ceiling is a fixed
  constraint of this spec.
- Per-app dashboard tuning. Once the baseline stabilizes, individual
  dashboards that need additional series can be addressed separately.
- Cluster-side Prometheus retention. All metrics continue to ship to
  Grafana Cloud; no local TSDB is introduced.

## Approach

Two phases delivered in sequence, both within the existing
`apps/grafana/overlays/{freeloader,understairs}` Kustomize layout. No new
shared base values file; per-cluster overlays remain the single source of
truth for what each cluster ships.

### Phase 0 - chart upgrade `~2.0` -> `~4.1`, behavior preserved

The v4 schema differs significantly from v2. The intent of Phase 0 is to
land the schema migration with telemetry equivalent to the v2 state, so
the Phase 1 diff is small and the cause of any regression is unambiguous.

Mapping applied to the current configs:

| v2 key                                       | v4 key                                                                 |
| -------------------------------------------- | ---------------------------------------------------------------------- |
| `destinations:` (array)                      | `destinations:` (map keyed by name)                                    |
| `destinations[N].auth.password` (Flux path)  | `destinations.<name>.auth.password`                                    |
| `clusterMetrics.node-exporter`               | `hostMetrics.linuxHosts`                                               |
| `clusterMetrics.kube-state-metrics`          | unchanged                                                              |
| `clusterMetrics.cadvisor`, `.kubelet`        | unchanged                                                              |
| `annotationAutodiscovery:` (top-level)       | unchanged; requires `collector: alloy-metrics`                         |
| `prometheusOperatorObjects.crds.deploy: true`| removed; CRDs installed via separate chart (see below)                 |
| `clusterEvents:`                             | unchanged; requires `collector: alloy-singleton`                       |
| `podLogs:`                                   | `podLogsViaLoki:`; requires `collector: alloy-daemonset`               |
| `nodeLogs:` (freeloader)                     | unchanged; requires `collector: alloy-daemonset`                       |
| `alloy-metrics:` (top-level)                 | `collectors.alloy-metrics: { presets: [clustered, statefulset], ... }` |
| `alloy-logs:` (top-level)                    | `collectors.alloy-daemonset: { presets: [daemonset, ...], ... }`       |
| `alloy-singleton:` (top-level)               | `collectors.alloy-singleton: { presets: [singleton], extraConfig, extraPorts }` |
| `integrations.alloy:`                        | preserved via the v4 equivalent; verified during implementation        |

Phase 0 does not change which metrics are kept or how often they are
scraped. `kube-state-metrics` stays on `useDefaultAllowList: false`,
`node-exporter` stays on `useIntegrationAllowList: true`, scrape interval
remains 90s. The cardinality cuts and the interval bump to 120s land in
Phase 1.

#### Prometheus Operator CRDs

v4 unbundles the Prometheus Operator CRDs. The v2 chart already installed
them into both clusters, so removing the bundling without a replacement
would orphan the existing CRDs and likely trigger Flux to prune them.

Mitigation: install
`prometheus-community/prometheus-operator-crds` (CRD-only Helm chart) as a
new app under `apps/prometheus-operator-crds` **before** bumping the
k8s-monitoring chart. The chart contains only the CRDs and adopts the
existing resources by name; if Helm refuses to adopt unmanaged CRDs,
`--take-ownership` (Helm 3.14+) is the documented escape hatch.

#### Flux `valuesFrom` indexing

`apps/grafana/base/helmrelease.yaml` references
`destinations[0].auth.password` and `destinations[1].auth.password`. The
v4 map-keyed schema requires these paths to become
`destinations.grafana-cloud-metrics.auth.password` and
`destinations.grafana-cloud-logs.auth.password`. The names match the
existing values, so no Secret rename is needed.

### Phase 1 - cardinality cuts and log trims

Applied on top of the v4 schema. Same baseline on both clusters except
where cluster-specific.

#### Metric cuts

| Source                              | Action                                                                                                                                          | Rationale                                                                            |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `clusterMetrics.kube-state-metrics` | Flip to `useDefaultAllowList: true`                                                                                                             | Curated subset for the Grafana Cloud Kubernetes integration. Biggest single win.     |
| `clusterMetrics.cadvisor`           | `useDefaultAllowList: true`, then `excludeMetrics: ["container_blkio_.*", "container_tasks_state", "container_memory_failures_total", "container_fs_io_.*"]` | Drops disk-IO histograms and per-container state counters.                          |
| `clusterMetrics.kubelet`            | `useDefaultAllowList: true`                                                                                                                     | Already curated; no extra includes.                                                  |
| `hostMetrics.linuxHosts`            | `useIntegrationAllowList: true` (unchanged)                                                                                                     | Already optimal.                                                                     |
| `clusterMetrics.apiserver`          | Enable with `useDefaultAllowList: true`, then `excludeMetrics: ["apiserver_request_duration_seconds_bucket", "apiserver_response_sizes_bucket", "apiserver_watch_events_sizes_bucket"]` | Keep request and error rate, drop bucket histograms.                                |
| `clusterMetrics.controlPlane`       | Disabled                                                                                                                                        | Not exposed on Talos by default.                                                     |
| `annotationAutodiscovery`           | Enabled, opt-in only (`prometheus.io/scrape=true`)                                                                                              | Preserves the annotation contract as the per-app control mechanism.                  |
| `prometheusOperatorObjects`         | Enabled; `serviceMonitors.excludeNamespaces: [kube-system]` and global `metricsTuning.excludeMetrics: [".*_bucket", ".*_sum"]`                  | Histograms emitted by app-installed ServiceMonitors are the worst remaining sources. |
| `integrations.alloy` self-monitoring| Keep current exclude lists                                                                                                                      | No-op.                                                                               |

#### Global tuning

- `global.scrapeInterval: 120s` (up from 90s). Kubernetes state changes
  slowly enough; halving sample frequency halves DPM on every metric.

#### Log trims (`podLogsViaLoki`)

- `excludeNamespaces: [kube-system, flux-system, velero]`.
- Keep app namespaces: `grafana`, `cert-manager`, `external-dns`,
  `external-secrets`, `argo-cd`, `envoy-gateway`, `cnpg-system`,
  `tailscale`, `longhorn-system`, `podinfo`, `retina`, `openebs`,
  `default`.
- Drop INFO lines from the two chattiest workloads via
  `extraLogProcessingStages` on the `alloy-daemonset` collector:
  - `{namespace="grafana",app="alloy"}` lines matching `level=info`
  - `{namespace="argo-cd",app="argocd-repo-server"}` lines matching
    `level=info`

#### `nodeLogs` on freeloader

Kept enabled (OCI host journal remains useful). Add an equivalent
`stage.match` to drop kubelet INFO lines.

#### Cluster-specific

- **understairs**: `alloy-singleton` keeps the syslog receiver
  (`extraPorts` + `extraConfig` under `collectors.alloy-singleton`). The
  OpenWRT `prometheus.scrape "openwrt"` block stays unchanged - three
  targets at 30s, low-hundreds of series.
- **freeloader**: `nodeLogs.enabled: true` stays. No syslog.

## Expected outcome

Rough estimate of active series with Phase 1 applied:

| Source                                         | Per cluster | Both clusters |
| ---------------------------------------------- | ----------- | ------------- |
| kube-state-metrics (default allowlist)         | ~1.5k       | ~3k           |
| cAdvisor (allowlist minus excluded histograms) | ~1.5k       | ~3k           |
| kubelet                                        | ~400        | ~800          |
| node-exporter (integration allowlist)          | ~400        | ~800          |
| apiserver (minus bucket histograms)            | ~600        | ~1.2k         |
| Annotation-discovered apps                     | -           | ~1.5k         |
| ServiceMonitor-discovered                      | -           | ~1k           |
| OpenWRT (understairs only)                     | -           | ~150          |
| **Total estimate**                             |             | **~11-12k**   |

Headroom of ~8k series under the 20k ceiling.

## Rollout

1. **Standalone Prometheus Operator CRDs** as a new app on both clusters.
2. **Phase 0** on `freeloader` first (smaller blast radius, no syslog
   dependency); observe for 24 hours; then `understairs`.
3. **Phase 1** as a single PR layering metric and log cuts onto both
   overlays.

## Verification

### Pre-merge

- Render each overlay locally with
  `helm template grafana/k8s-monitoring --version 4.1.4 -f <merged-values>`
  and diff against the v2-rendered output to confirm Phase 0 produces an
  equivalent Alloy configuration.
- For Phase 1, inspect the rendered Alloy relabel and drop rules.

### Post-deploy (Phase 0)

- `kubectl -n grafana get pods` - collectors reconcile without restart
  loops.
- Alloy `/metrics` endpoint shows expected component counts.
- Existing Grafana Cloud dashboards still resolve their queries.
- `kubectl -n grafana get secret <alloy-rendered-secret>` confirms auth
  passwords were injected via the new map-keyed `valuesFrom` path.

### Post-deploy (Phase 1)

- Grafana Cloud Billing / Usage / Active Series drops to ~12k within one
  hour and stays there.
- Bookmark queries:
  - `count by (__name__)({cluster=~"freeloader|understairs"})`
  - `topk(20, count by (__name__)({cluster=~"freeloader|understairs"}))`
- If active series exceed 18k after 24 hours, identify top families via
  the Cardinality view and iterate.

## Risks and mitigations

| Risk                                                                            | Mitigation                                                                                                                |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Standalone CRDs chart conflicts with existing CRDs installed by the v2 chart    | Use Helm 3.14+ `--take-ownership` on install if Helm refuses to adopt unmanaged CRDs.                                     |
| v4 chart removes an Alloy component currently in use (e.g., syslog receiver)    | Pre-merge dry-render of understairs values catches this; the `collectors.alloy-singleton.extraConfig` path is documented. |
| `destinations[0]` path produces a silent broken Helm release post-migration     | Pre-merge render and post-deploy secret inspection confirm the new map-keyed path renders correctly.                      |
| Phase 1 cuts too aggressive - a dashboard breaks                                | Allowlists are additive; re-add specific metrics via `includeMetrics` rather than reverting `useDefaultAllowList`.        |
| `useDefaultAllowList: true` on kube-state-metrics drops something in use        | Default allowlist matches the stock Grafana Cloud Kubernetes integration dashboards. Worst case: add specific `kube_*` series via `includeMetrics`. |
| OpenWRT scrape block ends up wrongly scoped after singleton refactor            | Verify `up{job="openwrt"}` on understairs post-deploy.                                                                    |

## File-level scope

- `apps/grafana/base/helmrelease.yaml` - chart version bump, `valuesFrom`
  path rewrite.
- `apps/grafana/base/helmrepository.yaml` - unchanged.
- `apps/grafana/overlays/freeloader/alloy-values.yaml` - full rewrite to
  v4 schema plus Phase 1 cuts.
- `apps/grafana/overlays/understairs/alloy-values.yaml` - full rewrite to
  v4 schema plus Phase 1 cuts; syslog receiver moves under
  `collectors.alloy-singleton`.
- `apps/grafana/overlays/understairs/helmrelease.patch.yaml` - reviewed
  during implementation, may need updates if the patch references
  destinations indices.
- `apps/prometheus-operator-crds/` (new) - HelmRelease for the
  CRD-only chart, wired into both cluster Kustomizations.
- `clusters/freeloader/kustomization.yaml`,
  `clusters/understairs/kustomization.yaml` - add the new CRDs app.
