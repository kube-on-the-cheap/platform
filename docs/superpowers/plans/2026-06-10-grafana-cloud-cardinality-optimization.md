# Grafana Cloud cardinality optimization implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore Grafana Cloud ingestion from both clusters under the 20k active-series free-tier ceiling, by upgrading the `grafana/k8s-monitoring` Helm chart from `~2.0` to `~4.1` and layering cardinality cuts and log trims on top of the v4 schema.

**Architecture:** Three independent PRs landed in sequence. First introduces a standalone `prometheus-operator-crds` Helm chart so the CRDs that v2 of `k8s-monitoring` bundled are owned independently before v4 unbundles them. Second migrates `apps/grafana` to the v4 chart schema with behavior preserved (Phase 0). Third applies the cardinality and log cuts (Phase 1). Freeloader's `grafana.yaml` is currently commented out of its cluster Kustomization, so the rollout enables freeloader during Phase 0 and validates there before touching understairs.

**Tech Stack:** Flux CD v2 (HelmRelease, Kustomization), Helm 3, `grafana/k8s-monitoring` 4.1.x, `prometheus-community/prometheus-operator-crds` 29.x, Grafana Alloy, SOPS (understairs only), External Secrets Operator (freeloader only). Talos Linux nodes on understairs; OCI/Oracle Linux on freeloader.

**Reference spec:** `docs/superpowers/specs/2026-06-09-grafana-cloud-cardinality-optimization-design.md`

---

## File structure

### New files

- `apps/prometheus-operator-crds/base/helmrepository.yaml` — `prometheus-community` HelmRepository.
- `apps/prometheus-operator-crds/base/helmrelease.yaml` — pins `prometheus-operator-crds ~29.0`, installs CRDs at cluster scope.
- `apps/prometheus-operator-crds/base/kustomization.yaml` — collects the two resources above.
- `clusters/freeloader/prometheus-operator-crds.yaml` — Flux Kustomization wiring the app into freeloader.
- `clusters/understairs/prometheus-operator-crds.yaml` — Flux Kustomization wiring the app into understairs.

### Modified files

- `apps/grafana/base/helmrelease.yaml` — bump chart version to `~4.1`, rewrite `valuesFrom` `targetPath` from array indices to map keys.
- `apps/grafana/overlays/freeloader/alloy-values.yaml` — full rewrite to v4 schema (Phase 0), then layered cuts (Phase 1).
- `apps/grafana/overlays/understairs/alloy-values.yaml` — full rewrite to v4 schema with `collectors.alloy-singleton` carrying the syslog receiver (Phase 0), then layered cuts (Phase 1).
- `clusters/freeloader/kustomization.yaml` — uncomment `grafana.yaml`, add `prometheus-operator-crds.yaml`.
- `clusters/understairs/kustomization.yaml` — add `prometheus-operator-crds.yaml`.

### File responsibilities

- `apps/prometheus-operator-crds/base/*` — single-purpose: own the Prometheus Operator CRDs (`monitoring.coreos.com` group). No controllers.
- `apps/grafana/base/helmrelease.yaml` — describes the Alloy/k8s-monitoring HelmRelease shared by both clusters; per-cluster differences live in overlays.
- `apps/grafana/overlays/<cluster>/alloy-values.yaml` — single source of truth for what the cluster ships to Grafana Cloud. Per-cluster, no shared base values file.
- `clusters/<cluster>/*.yaml` — Flux entrypoint that activates the app on that cluster, with the right `dependsOn` chain.

---

## Conventions used in this plan

- **Editor-level dry-render before each merge.** Every overlay change is rendered locally with `helm template` and inspected before commit. The plan shows the exact command.
- **One Flux reconciliation per task.** After each commit that should reconcile, the plan shows the `flux reconcile` commands and the expected pod state.
- **kubectl context.** Where context matters, the plan shows the explicit `--context` flag. Set the active context to either `freeloader` or `understairs` (whatever your kubeconfig calls them) before running cluster-touching commands.
- **Commit style.** Follows the repo's Conventional Commits style (see `git log` for examples). Pre-commit hooks run; if they reformat anything, re-stage and re-commit.

---

## Task 1: Add Prometheus Operator CRDs app skeleton

**Files:**
- Create: `apps/prometheus-operator-crds/base/helmrepository.yaml`
- Create: `apps/prometheus-operator-crds/base/helmrelease.yaml`
- Create: `apps/prometheus-operator-crds/base/kustomization.yaml`

- [ ] **Step 1: Create the HelmRepository**

Create `apps/prometheus-operator-crds/base/helmrepository.yaml`:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 24h
  url: https://prometheus-community.github.io/helm-charts
```

- [ ] **Step 2: Create the HelmRelease**

Create `apps/prometheus-operator-crds/base/helmrelease.yaml`:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prometheus-operator-crds
  namespace: flux-system
spec:
  interval: 30m
  timeout: 5m
  releaseName: prometheus-operator-crds
  targetNamespace: flux-system
  chart:
    spec:
      chart: prometheus-operator-crds
      version: "~29.0"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  install:
    crds: CreateReplace
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
```

