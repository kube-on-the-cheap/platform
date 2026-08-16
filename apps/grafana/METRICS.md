# Grafana Cloud metrics inventory

This file is the canonical inventory of metrics shipped to Grafana Cloud from
the `freeloader` and `understairs` clusters. Both cluster overlays under
`apps/grafana/overlays/{cluster}/alloy-values.yaml` express these lists via
`useDefaultAllowList: false` + `includeMetrics: [...]` (or, where the chart
source has no `useDefaultAllowList` field, via `includeMetrics` alone).

**Convention:** any PR that changes an overlay's `includeMetrics` MUST update
this file in the same commit. Any PR that changes this file MUST update the
overlays. The two are kept in sync by hand; there is no generator.

Design spec: `docs/superpowers/specs/2026-08-16-grafana-cloud-whitelist-metrics-design.md`.

## Signal legend

- **node** - node health, resource pressure, kubelet reachability
- **pod** - pod/container state, restarts, waiting/terminated reasons
- **storage** - PV/PVC state, volume free space
- **monitoring-health** - Alloy pipeline health, remote_write success

## `clusterMetrics.kube-state-metrics`

| Metric                                              | Signal  | Rationale                                                     |
| --------------------------------------------------- | ------- | ------------------------------------------------------------- |
| `kube_node_info`                                    | node    | Node inventory join label                                     |
| `kube_node_status_condition`                        | node    | NotReady / DiskPressure / MemoryPressure                      |
| `kube_node_spec_unschedulable`                      | node    | Cordoned nodes                                                |
| `kube_pod_info`                                     | pod     | Pod inventory join label                                      |
| `kube_pod_status_phase`                             | pod     | Pending / Running / Failed / Succeeded / Unknown              |
| `kube_pod_status_ready`                             | pod     | Readiness gate                                                |
| `kube_pod_container_status_restarts_total`          | pod     | Restart loops                                                 |
| `kube_pod_container_status_waiting_reason`          | pod     | CrashLoopBackOff / ImagePullBackOff / etc.                    |
| `kube_pod_container_status_terminated_reason`       | pod     | OOMKilled / Error / Completed                                 |
| `kube_deployment_status_replicas_available`         | pod     | Deployment health                                             |
| `kube_deployment_status_replicas_unavailable`       | pod     | Deployment degradation                                        |
| `kube_statefulset_status_replicas_ready`            | pod     | StatefulSet health                                            |
| `kube_daemonset_status_number_unavailable`          | pod     | DaemonSet degradation                                         |
| `kube_persistentvolume_status_phase`                | storage | PV Bound / Released / Failed                                  |
| `kube_persistentvolumeclaim_status_phase`           | storage | PVC Bound / Pending / Lost                                    |

## `clusterMetrics.cadvisor`

| Metric                              | Signal | Rationale                                    |
| ----------------------------------- | ------ | -------------------------------------------- |
| `container_cpu_usage_seconds_total` | pod    | Per-container CPU rate for hot-loop detection |
| `container_memory_working_set_bytes`| pod    | Working-set memory used                       |
| `container_oom_events_total`        | pod    | OOM kill counter                              |
| `container_last_seen`               | pod    | Container liveness heartbeat                  |

## `clusterMetrics.kubelet`

| Metric                                  | Signal  | Rationale                            |
| --------------------------------------- | ------- | ------------------------------------ |
| `kubelet_node_name`                     | node    | Kubelet up + node identity label     |
| `kubelet_volume_stats_available_bytes`  | storage | PV free space                        |
| `kubelet_volume_stats_capacity_bytes`   | storage | PV capacity for percentage full calc |

## `clusterMetrics.apiServer`

The chart's `apiServer` source has no `useDefaultAllowList` field; empty
`includeMetrics` means keep all. The list itself is the whitelist.

| Metric                              | Signal | Rationale                                          |
| ----------------------------------- | ------ | -------------------------------------------------- |
| `apiserver_request_total`           | node   | Request rate + error rate (via `code` label)       |
| `apiserver_current_inflight_requests` | node | Saturation indicator                               |

## `hostMetrics.linuxHosts`

Uses `useIntegrationAllowList: true` (powers the Grafana Cloud Linux Node
integration dashboard). Not enumerated here; the integration list is the
source of truth. Additional excludes applied to trim per-device noise:

Excluded via `excludeMetrics`:
- `node_network_receive_.*`
- `node_network_transmit_.*`
- `node_disk_io_time_weighted_seconds_total`
- `node_scrape_collector_.*`

## `integrations.alloy` self-monitoring

Alloy self-monitoring uses its own instance-scoped `includeMetrics`. Metrics
kept for the `alloy` instance (daemonset + singleton):

| Metric                                            | Signal            | Rationale                                     |
| ------------------------------------------------- | ----------------- | --------------------------------------------- |
| `alloy_build_info`                                | monitoring-health | Build/version join label                      |
| `prometheus_remote_storage_samples_total`         | monitoring-health | Samples pushed to Grafana Cloud (rate)        |
| `prometheus_remote_storage_samples_failed_total`  | monitoring-health | Failed pushes - non-zero means pipeline broken|
| `prometheus_remote_storage_sent_bytes_total`      | monitoring-health | Egress bytes rate                             |

For the `alloy-metrics` instance the chart's excludes remain in effect (drop
the two noisiest histograms). See the overlay files for exact form.

## `annotationAutodiscovery`

Enabled as an opt-in contract (`prometheus.io/scrape=true`). No consumers
today. When an app opts in, its metrics flow subject to these excludes:

- `go_.*`
- `process_.*`
- `.*_bucket`
- `.*_sum`

## `prometheusOperatorObjects`

**Disabled.** No app currently ships a ServiceMonitor or PodMonitor we want
consumed. To re-enable for a specific namespace, uncomment the stub in the
overlay and add the namespace under `serviceMonitors.namespaces` /
`podMonitors.namespaces`. Do not add `excludeNamespaces` - deny-by-default
via positive `namespaces` scoping.

## Cluster-specific

- **understairs**: OpenWRT scrape via `prometheus.scrape "openwrt"` in
  `collectors.alloy-singleton.extraConfig`. Three targets, ~150 series.
  Not part of the whitelist (uses raw Alloy config, not chart `metricsTuning`).
- **freeloader**: `nodeLogs.enabled: true` (OCI host journal). Not a metric
  source; logs are out of scope for this inventory.
