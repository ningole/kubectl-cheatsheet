# Kubernetes CI Troubleshooting

Use this guide when a CI deployment or smoke test fails for a Pod, Deployment, Service, Ingress, or HPA. Start with read-only checks, use the intended cluster and namespace, and preserve command output in the job logs or artifacts. Never print Secret values or kubeconfig credentials in CI logs.

## Before investigating

Set names to match the resources created by the pipeline:

```bash
NAMESPACE=staging
APP=web
DEPLOYMENT=web
SERVICE=web
INGRESS=web
HPA=web

kubectl config current-context
kubectl get deployment,pods,service,ingress,hpa -n "$NAMESPACE"
kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp
```

Confirm the current context and namespace are the deployment target. Use an explicit `-n "$NAMESPACE"` on each namespaced command rather than relying on the context's default namespace.

## Pod is Pending, restarting, or failing

```bash
kubectl get pods -n "$NAMESPACE" -l app="$APP" -o wide
kubectl describe pods -n "$NAMESPACE" -l app="$APP"
kubectl logs -n "$NAMESPACE" -l app="$APP" --all-containers=true --tail=100
kubectl logs -n "$NAMESPACE" POD_NAME -c CONTAINER_NAME --previous --tail=100
kubectl get pvc -n "$NAMESPACE"
```

Inspect Pod conditions, container states, and Events:

| Symptom or event | Check |
| --- | --- |
| `Pending` / `FailedScheduling` | Available CPU and memory, node selectors, affinity, taints/tolerations, and PVC binding |
| `ImagePullBackOff` / `ErrImagePull` | Image name/tag, registry reachability, and pull-secret configuration (never print secret data) |
| `CrashLoopBackOff` | Current and previous container logs, command/arguments, configuration references, and probe failures |
| `OOMKilled` | Container memory limit and observed usage |
| `CreateContainerConfigError` | Referenced ConfigMaps, Secrets, volumes, and environment variable keys |
| Probe failures | Probe path/port, startup time, and application health endpoint |

Use the selector matching the actual Pod labels; `app="$APP"` is only an example.

## Deployment rollout is stuck or unavailable

```bash
kubectl rollout status deployment/"$DEPLOYMENT" \
  -n "$NAMESPACE" --timeout=2m
kubectl describe deployment "$DEPLOYMENT" -n "$NAMESPACE"
kubectl get replicasets -n "$NAMESPACE" -l app="$APP"
kubectl rollout history deployment/"$DEPLOYMENT" -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -l app="$APP" -o wide
```

Compare desired, updated, ready, and available replicas. Inspect ReplicaSet and Pod Events, image version, resource requests/limits, probes, and scheduling constraints. A rollout timeout should fail the CI job; collect diagnostics before deciding whether a rollback is appropriate.

## Service has no reachable backends

```bash
kubectl describe service "$SERVICE" -n "$NAMESPACE"
kubectl get service "$SERVICE" -n "$NAMESPACE" -o yaml
kubectl get pods -n "$NAMESPACE" --show-labels
kubectl get endpointslice -n "$NAMESPACE" \
  -l kubernetes.io/service-name="$SERVICE"
```

Verify the Service selector matches Pod labels, the Pods are Ready, and `port`/`targetPort` match the application. An empty EndpointSlice usually means there are no matching ready backends. If endpoints exist, check their ports and protocol, then test connectivity from an approved in-cluster test client.

## Ingress is not routing traffic

```bash
kubectl describe ingress "$INGRESS" -n "$NAMESPACE"
kubectl get ingress "$INGRESS" -n "$NAMESPACE" -o yaml
kubectl get ingressclass
kubectl get service "$SERVICE" -n "$NAMESPACE"
kubectl get endpointslice -n "$NAMESPACE" \
  -l kubernetes.io/service-name="$SERVICE"
kubectl get pods -A -l app.kubernetes.io/component=controller
```

Check the Ingress class, host, path/path type, TLS configuration, and backend Service name and port. Confirm the backend Service has endpoints and the ingress controller is healthy. Controller labels and log locations vary by implementation; identify the controller and inspect its Events/logs using the correct namespace and labels. If CI can reach the Ingress address, run an HTTP/TLS smoke test using the configured host.

## HPA has unknown metrics or will not scale

```bash
kubectl describe hpa "$HPA" -n "$NAMESPACE"
kubectl get hpa "$HPA" -n "$NAMESPACE" -o yaml
kubectl top pods -n "$NAMESPACE"
kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE" -o yaml
```

Check HPA Events, current versus desired replicas, target resource, and min/max bounds. `unknown` metrics can indicate Metrics Server or a custom metrics API issue. CPU/memory utilization targets require corresponding resource requests on the target containers. Verify the HPA targets the expected Deployment and that scaling is not constrained by another controller or policy.

## Capture diagnostics on CI failure

Use bounded waits so jobs cannot hang indefinitely. This example keeps the rollout failure as the job result while attempting best-effort diagnostics:

```bash
if ! kubectl rollout status deployment/"$DEPLOYMENT" \
  -n "$NAMESPACE" --timeout=2m; then
  kubectl get deployment,pods,service,ingress,hpa -n "$NAMESPACE" || true
  kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp || true
  kubectl describe deployment "$DEPLOYMENT" -n "$NAMESPACE" || true
  kubectl describe pods -n "$NAMESPACE" -l app="$APP" || true
  exit 1
fi
```

The `|| true` is limited to best-effort diagnostics, so a failed log collection cannot hide the rollout failure. Avoid patching, deleting, or rolling back resources automatically from generic troubleshooting steps; make those changes an explicit, reviewed pipeline action.