- [ ] **Step 3: Create the kustomization**

Create `apps/prometheus-operator-crds/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml
  - helmrelease.yaml
```

- [ ] **Step 4: Verify the app renders cleanly**

Run from the repo root:

```bash
kubectl kustomize apps/prometheus-operator-crds/base
```

Expected: YAML output containing both the HelmRepository and the HelmRelease, no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/prometheus-operator-crds/
git commit -m "feat(prometheus-operator-crds): add CRD-only chart app

Pre-step for migrating grafana/k8s-monitoring from ~2.0 to ~4.1, which
unbundles the Prometheus Operator CRDs."
```

---

## Task 2: Wire CRDs app into understairs cluster

**Files:**
- Create: `clusters/understairs/prometheus-operator-crds.yaml`
- Modify: `clusters/understairs/kustomization.yaml`

- [ ] **Step 1: Create the Flux Kustomization for understairs**

Create `clusters/understairs/prometheus-operator-crds.yaml`:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prometheus-operator-crds
  namespace: flux-system
spec:
  interval: 15m0s
  sourceRef:
    kind: GitRepository
    name: platform
  path: ./apps/prometheus-operator-crds/base
  prune: true
  wait: true
  timeout: 10m0s
```

- [ ] **Step 2: Add to the cluster kustomization**

Modify `clusters/understairs/kustomization.yaml` so the resources list includes the new file. The file currently lists 20 resources; insert the new one immediately before `grafana.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cilium.yaml
  - openebs-localpv.yaml
  - csi-driver-nfs.yaml
  - tailscale.yaml
  - tailscale-config.yaml
  - cert-manager.yaml
  - certificates.yaml
  - envoy-gateway.yaml
  - envoy-gateway-config.yaml
  - cloudnative-pg.yaml
  - cloudnative-pg-config.yaml
  - shared-secrets.yaml
  - external-secrets-operator.yaml
  - external-secrets.yaml
  - external-dns.yaml
  - prometheus-operator-crds.yaml
  - grafana.yaml
  - apps-bootstrap.yaml
```

- [ ] **Step 3: Verify the cluster kustomization renders**

```bash
kubectl kustomize clusters/understairs
```

Expected: success; output should now include the new Flux Kustomization for `prometheus-operator-crds`.

- [ ] **Step 4: Commit**

```bash
git add clusters/understairs/prometheus-operator-crds.yaml clusters/understairs/kustomization.yaml
git commit -m "feat(understairs): install prometheus-operator-crds

Standalone CRD chart; required before k8s-monitoring v4 which no longer
bundles them."
```

- [ ] **Step 5: Push and reconcile**

```bash
git push origin main
flux --context understairs reconcile source git platform
flux --context understairs reconcile kustomization prometheus-operator-crds --with-source
```

Expected: `prometheus-operator-crds` Kustomization reaches `Ready=True`.

- [ ] **Step 6: Verify CRDs are owned by the new release**

```bash
kubectl --context understairs get crd -l app.kubernetes.io/name=prometheus-operator-crds
```

Expected: list of `monitoring.coreos.com` CRDs (servicemonitors, podmonitors, prometheuses, alertmanagers, probes, prometheusrules, thanosrulers, scrapeconfigs, alertmanagerconfigs).

If Helm refuses ownership (CRDs were previously created by `grafana/k8s-monitoring`), the HelmRelease will show a `cannot patch ... resource ... not managed by Helm` error. Resolve with:

```bash
for crd in alertmanagerconfigs alertmanagers podmonitors probes prometheusagents prometheuses prometheusrules scrapeconfigs servicemonitors thanosrulers; do
  kubectl --context understairs annotate crd "${crd}.monitoring.coreos.com" meta.helm.sh/release-name=prometheus-operator-crds meta.helm.sh/release-namespace=flux-system --overwrite
  kubectl --context understairs label crd "${crd}.monitoring.coreos.com" app.kubernetes.io/managed-by=Helm --overwrite
done
```

Then re-reconcile (`flux --context understairs reconcile helmrelease prometheus-operator-crds`).

---

## Task 3: Wire CRDs app into freeloader cluster

**Files:**
- Create: `clusters/freeloader/prometheus-operator-crds.yaml`
- Modify: `clusters/freeloader/kustomization.yaml`

- [ ] **Step 1: Create the Flux Kustomization for freeloader**

Create `clusters/freeloader/prometheus-operator-crds.yaml`:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prometheus-operator-crds
  namespace: flux-system
spec:
  interval: 15m0s
  sourceRef:
    kind: GitRepository
    name: platform
  path: ./apps/prometheus-operator-crds/base
  prune: true
  wait: true
  timeout: 10m0s
