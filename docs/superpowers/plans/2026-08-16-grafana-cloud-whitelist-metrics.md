# Grafana Cloud whitelist-based metrics collection - Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current permissive `useDefaultAllowList: true` configuration in both cluster overlays with a strict per-source metric whitelist, driving Grafana Cloud active-series usage from ~12k to ~4k while preserving node/pod/storage/monitoring-pipeline visibility.

**Architecture:** Per-source `useDefaultAllowList: false` + explicit `includeMetrics: [...]` inside each cluster overlay's `alloy-values.yaml`, using the k8s-monitoring chart's canonical `metricsTuning` idiom. Duplicated across two overlays (no shared component at N=2 clusters). A new `apps/grafana/METRICS.md` file is the canonical inventory kept in sync with the overlays.

**Tech Stack:** Flux HelmRelease, `grafana/k8s-monitoring` chart `~4.1`, Alloy collectors, Kustomize overlays.

**Spec reference:** `docs/superpowers/specs/2026-08-16-grafana-cloud-whitelist-metrics-design.md`

---

## File Structure

**Files created:**
- `apps/grafana/METRICS.md` - canonical inventory of whitelisted metrics with signal and rationale per row.

**Files modified:**
- `apps/grafana/overlays/freeloader/alloy-values.yaml` - flip six sources to explicit whitelist form, disable `prometheusOperatorObjects`, tighten `annotationAutodiscovery`, extend Alloy self-monitoring includes.
- `apps/grafana/overlays/understairs/alloy-values.yaml` - identical changes to the whitelist portions. Syslog receiver and OpenWRT scrape blocks under `collectors.alloy-singleton.extraConfig` MUST remain untouched.

**Files untouched:**
- `apps/grafana/base/helmrelease.yaml`, `apps/grafana/base/helmrepository.yaml`, `apps/grafana/base/kustomization.yaml`, `apps/grafana/base/namespace.yaml`, `apps/grafana/base/kustomizeconfig.yaml`, `apps/grafana/base/alloy-token.yaml`.
- All Flux `Kustomization` files under `clusters/`.
- Overlays' `ingress.tailscale.yaml`, `kustomization.yaml`, `alloy-token.sops.yaml`, `kustomizeconfig.yaml`.

---

## Task 1: Add METRICS.md inventory

**Files:**
- Create: `apps/grafana/METRICS.md`

- [ ] **Step 1: Write the inventory file**

Create `apps/grafana/METRICS.md` with the exact content below (copy-paste, do not paraphrase):

````markdown
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
````

- [ ] **Step 2: Verify file exists**

Run: `ls -la /Users/marco/Repositories/platform/apps/grafana/METRICS.md`
Expected: file listed, non-zero size.

- [ ] **Step 3: Commit**

```bash
cd /Users/marco/Repositories/platform
git add apps/grafana/METRICS.md
git commit -m "docs(grafana): add metrics inventory for Grafana Cloud whitelist"
```

Expected: pre-commit hooks pass, one file changed.

---

## Task 2: Update freeloader overlay to strict whitelist

**Files:**
- Modify: `apps/grafana/overlays/freeloader/alloy-values.yaml`

- [ ] **Step 1: Replace the file contents**

Overwrite `apps/grafana/overlays/freeloader/alloy-values.yaml` with exactly this content:

