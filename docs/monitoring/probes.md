# Service probes (blackbox-exporter)

## Function

- Prometheus sends each target URL to blackbox-exporter.
- blackbox-exporter connects to the target and returns metrics.
- The probes use in-cluster service addresses. They do not use the internet, Cloudflare or Tailscale.
- UptimeRobot checks the public URLs from outside.

## Modules

| Module | Prober | Success |
| --- | --- | --- |
| `http_2xx` | HTTP | Status 2xx. Redirects are followed. |
| `http_2xx_or_401` | HTTP | Status 200 or 401. Use for services that need a login. |
| `tcp_connect` | TCP | The TCP connection opens. Use for non-HTTP services. |

- File: `monitoring/controllers/base/blackbox-exporter/release.yaml`.

## Targets

| Probe | Module | Targets |
| --- | --- | --- |
| `services-http` | `http_2xx` | actualbudget, home-app, home-blog (public and private), linkding, mealie, norish, Grafana |
| `services-http-auth` | `http_2xx_or_401` | obsidian-livesync CouchDB |
| `services-tcp` | `tcp_connect` | gizmosql |

- File: `monitoring/configs/staging/probes/probes.yaml`.

## Metrics

| Metric | Meaning |
| --- | --- |
| `probe_success` | 1 = target is up. 0 = target is down. |
| `probe_duration_seconds` | Total probe time |
| `probe_http_status_code` | HTTP status code |
| `probe_ssl_earliest_cert_expiry` | Unix time when the TLS certificate expires (HTTPS only) |

## Alerts

| Alert | Condition | Severity |
| --- | --- | --- |
| `ServiceDown` | `probe_success == 0` for 3 min | critical |
| `ServiceSlow` | Average probe time > 2 s for 10 min | warning |
| `BlackboxProbesMissing` | No `probe_success` metric for 10 min | warning |

- File: `monitoring/configs/staging/probes/rules.yaml`.

## Procedures

### Add a target

1. Find the service name, namespace and port: `kubectl get svc -A`.
2. Test the URL from a pod:

   ```sh
   kubectl run probe-test --rm -i --restart=Never --image=curlimages/curl -- \
     curl -sL -o /dev/null -w '%{http_code}\n' http://<svc>.<ns>.svc:<port>/
   ```

3. Select the module from the status code. 2xx → `http_2xx`. 401 → `http_2xx_or_401`.
4. Add the URL to the `static` list of the applicable Probe.
5. Commit and push. Flux applies the change in approximately 1 min.
6. Make sure that the target shows `probe_success = 1` in the Prometheus UI.

### Remove a target

- Remove the URL from the Probe. Do this before you remove the service. If not, `ServiceDown` starts.

### Test a probe manually

```sh
kubectl -n monitoring port-forward svc/blackbox-exporter 9115
curl 'http://localhost:9115/probe?module=http_2xx&target=http://mealie.mealie.svc:9000/&debug=true'
```

- `debug=true` shows each step of the probe.