```

- [ ] **Step 2: Add to the cluster kustomization**

Modify `clusters/freeloader/kustomization.yaml`. The existing file has `grafana.yaml` commented out — leave that alone for now (Task 8 will uncomment it). Insert `prometheus-operator-crds.yaml` after `external-dns.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cert-manager.yaml
  - certificates.yaml
  - shared-secrets.yaml
  - external-secrets-operator.yaml
  - external-secrets.yaml
  - external-dns.yaml
  - prometheus-operator-crds.yaml
  # - project-contour.yaml
  - envoy-gateway.yaml
  - envoy-gateway-config.yaml
  - tailscale.yaml
  - tailscale-config.yaml
  # - grafana.yaml
  # - argo-cd.yaml
  # - velero.yaml
  - cloudnative-pg.yaml
  # - local-path-provisioner.yaml
  - openebs-localpv.yaml
  # - retina.yaml
  # - longhorn.yaml
  # - podinfo.yaml
  - apps-bootstrap.yaml
```

- [ ] **Step 3: Verify the cluster kustomization renders**

```bash
kubectl kustomize clusters/freeloader
```

Expected: success.

- [ ] **Step 4: Commit, push, reconcile**

```bash
git add clusters/freeloader/prometheus-operator-crds.yaml clusters/freeloader/kustomization.yaml
git commit -m "feat(freeloader): install prometheus-operator-crds

Standalone CRD chart; required before k8s-monitoring v4 which no longer
bundles them."
git push origin main
flux --context freeloader reconcile source git platform
flux --context freeloader reconcile kustomization prometheus-operator-crds --with-source
```

Expected: `prometheus-operator-crds` Kustomization reaches `Ready=True`.

- [ ] **Step 5: Verify CRDs**

```bash
kubectl --context freeloader get crd -l app.kubernetes.io/name=prometheus-operator-crds
```

If freeloader does not yet have the CRDs (because `grafana.yaml` has been commented out, so v2 never installed them on this cluster), they are simply created fresh by the new release. No ownership transfer needed. The list should still show the same CRDs as understairs.

---

## Task 4: Phase 0 — bump chart version and rewrite Flux `valuesFrom` paths

**Files:**
- Modify: `apps/grafana/base/helmrelease.yaml`

- [ ] **Step 1: Update `apps/grafana/base/helmrelease.yaml`**

Replace the entire file contents with:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: alloy
  namespace: flux-system
spec:
  interval: 30m
  timeout: 5m
  releaseName: alloy
  targetNamespace: grafana
  chart:
    spec:
      chart: k8s-monitoring
      version: "~4.1"
      sourceRef:
        kind: HelmRepository
        name: grafana
  install:
    crds: CreateReplace
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
  valuesFrom:
    - kind: ConfigMap
      name: alloy-values
    - kind: Secret
      name: alloy-token
      targetPath: destinations.grafana-cloud-metrics.auth.password
      valuesKey: alloy-token
    - kind: Secret
      name: alloy-token
      targetPath: destinations.grafana-cloud-logs.auth.password
      valuesKey: alloy-token
```

Two changes versus the previous file:
- `version: "~2.0"` → `version: "~4.1"`.
- Two `targetPath` entries change from `destinations[0].auth.password` / `destinations[1].auth.password` to the map-keyed form. The names `grafana-cloud-metrics` and `grafana-cloud-logs` must match the keys we use in both overlays' `alloy-values.yaml` (Tasks 5 and 6 do this).

- [ ] **Step 2: Verify the base renders**

```bash
kubectl kustomize apps/grafana/base
```

Expected: success.

- [ ] **Step 3: Do NOT commit yet**

Tasks 5 and 6 land the matching overlay rewrites in the same commit; otherwise Flux would attempt to render a v4 chart against v2-schema values and fail.

---

## Task 5: Phase 0 — rewrite understairs overlay to v4 schema (behavior preserved)

**Files:**
- Modify: `apps/grafana/overlays/understairs/alloy-values.yaml`

- [ ] **Step 1: Replace the file contents**

Replace `apps/grafana/overlays/understairs/alloy-values.yaml` with the v4-schema equivalent of the current v2 config. **No cardinality changes** — those land in Task 9.