```yaml
---
# Metric whitelist inventory: see apps/grafana/METRICS.md.
# Any change to includeMetrics/excludeMetrics here must be mirrored in METRICS.md.
cluster:
  name: freeloader

destinations:
  grafana-cloud-metrics:
    type: prometheus
    url: https://prometheus-prod-24-prod-eu-west-2.grafana.net/api/prom/push
    auth:
      type: basic
      username: "2355589"
  grafana-cloud-logs:
    type: loki
    url: https://logs-prod-012.grafana.net/loki/api/v1/push
    auth:
      type: basic
      username: "1173472"

global:
  scrapeInterval: 120s

# -- Kubernetes observability --

clusterMetrics:
  enabled: true
  collector: alloy-metrics
  kube-state-metrics:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - kube_node_info
        - kube_node_status_condition
        - kube_node_spec_unschedulable
        - kube_pod_info
        - kube_pod_status_phase
        - kube_pod_status_ready
        - kube_pod_container_status_restarts_total
        - kube_pod_container_status_waiting_reason
        - kube_pod_container_status_terminated_reason
        - kube_deployment_status_replicas_available
        - kube_deployment_status_replicas_unavailable
        - kube_statefulset_status_replicas_ready
        - kube_daemonset_status_number_unavailable
        - kube_persistentvolume_status_phase
        - kube_persistentvolumeclaim_status_phase
  kubelet:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - kubelet_node_name
        - kubelet_volume_stats_available_bytes
        - kubelet_volume_stats_capacity_bytes
  cadvisor:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - container_cpu_usage_seconds_total
        - container_memory_working_set_bytes
        - container_oom_events_total
        - container_last_seen
  apiServer:
    enabled: true
    metricsTuning:
      includeMetrics:
        - apiserver_request_total
        - apiserver_current_inflight_requests
  controlPlane:
    enabled: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true
      excludeMetrics:
        - node_network_receive_.*
        - node_network_transmit_.*
        - node_disk_io_time_weighted_seconds_total
        - node_scrape_collector_.*

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port
  metricsTuning:
    excludeMetrics:
      - go_.*
      - process_.*
      - .*_bucket
      - .*_sum

# Disabled by design: no app currently ships a ServiceMonitor or PodMonitor
# that should be consumed. To re-enable for a namespace, flip enabled: true
# and populate serviceMonitors.namespaces (positive allowlist).
prometheusOperatorObjects:
  enabled: false
  # collector: alloy-metrics
  # serviceMonitors:
  #   namespaces: [<namespace>]
  #   metricsTuning:
  #     excludeMetrics: [".*_bucket", ".*_sum"]
  # podMonitors:
  #   namespaces: [<namespace>]
  #   metricsTuning:
  #     excludeMetrics: [".*_bucket", ".*_sum"]

clusterEvents:
  enabled: true
  collector: alloy-singleton

podLogsViaLoki:
  enabled: true
  collector: alloy-daemonset
  excludeNamespaces: [kube-system, flux-system, velero]
  extraLogProcessingStages: |-
    stage.match {
      selector = "{namespace=\"grafana\", app=\"alloy\"} |~ \"level=info\""
      action = "drop"
    }
    stage.match {
      selector = "{namespace=\"argo-cd\", app=\"argocd-repo-server\"} |~ \"level=info\""
      action = "drop"
    }

nodeLogs:
  enabled: true
  collector: alloy-daemonset
  extraLogProcessingStages: |-
    stage.match {
      selector = "{unit=\"kubelet.service\"} |~ \"level=info\""
      action = "drop"
    }

# -- Self-monitoring --

integrations:
  alloy:
    instances:
      - name: alloy-metrics
        labelSelectors:
          app.kubernetes.io/name: alloy-metrics
        metrics:
          tuning:
            excludeMetrics:
              - alloy_component_dependencies_wait_seconds_bucket
              - alloy_component_evaluation_seconds_bucket
      - name: alloy
        labelSelectors:
          app.kubernetes.io/name: [alloy-daemonset, alloy-singleton]
        metrics:
          tuning:
            useDefaultAllowList: false
            includeMetrics:
              - alloy_build_info
              - prometheus_remote_storage_samples_total
              - prometheus_remote_storage_samples_failed_total
              - prometheus_remote_storage_sent_bytes_total

# -- Collectors --

collectors:
  alloy-metrics:
    enabled: true
    presets: [clustered, statefulset]
  alloy-daemonset:
    enabled: true
    presets: [daemonset, filesystem-log-reader]
  alloy-singleton:
    enabled: true
    presets: [singleton]

telemetryServices:
  kube-state-metrics:
    deploy: true
  node-exporter:
    deploy: true
```

- [ ] **Step 2: Verify YAML parses**

Run: `yq . /Users/marco/Repositories/platform/apps/grafana/overlays/freeloader/alloy-values.yaml > /dev/null`
Expected: no output, exit code 0. Any parse error means the paste is corrupted; re-do step 1.

- [ ] **Step 3: Spot-check key structural changes**

Run these four checks; each must produce the exact output shown.

```bash
yq '.clusterMetrics."kube-state-metrics".metricsTuning.useDefaultAllowList' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/freeloader/alloy-values.yaml
# Expected: false

yq '.clusterMetrics.apiServer.metricsTuning | has("useDefaultAllowList")' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/freeloader/alloy-values.yaml
# Expected: false

yq '.prometheusOperatorObjects.enabled' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/freeloader/alloy-values.yaml
# Expected: false

yq '.integrations.alloy.instances[] | select(.name == "alloy") | .metrics.tuning.includeMetrics | length' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/freeloader/alloy-values.yaml
# Expected: 4
```

