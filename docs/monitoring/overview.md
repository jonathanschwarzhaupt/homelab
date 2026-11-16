# Monitoring overview

## Layers

| Layer | Tool | Location | Finds |
| --- | --- | --- | --- |
| 1. Outside checks | UptimeRobot | Internet | Public URL down, node down, internet down, tunnel down |
| 2. Dead man's switch | healthchecks.io | Internet | Alertmanager or the full cluster stops |
| 3. Service probes | blackbox-exporter | Cluster | Service endpoint down or slow |
| 4. Cluster metrics and rules | kube-prometheus-stack | Cluster | Pods, nodes, disks, jobs, operators |

- Layers 1 and 2 run outside the cluster. They work when the node `alfie` is down.
- Layers 3 and 4 run on `alfie`. They stop when `alfie` stops.
- All in-cluster alerts go to the ntfy app on the phone.

## Data flow

```
targets (pods, nodes, services)
   │  scrape every 30–60 s
   ▼
Prometheus ──► evaluates PrometheusRules ──► Alertmanager ──► routes
   ▲                                                        ├─ ntfy (phone)
   │                                                        ├─ healthchecks.io (Watchdog)
blackbox-exporter (probes services)                         └─ null (discarded)
```

## Components

| Component | Purpose | Git path |
| --- | --- | --- |
| kube-prometheus-stack | Prometheus, Alertmanager, Grafana, operator, default rules | `monitoring/controllers/base/kube-prometheus-stack/` |
| Alertmanager config | Routes and receivers | `monitoring/controllers/staging/kube-prometheus-stack/release-overlay.yaml` |
| Notification secret | ntfy URL, healthchecks.io URL (1Password) | `monitoring/controllers/staging/kube-prometheus-stack/alertmanager-notifications-secret.yaml` |
| blackbox-exporter | HTTP and TCP probes | `monitoring/controllers/base/blackbox-exporter/` |
| Probes and probe rules | Probe targets, `ServiceDown` alerts | `monitoring/configs/staging/probes/` |
| Dashboards | Grafana dashboards as ConfigMaps | `monitoring/configs/staging/dashboards/` |

## Access

- Grafana: `https://monitoring.shrimp-algol.ts.net` (Tailscale).
- Prometheus UI: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090`, then open `http://localhost:9090`.
- Alertmanager UI: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093`, then open `http://localhost:9093`.

## k3s specific settings

- k3s runs the controller manager, scheduler and kube-proxy in one process.
- Prometheus cannot scrape them as separate targets.
- Thus `kubeControllerManager`, `kubeScheduler` and `kubeProxy` are disabled in the chart values.
- This also removes their `...Down` alerts, which were always active.