```yaml
---
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
  scrapeInterval: 90s

# -- Kubernetes observability --

clusterMetrics:
  enabled: true
  collector: alloy-metrics
  kube-state-metrics:
    metricsTuning:
      useDefaultAllowList: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port

prometheusOperatorObjects:
  enabled: true
  collector: alloy-metrics

clusterEvents:
  enabled: true
  collector: alloy-singleton

podLogsViaLoki:
  enabled: true
  collector: alloy-daemonset

# Talos Linux has no systemd-journald (/var/log/journal); nodeLogs stays off.

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

# -- Collectors --

collectors:
  alloy-metrics:
    enabled: true
    presets: [clustered, statefulset]
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
    tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
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

Notes for the implementer:
- The export names `loki.write.grafana_cloud_logs.receiver` and `prometheus.remote_write.grafana_cloud_metrics.receiver` referenced in `extraConfig` follow the v4 chart's naming convention: `<type>.write.<destination-key-with-dashes-replaced-by-underscores>`. The destination keys above are `grafana-cloud-logs` and `grafana-cloud-metrics`, hence the underscores.
- `presets: [clustered, statefulset]` matches the chart's official metrics-tuning example.
- `presets: [daemonset, filesystem-log-reader]` is the v4 equivalent of v2's `alloy-logs` (daemonset-mode log reader). The `privileged` preset is intentionally omitted — Talos rejects privileged pods unless the namespace allows it; the grafana namespace already has `pod-security.kubernetes.io/enforce: privileged` so it would work, but we keep the minimal preset set for now.
- The default `interval` for the OpenWRT scrape stays at 30s; this is intentional (low-cardinality and useful).

- [ ] **Step 2: Render with helm to confirm v4 accepts the values**

The chart pulls extra files via Flux's `valuesFrom`. To dry-render locally, simulate the full values document:

```bash
mkdir -p /tmp/alloy-render
helm repo update grafana 2>/dev/null
# Build a synthesised values file with a placeholder password (real one is injected by Flux at runtime).
cp apps/grafana/overlays/understairs/alloy-values.yaml /tmp/alloy-render/understairs-values.yaml
yq -i '.destinations.grafana-cloud-metrics.auth.password = "PLACEHOLDER"' /tmp/alloy-render/understairs-values.yaml
yq -i '.destinations.grafana-cloud-logs.auth.password    = "PLACEHOLDER"' /tmp/alloy-render/understairs-values.yaml
helm template alloy grafana/k8s-monitoring --version "4.1.4" \
  --namespace grafana \
  -f /tmp/alloy-render/understairs-values.yaml > /tmp/alloy-render/understairs.yaml
echo "Render OK, output at /tmp/alloy-render/understairs.yaml"
```

Expected: exit code 0, no schema errors. If the chart errors out about an unknown key, fix the key and re-render.

- [ ] **Step 3: Inspect the generated Alloy config**

```bash
grep -A 2 'loki.source.syslog' /tmp/alloy-render/understairs.yaml
grep -A 2 'prometheus.scrape "openwrt"' /tmp/alloy-render/understairs.yaml
grep 'loki.write.grafana_cloud_logs' /tmp/alloy-render/understairs.yaml | head -3
grep 'prometheus.remote_write.grafana_cloud_metrics' /tmp/alloy-render/understairs.yaml | head -3
```

Expected: the syslog block, the openwrt scrape block, and the matching `loki.write` / `prometheus.remote_write` export components all appear in the rendered config.

- [ ] **Step 4: Do NOT commit yet**

Task 6 makes the matching freeloader change; both overlays plus the base bump in Task 4 land in a single commit in Task 7.

---

## Task 6: Phase 0 — rewrite freeloader overlay to v4 schema (behavior preserved)

**Files:**
- Modify: `apps/grafana/overlays/freeloader/alloy-values.yaml`

- [ ] **Step 1: Replace the file contents**

Replace `apps/grafana/overlays/freeloader/alloy-values.yaml` with the v4-schema equivalent:

```yaml
---
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
  scrapeInterval: 90s

# -- Kubernetes observability --

clusterMetrics:
  enabled: true
  collector: alloy-metrics
  kube-state-metrics:
    metricsTuning:
      useDefaultAllowList: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port

prometheusOperatorObjects:
  enabled: true
  collector: alloy-metrics

clusterEvents:
  enabled: true
  collector: alloy-singleton

podLogsViaLoki:
  enabled: true
  collector: alloy-daemonset

# OCI nodes have systemd journals; node logs ship via the daemonset collector.
nodeLogs:
  enabled: true
  collector: alloy-daemonset

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

Notes:
- No `tolerations`/`nodeSelector` on `alloy-metrics` here — freeloader does not pin to control-plane nodes the way understairs does.
- No syslog receiver — freeloader has no OpenWRT-equivalent.

- [ ] **Step 2: Render with helm to confirm v4 accepts the values**

```bash
cp apps/grafana/overlays/freeloader/alloy-values.yaml /tmp/alloy-render/freeloader-values.yaml
yq -i '.destinations.grafana-cloud-metrics.auth.password = "PLACEHOLDER"' /tmp/alloy-render/freeloader-values.yaml
yq -i '.destinations.grafana-cloud-logs.auth.password    = "PLACEHOLDER"' /tmp/alloy-render/freeloader-values.yaml
helm template alloy grafana/k8s-monitoring --version "4.1.4" \
  --namespace grafana \
  -f /tmp/alloy-render/freeloader-values.yaml > /tmp/alloy-render/freeloader.yaml
echo "Render OK, output at /tmp/alloy-render/freeloader.yaml"
```

Expected: exit code 0, no schema errors.

- [ ] **Step 3: Inspect rendered output for surprises**

```bash
grep -E 'kind: (Deployment|DaemonSet|StatefulSet)' /tmp/alloy-render/freeloader.yaml
```