- [ ] **Step 4: Commit**

```bash
cd /Users/marco/Repositories/platform
git add apps/grafana/overlays/freeloader/alloy-values.yaml
git commit -m "feat(grafana): apply strict per-source metric whitelist to freeloader"
```

Expected: pre-commit hooks pass, one file changed.

---

## Task 3: Update understairs overlay to strict whitelist

**Files:**
- Modify: `apps/grafana/overlays/understairs/alloy-values.yaml`

- [ ] **Step 1: Replace the file contents**

Overwrite `apps/grafana/overlays/understairs/alloy-values.yaml` with exactly this content. Note that `collectors.alloy-singleton` retains its `nodeSelector`, `tolerations`, `extraService`, `extraPorts`, and `extraConfig` (syslog + OpenWRT) blocks; `collectors.alloy-metrics` retains its control-plane `nodeSelector` and `tolerations`.

```yaml
---
# Metric whitelist inventory: see apps/grafana/METRICS.md.
# Any change to includeMetrics/excludeMetrics here must be mirrored in METRICS.md.
cluster:
  name: understairs

destinations:
  grafana-cloud-metrics:
    type: prometheus
    url: https://prometheus-prod-24-prod-eu-west-2.grafana.net/api/prom/push
    auth:
      type: basic
      username: "2355589"
  grafana-cloud-logs:
    type: loki
    url: https://logs-prod-012.grafana.net/loki/api/v1/push
    auth:
      type: basic
      username: "1173472"

global:
  scrapeInterval: 120s

# -- Kubernetes observability --

clusterMetrics:
  enabled: true
  collector: alloy-metrics
  kube-state-metrics:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - kube_node_info
        - kube_node_status_condition
        - kube_node_spec_unschedulable
        - kube_pod_info
        - kube_pod_status_phase
        - kube_pod_status_ready
        - kube_pod_container_status_restarts_total
        - kube_pod_container_status_waiting_reason
        - kube_pod_container_status_terminated_reason
        - kube_deployment_status_replicas_available
        - kube_deployment_status_replicas_unavailable
        - kube_statefulset_status_replicas_ready
        - kube_daemonset_status_number_unavailable
        - kube_persistentvolume_status_phase
        - kube_persistentvolumeclaim_status_phase
  kubelet:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - kubelet_node_name
        - kubelet_volume_stats_available_bytes
        - kubelet_volume_stats_capacity_bytes
  cadvisor:
    metricsTuning:
      useDefaultAllowList: false
      includeMetrics:
        - container_cpu_usage_seconds_total
        - container_memory_working_set_bytes
        - container_oom_events_total
        - container_last_seen
  apiServer:
    enabled: true
    metricsTuning:
      includeMetrics:
        - apiserver_request_total
        - apiserver_current_inflight_requests
  controlPlane:
    enabled: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true
      excludeMetrics:
        - node_network_receive_.*
        - node_network_transmit_.*
        - node_disk_io_time_weighted_seconds_total
        - node_scrape_collector_.*

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port
  metricsTuning:
    excludeMetrics:
      - go_.*
      - process_.*
      - .*_bucket
      - .*_sum

# Disabled by design: no app currently ships a ServiceMonitor or PodMonitor
# that should be consumed. To re-enable for a namespace, flip enabled: true
# and populate serviceMonitors.namespaces (positive allowlist).
prometheusOperatorObjects:
  enabled: false
  # collector: alloy-metrics
  # serviceMonitors:
  #   namespaces: [<namespace>]
  #   metricsTuning:
  #     excludeMetrics: [".*_bucket", ".*_sum"]
  # podMonitors:
  #   namespaces: [<namespace>]
  #   metricsTuning:
  #     excludeMetrics: [".*_bucket", ".*_sum"]

clusterEvents:
  enabled: true
  collector: alloy-singleton

podLogsViaLoki:
  enabled: true
  collector: alloy-daemonset
  excludeNamespaces: [kube-system, flux-system, velero]
  extraLogProcessingStages: |-
    stage.match {
      selector = "{namespace=\"grafana\", app=\"alloy\"} |~ \"level=info\""
      action = "drop"
    }
    stage.match {
      selector = "{namespace=\"argo-cd\", app=\"argocd-repo-server\"} |~ \"level=info\""
      action = "drop"
    }

# Talos Linux has no systemd-journald (/var/log/journal).
nodeLogs:
  enabled: false

# -- Self-monitoring --

integrations:
  alloy:
    instances:
      - name: alloy-metrics
        labelSelectors:
          app.kubernetes.io/name: alloy-metrics
        metrics:
          tuning:
            excludeMetrics:
              - alloy_component_dependencies_wait_seconds_bucket
              - alloy_component_evaluation_seconds_bucket
      - name: alloy
        labelSelectors:
          app.kubernetes.io/name: [alloy-daemonset, alloy-singleton]
        metrics:
          tuning:
            useDefaultAllowList: false
            includeMetrics:
              - alloy_build_info
              - prometheus_remote_storage_samples_total
              - prometheus_remote_storage_samples_failed_total
              - prometheus_remote_storage_sent_bytes_total

# -- Collectors --

collectors:
  alloy-metrics:
    enabled: true
    presets: [clustered, statefulset]
    controller:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
  alloy-daemonset:
    enabled: true
    presets: [daemonset, filesystem-log-reader]
  alloy-singleton:
    enabled: true
    presets: [singleton]
    controller:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
    extraService:
      enabled: true
    alloy:
      extraPorts:
        - name: syslog
          port: 1514
          targetPort: 1514
          protocol: TCP
    extraConfig: |-
      // Syslog receiver for network devices (OpenWRT, switches, APs)
      loki.source.syslog "network_devices" {
        listener {
          address  = "0.0.0.0:1514"
          protocol = "tcp"
          labels   = {
            job = "syslog/network-devices",
          }
        }
        forward_to = [loki.write.grafana_cloud_logs.receiver]
      }

      // Prometheus scrape for OpenWRT node exporters
      prometheus.scrape "openwrt" {
        targets = [
          {"__address__" = "192.168.20.5:9100", "instance" = "ap-01"},
          {"__address__" = "192.168.20.6:9100", "instance" = "ap-02"},
          {"__address__" = "192.168.20.7:9100", "instance" = "ap-03"},
        ]
        scrape_interval = "30s"
        forward_to      = [prometheus.remote_write.grafana_cloud_metrics.receiver]
      }

telemetryServices:
  kube-state-metrics:
    deploy: true
  node-exporter:
    deploy: true
```

