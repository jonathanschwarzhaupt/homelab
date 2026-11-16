# Alerting

## Routing

- Alertmanager receives alerts from Prometheus.
- It groups alerts by `alertname` and `namespace`.
- It sends each group to one receiver. The first route that matches wins.

| Order | Match | Receiver | Result |
| --- | --- | --- | --- |
| 1 | `alertname = Watchdog` | `heartbeat` | POST to healthchecks.io every minute |
| 2 | `alertname = InfoInhibitor` | `null` | Discarded |
| 3 | `severity = info` | `null` | Discarded |
| 4 | `severity = critical` | `ntfy-critical` | ntfy, priority 5 (urgent), tag 🚨 |
| – | all other alerts | `ntfy` | ntfy, priority 3 (default), tag ⚠️ |

- Timers:
  - `group_wait: 30s`: wait 30 s for more alerts of the same group before the first message.
  - `group_interval: 5m`: send changes of a group after 5 min at the earliest.
  - `repeat_interval: 12h`: send an alert again after 12 h while it is active.
- Inhibit rules (chart default):
  - A `critical` alert mutes `warning` and `info` alerts with the same `alertname` and `namespace`.
  - A `warning` alert mutes `info` alerts with the same `alertname` and `namespace`.
- Config file: `monitoring/controllers/staging/kube-prometheus-stack/release-overlay.yaml`.
- Helm merges this config with the chart default. Lists (`routes`, `receivers`) replace the default lists.

## ntfy

- Alertmanager has no ntfy receiver. It uses a `webhook_configs` receiver.
- The URL has the form `https://ntfy.sh/<topic>?template=alertmanager`.
- `template=alertmanager` tells ntfy to make a title and a message from the Alertmanager JSON.
- The headers `X-Priority` and `X-Tags` set the priority and the icon.
- The topic name is the password. Anyone who knows the topic can read and send messages.
  - Use a long random topic name.
  - Keep the full URL only in 1Password.

## Dead man's switch

- The `Watchdog` alert is always active. It proves that Prometheus and Alertmanager work.
- Alertmanager sends `Watchdog` to healthchecks.io every minute.
- healthchecks.io sends a notification when the pings stop.
- Recommended healthchecks.io settings: period 1 minute, grace time 5 minutes.
- Set the healthchecks.io notification to the same ntfy topic or to e-mail.

## Secret

- 1Password item: vault `apps`, item `alertmanager-notifications`.
- The 1Password operator makes the Kubernetes Secret `monitoring/alertmanager-notifications`.

| Field | Value |
| --- | --- |
| `ntfy-url` | `https://ntfy.sh/<topic>?template=alertmanager` |
| `healthchecks-url` | `https://hc-ping.com/<uuid>` |

- Alertmanager reads the files `/etc/alertmanager/secrets/alertmanager-notifications/<field>`.
- **Caution:** If the Secret does not exist, the Alertmanager pod cannot start.

## Procedures

### Send a test alert

1. Open a port-forward: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093`.
2. Send the alert:

   ```sh
   amtool alert add TestAlert severity=warning namespace=test \
     --annotation=summary="Test from amtool" \
     --alertmanager.url=http://localhost:9093
   ```

3. Make sure that the phone shows the message after approximately 30 s.
4. The test alert stops after 5 min (`resolve_timeout`). Then ntfy shows "Resolved".

- No `amtool` on the workstation? Run it in the pod:
  `kubectl -n monitoring exec alertmanager-kube-prometheus-stack-alertmanager-0 -c alertmanager -- amtool alert add ... --alertmanager.url=http://localhost:9093`.

### Silence an alert

- Use a silence for planned work. Example: 2 h for one alert:

  ```sh
  amtool silence add alertname=ServiceDown instance=~".*mealie.*" \
    --duration=2h --comment="mealie upgrade" \
    --alertmanager.url=http://localhost:9093
  ```

- Show silences: `amtool silence query`.
- Remove a silence: `amtool silence expire <id>`.

### Show active alerts

- `amtool alert query --alertmanager.url=http://localhost:9093`.
- `amtool alert query -o extended <alertname>` shows all labels.

### Test a route

- Make sure that an alert goes to the correct receiver:

  ```sh
  amtool config routes test --config.file=<file> alertname=ServiceDown severity=critical
  ```

### Disable a default alert

- Use the chart values. Example:

  ```yaml
  defaultRules:
    disabled:
      KubeHpaMaxedOut: true
  ```

### Add a new alert rule

1. Write a `PrometheusRule` in `monitoring/configs/staging/`.
2. Add the label `release: kube-prometheus-stack`. Prometheus ignores rules without this label.
3. Give each rule a `severity` label: `critical`, `warning` or `info`.
4. Give each rule `summary` and `description` annotations. ntfy shows them.
5. Test the expression in the Prometheus UI first.
6. Example: `monitoring/configs/staging/probes/rules.yaml`.

## Severity guide

| Severity | Use when | Phone |
| --- | --- | --- |
| `critical` | A service is down. Data loss is possible. Act now. | Urgent sound |
| `warning` | A fault will become a problem. Act this week. | Normal message |
| `info` | Information only. | No message |