Expected output names (the v4 chart deploys one collector per `collectors.*` entry):

```
kind: StatefulSet   # alloy-metrics
kind: DaemonSet     # alloy-daemonset
kind: Deployment    # alloy-singleton
```

If any are missing, the corresponding `collectors.<name>.enabled: true` entry is wrong or the preset list is malformed.

---

## Task 7: Phase 0 — commit base + both overlays, reconcile freeloader first

**Files:**
- Modify: `apps/grafana/base/helmrelease.yaml` (from Task 4)
- Modify: `apps/grafana/overlays/freeloader/alloy-values.yaml` (from Task 6)
- Modify: `apps/grafana/overlays/understairs/alloy-values.yaml` (from Task 5)

- [ ] **Step 1: Verify all overlays render via kustomize**

```bash
kubectl kustomize apps/grafana/overlays/understairs
kubectl kustomize apps/grafana/overlays/freeloader
```

Both should succeed. The generated `alloy-values` ConfigMap should contain the new map-keyed `destinations` structure.

- [ ] **Step 2: Commit**

```bash
git add apps/grafana/base/helmrelease.yaml \
        apps/grafana/overlays/freeloader/alloy-values.yaml \
        apps/grafana/overlays/understairs/alloy-values.yaml
git commit -m "feat(grafana): migrate k8s-monitoring chart from ~2.0 to ~4.1

Phase 0 of the cardinality optimization effort. Schema rewrite only;
metric selection and scrape interval unchanged.

- destinations changes from list to map (keyed by name)
- clusterMetrics.node-exporter -> hostMetrics.linuxHosts
- podLogs -> podLogsViaLoki
- alloy-{metrics,logs,singleton} top-level blocks moved under collectors.*
  with explicit presets
- understairs syslog receiver and OpenWRT scrape preserved under
  collectors.alloy-singleton

See docs/superpowers/specs/2026-06-09-grafana-cloud-cardinality-optimization-design.md"
git push origin main
```

- [ ] **Step 3: Reconcile freeloader first (clean slate — grafana.yaml is still commented out, so this only validates the prometheus-operator-crds release)**

Freeloader's Grafana app is not yet wired in — that happens in Task 8. The push here is purely so the next task can pull the new manifests. Skip Flux reconciliation for freeloader at this step.

- [ ] **Step 4: Reconcile understairs (it has `grafana.yaml` active)**

```bash
flux --context understairs reconcile source git platform
flux --context understairs reconcile kustomization grafana --with-source
flux --context understairs reconcile helmrelease alloy
```

Expected: HelmRelease shows `Reconciliation succeeded` after a minute or two. New collector pods appear in `grafana` namespace.

- [ ] **Step 5: Verify understairs collectors are running**

```bash
kubectl --context understairs -n grafana get pods
```

Expected: pods for `alloy-metrics-*` (StatefulSet), `alloy-daemonset-*` (one per node), and `alloy-singleton-*` (Deployment), all `Running`. The old `alloy-logs-*` daemonset name from v2 should be gone (Flux pruned it because the resource name changed).

- [ ] **Step 6: Verify the syslog port is still bound on alloy-singleton**

```bash
kubectl --context understairs -n grafana get svc -l app.kubernetes.io/name=alloy-singleton -o jsonpath='{.items[*].spec.ports[*].port}'
```

Expected output includes `1514`.

- [ ] **Step 7: Verify auth secret was injected via the new map-keyed path**

```bash
kubectl --context understairs -n grafana exec -it deploy/alloy-singleton -- /bin/alloy-config-cat 2>/dev/null \
  | grep -A 1 'prometheus.remote_write "grafana_cloud_metrics"' \
  || kubectl --context understairs -n grafana get cm -l app.kubernetes.io/name=alloy-metrics -o yaml | grep -c grafana_cloud_metrics
```

Expected: at least one match. If you see `username = "2355589"` followed by `password = ""`, the `valuesFrom` `targetPath` rewrite is broken — revisit Task 4 Step 1.

- [ ] **Step 8: Verify Grafana Cloud is receiving understairs metrics**

In Grafana Cloud Explore, run:

```
up{cluster="understairs"}
```

Expected: at least one series, returning `1`, within ~2 minutes of the HelmRelease being healthy. If empty, check Alloy logs:

```bash
kubectl --context understairs -n grafana logs statefulset/alloy-metrics --tail=200 | grep -iE 'error|denied|push'
```

- [ ] **Step 9: Observe for 24 hours before proceeding to Task 8**

Before enabling freeloader, let understairs run for at least 24 hours on the v4 chart with v2-equivalent behavior. Watch the active-series count in Grafana Cloud (Billing / Usage / Active Series). It should be roughly equivalent to the historical understairs cardinality before Alloy was disabled, plus or minus 10%. The exact number doesn't matter at this step — the goal is "no regression in *behavior*."

