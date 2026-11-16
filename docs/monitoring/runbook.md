# Monitoring runbook

## PrometheusOperatorSyncFailed: "statefulset already exists"

- **Symptom:**
  - Operator log: `creating statefulset failed: statefulsets.apps "alertmanager-..." already exists`.
  - `kubectl get alertmanager` shows `Reconciled=False`, reason `StatefulSetNotFound`.
  - Changes to the Alertmanager values have no effect.
- **Cause:**
  - The operator finds its StatefulSets by labels.
  - An old operator version copied the Helm labels to the StatefulSet. `app.kubernetes.io/managed-by` became `Helm`.
  - The operator does not find the StatefulSet. It tries to make a new one. Kubernetes refuses it.
- **Repair:**
  1. Compare `.status.selector` of the `Alertmanager` with the labels of the StatefulSet.
  2. Set the missing labels:

     ```sh
     kubectl -n monitoring label sts alertmanager-kube-prometheus-stack-alertmanager \
       app.kubernetes.io/managed-by=prometheus-operator \
       app.kubernetes.io/name=alertmanager \
       app.kubernetes.io/instance=kube-prometheus-stack-alertmanager \
       alertmanager=kube-prometheus-stack-alertmanager --overwrite
     ```

  3. Make sure that `Reconciled=True`: `kubectl -n monitoring get alertmanager -o yaml`.
- **Do not** delete the StatefulSet with `--cascade=orphan`. Flux does not make it again. Only the operator makes it, and the label fault stays.
- Occurred: 2026-02-14 to 2026-11-16.

## Alertmanager pod does not start after a change

- **Symptom:** Pod `alertmanager-...-0` is `ContainerCreating` or `CrashLoopBackOff`.
- **Possible causes:**
  - Secret `alertmanager-notifications` does not exist. Examine `kubectl -n monitoring get onepassworditem alertmanager-notifications -o yaml`.
  - The config is not valid. Examine the operator log and the `config-reloader` container log.
- **Repair:** Correct the 1Password item or the values. Validate with `amtool check-config` before you push.

## No ntfy messages

1. Send a test alert. See [alerting.md](alerting.md).
2. Examine the Alertmanager log: `kubectl -n monitoring logs alertmanager-kube-prometheus-stack-alertmanager-0 -c alertmanager | grep -i notify`.
3. Make sure that the phone subscribes to the same topic as in the 1Password item.
4. Make sure that the URL ends with `?template=alertmanager`.

## healthchecks.io reports "down"

- Alertmanager or Prometheus stopped, or the node is down.
- Examine in this sequence:
  1. UptimeRobot. If the public URLs are also down, the node or the internet connection is down.
  2. `kubectl get nodes`.
  3. `kubectl -n monitoring get pods`.

## ServiceDown

1. Find the target from the `instance` label.
2. Examine the pods: `kubectl -n <ns> get pods`.
3. Examine the pod events and logs: `kubectl -n <ns> describe pod <pod>`, `kubectl -n <ns> logs <pod>`.
4. Test the probe manually. See [probes.md](probes.md).

## Flux OCIRepository: "requires gzip-compressed body"

- **Symptom:** OCIRepository `Ready=False`: `failed to extract layer contents from artifact: requires gzip-compressed body: gzip: invalid header`.
- **Cause:** Flux unpacks the first layer of the OCI artifact. Some Helm charts put the provenance (`.prov`) layer first. Example: cert-manager 1.21.3.
- **Repair:** Select the chart layer:

  ```yaml
  spec:
    layerSelector:
      mediaType: "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
      operation: copy
  ```

- Occurred: cert-manager, 2026-11-12.