- [ ] **Step 2: Verify YAML parses**

Run: `yq . /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml > /dev/null`
Expected: no output, exit code 0.

- [ ] **Step 3: Spot-check whitelist structural changes**

```bash
yq '.clusterMetrics."kube-state-metrics".metricsTuning.useDefaultAllowList' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: false

yq '.clusterMetrics.apiServer.metricsTuning | has("useDefaultAllowList")' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: false

yq '.prometheusOperatorObjects.enabled' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: false
```

- [ ] **Step 4: Spot-check that cluster-specific blocks are preserved**

```bash
yq '.collectors."alloy-singleton".alloy.extraPorts[0].port' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: 1514

yq '.collectors."alloy-singleton".extraConfig' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml | grep -c 'openwrt'
# Expected: 1 (at least; the block contains one 'openwrt' reference)

yq '.collectors."alloy-metrics".controller.nodeSelector."node-role.kubernetes.io/control-plane"' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: "" (empty string)

yq '.nodeLogs.enabled' \
  /Users/marco/Repositories/platform/apps/grafana/overlays/understairs/alloy-values.yaml
# Expected: false
```

- [ ] **Step 5: Commit**

```bash
cd /Users/marco/Repositories/platform
git add apps/grafana/overlays/understairs/alloy-values.yaml
git commit -m "feat(grafana): apply strict per-source metric whitelist to understairs"
```

Expected: pre-commit hooks pass, one file changed.

---

## Task 4: Pre-merge render verification (both overlays)

This task does not modify files. It verifies that both overlays render to
sensible Alloy configuration before we let Flux apply them. Failures here
mean going back to Task 2 or 3 and fixing the offending value.

**Files:**
- Read: `apps/grafana/overlays/freeloader/alloy-values.yaml`
- Read: `apps/grafana/overlays/understairs/alloy-values.yaml`

- [ ] **Step 1: Ensure the chart repo is registered locally**