If the series count exceeds 18k on understairs alone (out of the 20k limit), do NOT proceed to Task 8 yet — jump to Task 9 (Phase 1 cuts) for understairs only, observe again, then return to Task 8.

---

## Task 8: Activate freeloader Grafana app

**Files:**
- Modify: `clusters/freeloader/kustomization.yaml`

- [ ] **Step 1: Uncomment `grafana.yaml`**

Modify `clusters/freeloader/kustomization.yaml` so `grafana.yaml` is enabled:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cert-manager.yaml
  - certificates.yaml
  - shared-secrets.yaml
  - external-secrets-operator.yaml
  - external-secrets.yaml
  - external-dns.yaml
  - prometheus-operator-crds.yaml
  # - project-contour.yaml
  - envoy-gateway.yaml
  - envoy-gateway-config.yaml
  - tailscale.yaml
  - tailscale-config.yaml
  - grafana.yaml
  # - argo-cd.yaml
  # - velero.yaml
  - cloudnative-pg.yaml
  # - local-path-provisioner.yaml
  - openebs-localpv.yaml
  # - retina.yaml
  # - longhorn.yaml
  # - podinfo.yaml
  - apps-bootstrap.yaml
```

- [ ] **Step 2: Verify the cluster kustomization renders**

```bash
kubectl kustomize clusters/freeloader
```

Expected: success; output now includes the `grafana` Flux Kustomization.

- [ ] **Step 3: Commit, push, reconcile**

```bash
git add clusters/freeloader/kustomization.yaml
git commit -m "feat(freeloader): re-enable grafana app

Now running on k8s-monitoring v4 schema after Phase 0 migration. The
prometheus-operator-crds chart from Task 1-3 owns the CRDs."
git push origin main
flux --context freeloader reconcile source git platform
flux --context freeloader reconcile kustomization grafana --with-source
flux --context freeloader reconcile helmrelease alloy
```

- [ ] **Step 4: Verify freeloader collectors are running**

```bash
kubectl --context freeloader -n grafana get pods
```

Expected: `alloy-metrics-*`, `alloy-daemonset-*`, `alloy-singleton-*`, all `Running`.

- [ ] **Step 5: Verify Grafana Cloud is receiving freeloader metrics**

In Grafana Cloud Explore:

```
up{cluster="freeloader"}
```

Expected: at least one series within ~2 minutes.

- [ ] **Step 6: Observe combined active series**

In Grafana Cloud Billing / Usage / Active Series, both `freeloader` and `understairs` should now contribute. Expected combined count (with no Phase 1 cuts yet): somewhere in the 15-25k range. If the limit is breached, Grafana Cloud's free tier will start dropping series — that's the trigger to proceed immediately to Task 9.

---

## Task 9: Phase 1 — apply cardinality cuts and log trims to understairs

**Files:**
- Modify: `apps/grafana/overlays/understairs/alloy-values.yaml`

- [ ] **Step 1: Replace `apps/grafana/overlays/understairs/alloy-values.yaml`**

Replace the file with the Phase 1 version. Changes from Task 5 are the metric tuning blocks under `clusterMetrics`, the addition of `clusterMetrics.kubelet`, `clusterMetrics.cadvisor`, and `clusterMetrics.apiserver`, the `metricsTuning` blocks under `prometheusOperatorObjects`, the bumped `scrapeInterval`, and the `excludeNamespaces` + `extraLogProcessingStages` under `podLogsViaLoki`:

```yaml
---
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
      useDefaultAllowList: true
  kubelet:
    metricsTuning:
      useDefaultAllowList: true
  cadvisor:
    metricsTuning:
      useDefaultAllowList: true
      excludeMetrics:
        - container_blkio_.*
        - container_tasks_state
        - container_memory_failures_total
        - container_fs_io_.*
  apiserver:
    enabled: true
    metricsTuning:
      useDefaultAllowList: true
      excludeMetrics:
        - apiserver_request_duration_seconds_bucket
        - apiserver_response_sizes_bucket
        - apiserver_watch_events_sizes_bucket
  controlPlane:
    enabled: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port

prometheusOperatorObjects:
  enabled: true
  collector: alloy-metrics
  serviceMonitors:
    excludeNamespaces: [kube-system]
  podMonitors:
    excludeNamespaces: [kube-system]
  metricsTuning:
    excludeMetrics:
      - .*_bucket
      - .*_sum

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

# Talos has no journald.

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

# -- Collectors --

collectors:
  alloy-metrics:
    enabled: true
    presets: [clustered, statefulset]
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
    tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
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

- [ ] **Step 2: Render and inspect**

```bash
cp apps/grafana/overlays/understairs/alloy-values.yaml /tmp/alloy-render/understairs-values.yaml
yq -i '.destinations.grafana-cloud-metrics.auth.password = "PLACEHOLDER"' /tmp/alloy-render/understairs-values.yaml
yq -i '.destinations.grafana-cloud-logs.auth.password    = "PLACEHOLDER"' /tmp/alloy-render/understairs-values.yaml
helm template alloy grafana/k8s-monitoring --version "4.1.4" --namespace grafana \
  -f /tmp/alloy-render/understairs-values.yaml > /tmp/alloy-render/understairs-phase1.yaml
echo "Phase 1 render OK"
```

