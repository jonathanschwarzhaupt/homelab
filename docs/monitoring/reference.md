# kube-prometheus-stack reference

## Components

| Component | Function |
| --- | --- |
| Prometheus operator | Reads the CRDs below. Makes the Prometheus and Alertmanager StatefulSets and their config. |
| Prometheus | Scrapes metrics. Stores them for 10 days (default). Evaluates rules. |
| Alertmanager | Groups, mutes and routes alerts to receivers. |
| Grafana | Shows dashboards. Uses Prometheus as data source. |
| kube-state-metrics | Makes metrics from Kubernetes objects (pods, jobs, deployments). |
| node-exporter | Makes metrics from the node (CPU, memory, disk, network). |

## Custom resources (CRDs)

| Kind | Function | Example |
| --- | --- | --- |
| `Prometheus` | One Prometheus server | Made by the chart |
| `Alertmanager` | One Alertmanager server | Made by the chart |
| `ServiceMonitor` | Scrape the pods behind a Service | CNPG, Flux, app metrics |
| `PodMonitor` | Scrape pods directly, without a Service | Pods with no Service |
| `Probe` | Probe targets through blackbox-exporter | `monitoring/configs/staging/probes/probes.yaml` |
| `PrometheusRule` | Alert rules and recording rules | `monitoring/configs/staging/probes/rules.yaml` |
| `AlertmanagerConfig` | Routes and receivers per namespace | Not used here |

## Selectors

- Prometheus loads only objects that match its selectors.
- In this cluster, each selector is `release: kube-prometheus-stack`.
- **Rule:** Add `labels: {release: kube-prometheus-stack}` to each `ServiceMonitor`, `PodMonitor`, `Probe` and `PrometheusRule`.
- Show the selectors:

  ```sh
  kubectl -n monitoring get prometheus -o yaml | grep -A2 -E 'Selector:'
  ```

- Other option: set `prometheus.prometheusSpec.<kind>SelectorNilUsesHelmValues: false`. Then Prometheus loads all objects.

## Who owns what

- Flux owns the HelmRelease.
- Helm owns the `Prometheus` and `Alertmanager` resources.
- The operator owns the StatefulSets. Flux and Helm do not make them.
- The operator finds its StatefulSets by labels: `app.kubernetes.io/managed-by=prometheus-operator`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`.
- If these labels are wrong, the operator does not find the StatefulSet. See [runbook.md](runbook.md).

## Alert rule anatomy

```yaml
- alert: ServiceDown                       # Name. Shown in ntfy.
  expr: probe_success{job="blackbox"} == 0  # PromQL. Each result series is one alert.
  for: 3m                                   # Must be true for 3 min. Prevents alerts from short faults.
  labels:
    severity: critical                      # Used by Alertmanager routes.
  annotations:
    summary: "{{ $labels.instance }} is down"   # Short text. Templates are allowed.
    description: "..."                           # Long text.
```

- Alert states: `inactive` → `pending` (expression true, `for` not complete) → `firing`.

## PromQL quick reference

| Query | Result |
| --- | --- |
| `up == 0` | Scrape targets that do not answer |
| `probe_success == 0` | Probe targets that are down |
| `rate(x_total[5m])` | Per-second rate of a counter over 5 min |
| `increase(x_total[1h])` | Increase of a counter in 1 h |
| `sum by (namespace) (...)` | Sum per namespace |
| `avg_over_time(x[10m])` | Average of a gauge over 10 min |
| `absent(x)` | 1 if the metric does not exist. Use to find missing data. |
| `predict_linear(x[6h], 4*3600)` | Value of `x` in 4 h, from a linear trend |
| `ALERTS{alertstate="firing"}` | All active alerts |

- Useful examples:
  - Container restarts in 1 h: `increase(kube_pod_container_status_restarts_total[1h]) > 0`.
  - Disk full in 24 h: `predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0`.
  - PVC usage: `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes`.
  - Last CNPG backup age: `time() - cnpg_collector_last_available_backup_timestamp`.

## Commands

| Task | Command |
| --- | --- |
| Show alert rules | `kubectl get prometheusrules -A` |
| Show probes | `kubectl get probes -A` |
| Show scrape targets | Prometheus UI → Status → Targets |
| Show the effective Alertmanager config | `kubectl -n monitoring get secret alertmanager-kube-prometheus-stack-alertmanager-generated -o jsonpath='{.data.alertmanager\.yaml\.gz}' \| base64 -d \| gunzip` |
| Show the reconcile status | `kubectl -n monitoring get alertmanager,prometheus -o wide` |
| Show operator errors | `kubectl -n monitoring logs deploy/kube-prometheus-stack-operator \| grep level=error` |
| Validate an Alertmanager config | `amtool check-config <file>` |
| Validate rules | `promtool check rules <file>` (rules only, not the CRD wrapper) |
| Force a Flux reconcile | `kubectl -n flux-system annotate kustomization monitoring-controllers reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite` |

## Good practice

- Alert on symptoms (service down, slow), not on causes (CPU high).
- Each `critical` alert must need an action now. If not, use `warning`.
- Use `for:` on each rule. This prevents alerts from short faults.
- Add a dead man's switch. A silent monitoring system is not a healthy system.
- Monitor from outside the cluster too. In-cluster monitoring stops with the cluster.
- Remove or tune alerts that are always active. They hide real alerts.
- Keep a runbook entry for each alert that occurred.
