# netapp-monitoring-operations

A Helm chart that deploys NetApp (and related storage) Prometheus alerting rules as
`PrometheusRule` resources, plus a `PodMonitor` that scrapes the NetApp Harvest
poller pods.

## Overview

The chart renders one `PrometheusRule` per alert file under `alerts/` and a
single `PodMonitor` for metric collection. Every rule group and every individual
alert can be toggled on or off through `values.yaml`, so you can tailor the alert
set to your environment without editing the rule files.

Covered components:

- **NetApp** — aggregate, cluster, disk, HA (storage failover), hardware, LUN,
  miscellaneous (key manager / SVM), network, NVDimm, volume, and a large set of
  EMS-derived event alerts.
- **Brocade** — switch error alerts. *(forge project — disabled by default)*
- **PowerScale** — error alerts. *(forge project — disabled by default)*
- **Pure Storage** — error alerts. *(forge project — disabled by default)*
- **Kubernetes** — PVC usage alerts. *(forge project — disabled by default)*
- **Harvest** — poller scrape-health alerts.

> **Scope note:** the Brocade, PowerScale, Pure Storage, and PVC usage rule groups
> belong to the **forge** project and are **not** part of the sci scope, so they are
> shipped **disabled by default**. Enable them by setting the relevant
> `prometheusRules.ruleGroups.*` keys to `true`.

## Prerequisites

- Helm 3+
- A Prometheus Operator deployment providing the `PrometheusRule` and `PodMonitor`
  CRDs (`monitoring.coreos.com/v1`).
- A Prometheus instance whose `ruleSelector` / `podMonitorSelector` matches the
  `labels` configured in `values.yaml` (default `prometheus: storage`).

## Installation

```bash
helm install netapp-monitoring-operations ./charts/netapp-monitoring-operations
```

Resources are created in the **release namespace** (`.Release.Namespace`). Use
`-n <namespace>` to control where they land:

```bash
helm install netapp-monitoring-operations ./charts/netapp-monitoring-operations -n storage-product
```

## Resource Naming

| Resource | Name |
|---|---|
| `PrometheusRule` | `<release>-<alert-file>` (e.g. `t-netapp-volume-alerts`) |
| `PodMonitor` | `<release>-harvest-pod-monitor` |

Both resources are created in `.Release.Namespace`.

## Configuration

### PrometheusRules

| Parameter | Description | Default |
|---|---|---|
| `prometheusRules.create` | Render the `PrometheusRule` resources | `true` |
| `prometheusRules.labels` | Labels applied to every `PrometheusRule`. `prometheus` must match the Prometheus `ruleSelector`. | `prometheus: storage` |
| `prometheusRules.annotations` | Annotations applied to every `PrometheusRule`. Falls back to `prometheus.io/alert: "true"` when unset. | `{}` |
| `prometheusRules.ruleGroups` | Map of `<fileKey>: true\|false` to enable/disable a whole alert file (one toggle per file). Must be a map; set to `{}` to enable all files. | NetApp + Harvest files `true`; Brocade/PowerScale/PureStorage/PVC (forge) `false` |
| `prometheusRules.commonLabels.support_group` | `support_group` label applied to every alert. | `storage` |
| `prometheusRules.commonLabels.team` | `team` label applied to every alert. | `sci-storage` |

### PodMonitor

| Parameter | Description | Default |
|---|---|---|
| `podMonitor.labels` | Labels applied to the `PodMonitor`. `prometheus` must match the Prometheus `podMonitorSelector`. | `prometheus: storage` |
| `podMonitor.targetNamespaces` | Namespace(s) where the target pods run. **Required.** | `[storage-product]` |
| `podMonitor.selectorMatchLabels` | Pod labels that uniquely select the target pods. Optional — when unset the `PodMonitor` selects **all** pods in `targetNamespaces`. | `ccloud/service: netapp-monitoring` |
| `podMonitor.port` | Container port name to scrape. | `metrics` |
| `podMonitor.path` | Metrics path. | `/metrics` |
| `podMonitor.scheme` | Scrape scheme. | `http` |
| `podMonitor.scrapeInterval` | Scrape interval. | `15s` |
| `podMonitor.jobLabel` | Pod label key whose value becomes the Prometheus `job` label. | `ccloud/service` |
| `podMonitor.relabelings` | Additional target relabelings. | `[]` |
| `podMonitor.additionalMetricRelabelings` | Metric relabelings appended after the `netapp_cluster` mapping. | `[]` |

### Required values and guards

- `podMonitor.targetNamespaces` is **required** — rendering fails with a clear
  message if it is unset, so the chart never produces a `PodMonitor` that scrapes
  all namespaces.
- `podMonitor.selectorMatchLabels` is **optional**. When omitted the `PodMonitor`
  has an empty pod selector and matches **all** pods in `targetNamespaces`; set it
  to uniquely select your target pods (and exclude others such as netapp-harvest).
- `prometheusRules.commonLabels.support_group` / `team` are **optional** and fall
  back to `storage` / `sci-storage` when `commonLabels` is omitted.

### Extra manifests

Render arbitrary Kubernetes objects alongside the chart's own resources by
listing them under `extraManifests`. Each entry is emitted verbatim and passed
through Helm's `tpl`, so template expressions inside the manifest (e.g.
`{{ .Release.Namespace }}`) are evaluated.

| Parameter | Description | Default |
|---|---|---|
| `extraManifests` | List of complete Kubernetes manifests rendered as-is. | `[]` |

A common use is an **owner-info** `ConfigMap` consumed by the Greenhouse
owner-info datasource:

```yaml
extraManifests:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: owner-of-storage-metrics-product
      namespace: storage-metrics
      labels:
        ccloud/support-group: storage
        ccloud/service: netapp-monitoring
        owner-info-version: "1.0.0"
      annotations:
        ccloud/support-group-datasource: owner-info
    data:
      support-group: storage
      service: netapp-monitoring
      helm-chart-url: https://github.com/cloudoperators/netapp-monitoring-operations
```

## Enabling / disabling alerts

Each alert file has a single on/off toggle (one key per file). Disable a whole
file by setting its key to `false`:

```yaml
prometheusRules:
  ruleGroups:
    pvcUsageAlerts: false
    netappEmsAlerts: false
```

The key is the camelCase form of the file name
(e.g. `brocade-error-alerts.yaml` → `brocadeErrorAlerts`,
`netapp_ems_alerts.yaml` → `netappEmsAlerts`).

When a file is disabled it renders as `groups: []` and produces an empty (but
valid) `PrometheusRule`.

## Chart Structure

```
charts/netapp-monitoring-operations/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── alerts.yaml
│   ├── extra-manifests.yaml
│   └── pod-monitor.yaml
└── alerts/
    ├── brocade-error-alerts.yaml
    ├── harvest-poller-alerts.yaml
    ├── netapp_aggr_alerts.yaml
    ├── netapp_cluster_alerts.yaml
    ├── netapp_disk_alerts.yaml
    ├── netapp_ems_alerts.yaml
    ├── netapp_ha_alerts.yaml
    ├── netapp_hw_alerts.yaml
    ├── netapp_lun_alerts.yaml
    ├── netapp_misc_alerts.yaml
    ├── netapp_network_alerts.yaml
    ├── netapp_nvdimm_alerts.yaml
    ├── netapp_volume_alerts.yaml
    ├── powerscale-error-alerts.yaml
    ├── purestorage-error-alerts.yaml
    └── pvc-usage-alerts.yaml
```

## Maintainers

| Name |
|---|
| Ganesh Kugulakrishnan |
| Chandrakanth Renduchintala |