Expected: exit 0. Compare against the Phase 0 render to verify the new `keep_metrics`/`drop_metrics` relabel blocks appear:

```bash
diff /tmp/alloy-render/understairs.yaml /tmp/alloy-render/understairs-phase1.yaml | head -80
```

The diff should show new relabel rules referencing `kube_pod_info`, `node_cpu_seconds_total`, etc. (the kube-state-metrics and node-exporter allowlists), and the exclude rules for the histograms.

- [ ] **Step 3: Do NOT commit yet**

Task 10 handles the freeloader equivalent so both clusters get Phase 1 in one commit.

---

## Task 10: Phase 1 — apply cardinality cuts and log trims to freeloader

**Files:**
- Modify: `apps/grafana/overlays/freeloader/alloy-values.yaml`

- [ ] **Step 1: Replace `apps/grafana/overlays/freeloader/alloy-values.yaml`**

```yaml
---
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
      useDefaultAllowList: true
  kubelet:
    metricsTuning:
      useDefaultAllowList: true
  cadvisor:
    metricsTuning:
      useDefaultAllowList: true
      excludeMetrics:
        - container_blkio_.*
        - container_tasks_state
        - container_memory_failures_total
        - container_fs_io_.*
  apiserver:
    enabled: true
    metricsTuning:
      useDefaultAllowList: true
      excludeMetrics:
        - apiserver_request_duration_seconds_bucket
        - apiserver_response_sizes_bucket
        - apiserver_watch_events_sizes_bucket
  controlPlane:
    enabled: false

hostMetrics:
  enabled: true
  collector: alloy-metrics
  linuxHosts:
    enabled: true
    metricsTuning:
      useIntegrationAllowList: true

annotationAutodiscovery:
  enabled: true
  collector: alloy-metrics
  annotations:
    scrape: prometheus.io/scrape
    metricsPath: prometheus.io/path
    metricsPortNumber: prometheus.io/port

prometheusOperatorObjects:
  enabled: true
  collector: alloy-metrics
  serviceMonitors:
    excludeNamespaces: [kube-system]
  podMonitors:
    excludeNamespaces: [kube-system]
  metricsTuning:
    excludeMetrics:
      - .*_bucket
      - .*_sum

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

- [ ] **Step 2: Render and inspect**

```bash
cp apps/grafana/overlays/freeloader/alloy-values.yaml /tmp/alloy-render/freeloader-values.yaml
yq -i '.destinations.grafana-cloud-metrics.auth.password = "PLACEHOLDER"' /tmp/alloy-render/freeloader-values.yaml
yq -i '.destinations.grafana-cloud-logs.auth.password    = "PLACEHOLDER"' /tmp/alloy-render/freeloader-values.yaml
helm template alloy grafana/k8s-monitoring --version "4.1.4" --namespace grafana \
  -f /tmp/alloy-render/freeloader-values.yaml > /tmp/alloy-render/freeloader-phase1.yaml
diff /tmp/alloy-render/freeloader.yaml /tmp/alloy-render/freeloader-phase1.yaml | head -80
```

Expected: same kinds of relabel rule additions as in Task 9 Step 2.

---

## Task 11: Phase 1 — commit, reconcile, observe

**Files:**
- Modify: `apps/grafana/overlays/understairs/alloy-values.yaml` (from Task 9)
- Modify: `apps/grafana/overlays/freeloader/alloy-values.yaml` (from Task 10)

- [ ] **Step 1: Commit**

```bash
git add apps/grafana/overlays/understairs/alloy-values.yaml \
        apps/grafana/overlays/freeloader/alloy-values.yaml
git commit -m "feat(grafana): apply cardinality cuts and log trims (Phase 1)

- kube-state-metrics, kubelet, cadvisor switched to useDefaultAllowList
- cadvisor: drop blkio, tasks_state, memory_failures, fs_io metrics
- apiserver enabled with default allowlist; drop bucket histograms
- prometheusOperatorObjects: exclude kube-system service/pod monitors;
  globally drop _bucket and _sum series from ServiceMonitor-discovered
  targets
- scrapeInterval 90s -> 120s
- podLogsViaLoki: drop kube-system/flux-system/velero namespaces;
  drop alloy and argocd-repo-server INFO lines
- freeloader nodeLogs: drop kubelet INFO

Expected combined active series: ~11-12k (vs ~20k cap).