```bash
helm repo add grafana https://grafana.github.io/helm-charts 2>/dev/null || true
helm repo update grafana
```

Expected: `...Successfully got an update from the "grafana" chart repository`.

- [ ] **Step 2: Render freeloader values**

The chart requires auth passwords for its destinations. Inject a dummy value
via `--set` since the real one comes from Flux `valuesFrom` at runtime.

```bash
cd /Users/marco/Repositories/platform
helm template alloy grafana/k8s-monitoring --version 4.1.4 \
  -f apps/grafana/overlays/freeloader/alloy-values.yaml \
  --set destinations.grafana-cloud-metrics.auth.password=dummy \
  --set destinations.grafana-cloud-logs.auth.password=dummy \
  > /tmp/render-freeloader.yaml
```

Expected: exit code 0, `/tmp/render-freeloader.yaml` non-empty.

- [ ] **Step 3: Confirm keep-rules for whitelisted metrics are present**

The chart implements `useDefaultAllowList: false` + `includeMetrics` via a
`prometheus.relabel` block with `action = "keep"` and a regex of the metric
names joined by `|`. Confirm one metric from each source appears in the
rendered config's keep rules:

```bash
grep -c 'kube_pod_status_phase' /tmp/render-freeloader.yaml
# Expected: at least 1

grep -c 'container_cpu_usage_seconds_total' /tmp/render-freeloader.yaml
# Expected: at least 1

grep -c 'kubelet_volume_stats_available_bytes' /tmp/render-freeloader.yaml
# Expected: at least 1

grep -c 'apiserver_request_total' /tmp/render-freeloader.yaml
# Expected: at least 1
```

- [ ] **Step 4: Confirm dropped metrics do NOT appear in keep-rule regexes**

Pick a handful of metrics that were on the previous default allowlist and
should now be excluded. They should NOT appear in the rendered config
except (permissibly) inside chart-internal comments.

```bash
grep -E 'container_blkio_|container_tasks_state|apiserver_request_duration_seconds_bucket' \
  /tmp/render-freeloader.yaml || echo "OK: not present"
# Expected: "OK: not present"

grep 'kube_replicaset_status_replicas' /tmp/render-freeloader.yaml || echo "OK: not present"
# Expected: "OK: not present"
```

If any of these show up as unquoted strings inside a `regex = "..."` line,
the whitelist is not being enforced - stop and re-check Task 2.

- [ ] **Step 5: Confirm prometheusOperatorObjects yields no discovery**

```bash
grep -c 'prometheus.operator.servicemonitors' /tmp/render-freeloader.yaml
# Expected: 0

grep -c 'prometheus.operator.podmonitors' /tmp/render-freeloader.yaml
# Expected: 0
```

- [ ] **Step 6: Render and check understairs the same way**

```bash
cd /Users/marco/Repositories/platform
helm template alloy grafana/k8s-monitoring --version 4.1.4 \
  -f apps/grafana/overlays/understairs/alloy-values.yaml \
  --set destinations.grafana-cloud-metrics.auth.password=dummy \
  --set destinations.grafana-cloud-logs.auth.password=dummy \
  > /tmp/render-understairs.yaml

# Same four keep-rule checks:
grep -c 'kube_pod_status_phase' /tmp/render-understairs.yaml               # >= 1
grep -c 'container_cpu_usage_seconds_total' /tmp/render-understairs.yaml   # >= 1
grep -c 'kubelet_volume_stats_available_bytes' /tmp/render-understairs.yaml # >= 1
grep -c 'apiserver_request_total' /tmp/render-understairs.yaml             # >= 1

# Same exclude checks:
grep -E 'container_blkio_|apiserver_request_duration_seconds_bucket' \
  /tmp/render-understairs.yaml || echo "OK: not present"

# ServiceMonitor/PodMonitor discovery off:
grep -c 'prometheus.operator.servicemonitors' /tmp/render-understairs.yaml  # 0
grep -c 'prometheus.operator.podmonitors' /tmp/render-understairs.yaml      # 0

# Cluster-specific bits still present:
grep -c 'loki.source.syslog "network_devices"' /tmp/render-understairs.yaml # 1
grep -c 'prometheus.scrape "openwrt"' /tmp/render-understairs.yaml          # 1
```

Each grep must match its expected count. If any fails, stop and re-check Task 3.

- [ ] **Step 7: Cleanup**