See docs/superpowers/specs/2026-06-09-grafana-cloud-cardinality-optimization-design.md"
git push origin main
```

- [ ] **Step 2: Reconcile both clusters**

```bash
flux --context understairs reconcile source git platform
flux --context understairs reconcile helmrelease alloy
flux --context freeloader reconcile source git platform
flux --context freeloader reconcile helmrelease alloy
```

Expected: both HelmReleases reach `Reconciliation succeeded`; collector pods restart with new ConfigMaps.

- [ ] **Step 3: Verify pods restarted with new config**

```bash
kubectl --context understairs -n grafana get pods --sort-by=.metadata.creationTimestamp
kubectl --context freeloader -n grafana get pods --sort-by=.metadata.creationTimestamp
```

Expected: most pods have an AGE of less than a minute (fresh from the reconcile).

- [ ] **Step 4: Verify the active series target was met**

Wait one hour, then check Grafana Cloud Billing / Usage / Active Series.

- Target: combined ~11-12k.
- Acceptable: anywhere up to 16k (leaves ~4k headroom).
- Action required if >18k: identify top metric families and iterate, see Step 5.

Bookmark these PromQL queries in Grafana Cloud Explore:

```
count by (__name__)({cluster=~"freeloader|understairs"})
topk(20, count by (__name__)({cluster=~"freeloader|understairs"}))
```

- [ ] **Step 5: Iterate if needed**

If active series exceed 18k, the most likely culprits in priority order:
1. **A noisy ServiceMonitor not caught by `_bucket`/`_sum` exclusion.** Add the metric name to `prometheusOperatorObjects.metricsTuning.excludeMetrics`.
2. **An app exposing high-cardinality labels via annotation autodiscovery.** Either remove the `prometheus.io/scrape=true` annotation or add metric filters under `annotationAutodiscovery.metricsTuning`.
3. **cAdvisor `container_memory_*` or `container_cpu_*` over very ephemeral pods.** Add the specific metric to `clusterMetrics.cadvisor.metricsTuning.excludeMetrics`.

Apply incrementally and re-reconcile. Re-check after one hour each time.

- [ ] **Step 6: Final verification**

In Grafana Cloud Explore, run:

```
sum by (cluster) (up)
```

Expected: two clusters, both returning `> 0`. Confirms metrics are flowing from both, and the per-cluster `cluster` label is set correctly.

```
count by (namespace) ({cluster="understairs"} |~ "")
```

Run via Loki against the understairs cluster. Expected: no entries for `namespace="kube-system"`, `namespace="flux-system"`, `namespace="velero"`. The exclusion is working.

---

## Self-review notes

**Spec coverage check:**

- ✅ Spec Goal 1 (under 20k series): Tasks 9-11.
- ✅ Spec Goal 2 (chart upgrade to ~4.1, behavior preserved): Tasks 4-7.
- ✅ Spec Goal 3 (node + workload + control plane visibility): Task 9/10 includes `apiserver` enabled with allowlist.
- ✅ Spec Goal 4 (opt-in annotation contract): `annotationAutodiscovery.enabled: true` with annotations preserved in Tasks 5, 6, 9, 10.
- ✅ Spec Goal 5 (log volume trim): `excludeNamespaces` and `extraLogProcessingStages` in Tasks 9 and 10.
- ✅ Phase 0 mapping table (spec section): implemented in Tasks 5 and 6.
- ✅ Prometheus Operator CRD un-bundling mitigation: Tasks 1-3.
- ✅ Flux `valuesFrom` indexing fix: Task 4.
- ✅ Phase 1 metric cuts table: Tasks 9 and 10.
- ✅ Rollout sequence (CRDs → Phase 0 → Phase 1): Tasks 1-3 → 4-8 → 9-11.
- ✅ Pre-merge `helm template` dry-render: Task 5 Step 2, Task 6 Step 2, Task 9 Step 2, Task 10 Step 2.
- ✅ Post-deploy verification (pod state, /metrics queries): Task 7 Steps 5-8, Task 8 Steps 4-6, Task 11 Steps 3-6.
- ⚠️ One implementation detail deferred to runtime: the exact key for `extraLogProcessingStages`. If the v4 chart uses a different key (e.g. `extraStages` or `processing.stages`), substitute it during Task 9 Step 2 dry-render — the helm template error will name the correct key. The spec calls this out as "verified during implementation."

**Placeholder scan:** No `TBD`/`TODO`/"implement later" markers in step bodies. The single ambiguity around `extraLogProcessingStages` is explicitly flagged for runtime resolution rather than left silently unfilled.

**Type/path consistency:**

- `destinations.grafana-cloud-metrics` / `destinations.grafana-cloud-logs` keys used consistently across `helmrelease.yaml` (Task 4), both overlays (Tasks 5/6/9/10), and the Alloy `extraConfig` (`loki.write.grafana_cloud_logs.receiver`, `prometheus.remote_write.grafana_cloud_metrics.receiver`).
- Collector names `alloy-metrics`, `alloy-daemonset`, `alloy-singleton` used consistently as `collectors.*` keys and as `collector:` references on each feature.
- `presets` lists match the chart's example: `[clustered, statefulset]` for alloy-metrics, `[daemonset, filesystem-log-reader]` for alloy-daemonset, `[singleton]` for alloy-singleton.