```bash
rm /tmp/render-freeloader.yaml /tmp/render-understairs.yaml
```

No commit for this task - it's verification only. If all checks passed,
proceed to Task 5.

---

## Task 5: Inventory / overlay sync check

**Files:**
- Read: `apps/grafana/METRICS.md`
- Read: `apps/grafana/overlays/freeloader/alloy-values.yaml`
- Read: `apps/grafana/overlays/understairs/alloy-values.yaml`

Confirm the metric names in the overlays match the metric names in the
inventory. This is a hand-diff; the two sources drift over time without a
generator.

- [ ] **Step 1: Extract overlay includeMetrics per source (freeloader)**

```bash
cd /Users/marco/Repositories/platform
yq '.clusterMetrics."kube-state-metrics".metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/freeloader/alloy-values.yaml | sort -u > /tmp/overlay-ksm.txt
yq '.clusterMetrics.cadvisor.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/freeloader/alloy-values.yaml | sort -u > /tmp/overlay-cadvisor.txt
yq '.clusterMetrics.kubelet.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/freeloader/alloy-values.yaml | sort -u > /tmp/overlay-kubelet.txt
yq '.clusterMetrics.apiServer.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/freeloader/alloy-values.yaml | sort -u > /tmp/overlay-apiserver.txt
yq '.integrations.alloy.instances[] | select(.name == "alloy") | .metrics.tuning.includeMetrics[]' \
  apps/grafana/overlays/freeloader/alloy-values.yaml | sort -u > /tmp/overlay-alloy-self.txt
```

- [ ] **Step 2: Extract METRICS.md metric names per source**

```bash
cd /Users/marco/Repositories/platform
awk '/^## `clusterMetrics.kube-state-metrics`/,/^## /' apps/grafana/METRICS.md \
  | grep -oE '`kube_[a-z_]+`' | tr -d '`' | sort -u > /tmp/inv-ksm.txt
awk '/^## `clusterMetrics.cadvisor`/,/^## /' apps/grafana/METRICS.md \
  | grep -oE '`container_[a-z_]+`' | tr -d '`' | sort -u > /tmp/inv-cadvisor.txt
awk '/^## `clusterMetrics.kubelet`/,/^## /' apps/grafana/METRICS.md \
  | grep -oE '`kubelet_[a-z_]+`' | tr -d '`' | sort -u > /tmp/inv-kubelet.txt
awk '/^## `clusterMetrics.apiServer`/,/^## /' apps/grafana/METRICS.md \
  | grep -oE '`apiserver_[a-z_]+`' | tr -d '`' | sort -u > /tmp/inv-apiserver.txt
awk '/^## `integrations.alloy` self-monitoring/,/^## /' apps/grafana/METRICS.md \
  | grep -oE '`(alloy_build_info|prometheus_remote_storage_[a-z_]+)`' | tr -d '`' | sort -u > /tmp/inv-alloy-self.txt
```

- [ ] **Step 3: Diff each pair**

```bash
diff /tmp/overlay-ksm.txt /tmp/inv-ksm.txt && echo "kube-state-metrics: match"
diff /tmp/overlay-cadvisor.txt /tmp/inv-cadvisor.txt && echo "cadvisor: match"
diff /tmp/overlay-kubelet.txt /tmp/inv-kubelet.txt && echo "kubelet: match"
diff /tmp/overlay-apiserver.txt /tmp/inv-apiserver.txt && echo "apiServer: match"
diff /tmp/overlay-alloy-self.txt /tmp/inv-alloy-self.txt && echo "alloy self: match"
```

Expected: each `diff` produces no output followed by `<source>: match`.
Any differences mean either the overlay or `METRICS.md` is wrong - fix and
re-run.

- [ ] **Step 4: Repeat step 1 for understairs**

```bash
cd /Users/marco/Repositories/platform
yq '.clusterMetrics."kube-state-metrics".metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/understairs/alloy-values.yaml | sort -u > /tmp/overlay-ksm-u.txt
yq '.clusterMetrics.cadvisor.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/understairs/alloy-values.yaml | sort -u > /tmp/overlay-cadvisor-u.txt
yq '.clusterMetrics.kubelet.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/understairs/alloy-values.yaml | sort -u > /tmp/overlay-kubelet-u.txt
yq '.clusterMetrics.apiServer.metricsTuning.includeMetrics[]' \
  apps/grafana/overlays/understairs/alloy-values.yaml | sort -u > /tmp/overlay-apiserver-u.txt
yq '.integrations.alloy.instances[] | select(.name == "alloy") | .metrics.tuning.includeMetrics[]' \
  apps/grafana/overlays/understairs/alloy-values.yaml | sort -u > /tmp/overlay-alloy-self-u.txt

diff /tmp/overlay-ksm-u.txt /tmp/inv-ksm.txt && echo "understairs kube-state-metrics: match"
diff /tmp/overlay-cadvisor-u.txt /tmp/inv-cadvisor.txt && echo "understairs cadvisor: match"
diff /tmp/overlay-kubelet-u.txt /tmp/inv-kubelet.txt && echo "understairs kubelet: match"
diff /tmp/overlay-apiserver-u.txt /tmp/inv-apiserver.txt && echo "understairs apiServer: match"
diff /tmp/overlay-alloy-self-u.txt /tmp/inv-alloy-self.txt && echo "understairs alloy self: match"
```

Expected: five `match` lines and no diff output.

- [ ] **Step 5: Cleanup**

```bash
rm -f /tmp/overlay-*.txt /tmp/inv-*.txt
```

No commit for this task. If any diff failed, go back to Task 1 or Task 2/3 to fix.

---

## Task 6: Report readiness

This task is a summary handoff to the user. No file changes, no verification
commands beyond `git status`. The deploy itself happens by pushing/merging to
`main`, which Flux picks up on the cluster - the user drives that step, and
post-deploy verification (active-series drop, dashboard walk-through) happens
in Grafana Cloud outside this repo.

- [ ] **Step 1: Show status and diff summary**

```bash
cd /Users/marco/Repositories/platform
git status
git log --oneline main..HEAD
git diff --stat main..HEAD
```

Expected: three commits ahead of main (docs + freeloader + understairs), only
three files touched (`apps/grafana/METRICS.md`, both `alloy-values.yaml`).

- [ ] **Step 2: Present the post-deploy plan to the user**

Report back to the user:

- All three commits are on the local branch, ready to push/PR.
- Pre-merge render checks (Task 4) passed for both overlays.
- Inventory/overlay sync check (Task 5) passed for both clusters.
- After Flux reconciles, verify in Grafana Cloud:
  - Billing -> Active Series drops to ~4-5k within one hour.
  - `topk(20, count by (__name__)({cluster=~"freeloader|understairs"}))`
    returns only names present in `apps/grafana/METRICS.md`.
  - `count(count by (__name__)({cluster=~"freeloader|understairs"}))`
    returns roughly 25-30.
  - `prometheus_remote_storage_samples_failed_total > 0` should stay false;
    bookmark it.
  - Walk the Grafana Cloud Kubernetes integration dashboards - any panel
    with "No data" is a follow-up (add metric to overlays + METRICS.md, or
    accept the panel as out of scope).

---

## Self-review

- **Spec coverage:**
  - Whitelist inventory (spec section) - Task 1 (METRICS.md) + Tasks 2/3 (overlays).
  - Two config bugs (apiServer useDefaultAllowList, prometheusOperatorObjects excludeNamespaces) - Tasks 2 and 3 both fix them, Task 4 grep verifies.
  - Self-monitoring extension (three remote_write metrics) - Task 2 step 3 check 4, Task 5 diff.
  - annotationAutodiscovery excludes - present in overlays (Task 2/3), rendered check not explicitly done but low risk.
  - Cluster-specific preservation (syslog, OpenWRT, nodeSelectors) - Task 3 step 4 checks all three.
  - File-level scope (only three files touched) - Task 6 step 1 verifies via `git diff --stat`.
  - Rollout (single PR / three commits) - Task 6 step 1 verifies.
  - Pre-merge verification (helm template + grep) - Task 4.
  - Post-deploy verification - Task 6 step 2 relays to user; happens in Grafana Cloud not this repo.
- **Placeholder scan:** No TBDs, no "similar to task N", every file path is absolute, every command has expected output, every code block is complete.
- **Type consistency:** Metric names identical across Task 1 (METRICS.md), Task 2 (freeloader), Task 3 (understairs). The five source keys (`kube-state-metrics`, `cadvisor`, `kubelet`, `apiServer`, `alloy` self-monitoring) match everywhere. Task 5 hard-diffs this rather than trusting it.
