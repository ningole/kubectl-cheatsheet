# Kubernetes Ingress Troubleshooting

Use this guide to diagnose an Ingress that is missing, unreachable, routing to the wrong backend, or returning HTTP/TLS errors. Examples use `kubectl` and Bash. Start with read-only checks and verify the current cluster before making changes.

## 1. Set the target and confirm access

Replace the sample values with the Ingress and backend used by your application:

```bash
NAMESPACE=staging
INGRESS=web
SERVICE=web
HOST=app.example.com
PATH_TO_TEST=/

kubectl config current-context
kubectl auth can-i get ingress -n "$NAMESPACE"
kubectl auth can-i get services -n "$NAMESPACE"
```

Confirm the context is the intended cluster. `Forbidden` or `no` from `kubectl auth can-i` means the operator needs the appropriate access; do not work around RBAC.

## 2. Trace the route from Ingress to Pods

```bash
kubectl get ingress "$INGRESS" -n "$NAMESPACE" -o wide
kubectl describe ingress "$INGRESS" -n "$NAMESPACE"
kubectl get ingress "$INGRESS" -n "$NAMESPACE" -o yaml
kubectl get ingressclass
kubectl get service "$SERVICE" -n "$NAMESPACE" -o yaml
kubectl get pods -n "$NAMESPACE" --show-labels
kubectl get endpointslice -n "$NAMESPACE" \
  -l kubernetes.io/service-name="$SERVICE" -o wide
kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp
```

Check each hop:

1. The Ingress has the expected host, path, path type, and `spec.ingressClassName`.
2. The referenced Service exists in the same namespace as the Ingress.
3. The backend port in the Ingress matches a Service `port` (not necessarily its `targetPort`).
4. The Service selector matches ready Pods, and EndpointSlices list backend addresses.
5. The Ingress controller for the selected class is running and has observed the Ingress.

IngressClass names and controller labels vary by cluster. Find the intended class in `kubectl get ingressclass -o yaml`, then identify the controller deployment and namespace from your cluster's installation documentation or platform team. Inspect its Pods, Events, and recent logs:

```bash
kubectl get deployments,pods,services -n "$CONTROLLER_NAMESPACE" -o wide
kubectl get events -n "$CONTROLLER_NAMESPACE" --sort-by=.metadata.creationTimestamp
kubectl logs -n "$CONTROLLER_NAMESPACE" "$CONTROLLER_POD" \
  --all-containers=true --since=15m
```

Set `CONTROLLER_NAMESPACE` and `CONTROLLER_POD` to the actual controller location and Pod name; do not assume a particular Ingress controller or namespace.

## 3. Check DNS, load balancer, and HTTP/TLS from the client

```bash
kubectl get ingress "$INGRESS" -n "$NAMESPACE" \
  -o jsonpath='{.status.loadBalancer.ingress[*].ip}{" "}{.status.loadBalancer.ingress[*].hostname}{"\n"}'
curl -v --connect-timeout 5 --max-time 15 "https://$HOST$PATH_TO_TEST"
```

Compare DNS resolution for `$HOST` with the Ingress load balancer address. If the Ingress has no address, check whether the controller or cloud load-balancer integration provisions one. If it has an address but clients cannot connect, check load-balancer health, listener ports, firewall/security-group rules, and network routes with the platform owner.

The HTTP response is a clue, not a diagnosis:

| Result | Likely area to inspect |
| --- | --- |
| DNS lookup failure or connection timeout | DNS record, load-balancer address/health, firewall, routing, or listener |
| Controller/default-backend `404` | Host/path mismatch, wrong IngressClass, or Ingress not accepted by controller |
| `502` / `503` | Missing/unready Service endpoints, wrong Service port, or controller-to-backend connectivity |
| `504` | Backend response time, application health, or network path from controller to backend |
| Certificate/name error | Ingress TLS configuration, certificate validity, or host/SAN mismatch |

If external access is unavailable, test from an approved in-cluster client: first call the Service DNS name and port, then compare with the external Ingress route. Avoid creating a debug Pod in production unless that is an approved operational procedure.

## 4. Check common configuration problems

### Wrong or missing IngressClass

Compare `spec.ingressClassName` on the Ingress with the intended class from `kubectl get ingressclass`. Older clusters may use the deprecated `kubernetes.io/ingress.class` annotation; follow the controller and cluster version's documented configuration.

Fix the manifest in the source of truth, then deploy it through the normal release/GitOps workflow. For an explicitly approved temporary hotfix only, setting the class can be done with:

```bash
INGRESS_CLASS=nginx
kubectl patch ingress "$INGRESS" -n "$NAMESPACE" --type=merge \
  -p "{\"spec\":{\"ingressClassName\":\"$INGRESS_CLASS\"}}"
```

Replace `nginx` with the actual class. This changes the live object and may be reverted by GitOps; commit the equivalent manifest change promptly.

### Ingress rule or backend mismatch

Compare the Ingress host/path and backend with the Service. A typical rule should reference the Service name and its declared Service port:

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Use the cluster's actual class, host, path, Service, and port. Correct the checked-in manifest rather than replacing the entire live Ingress with copied output from `kubectl get -o yaml`.

### Service has no ready endpoints

If the EndpointSlice is empty, compare the Service selector with Pod labels and check Pod readiness:

```bash
kubectl describe service "$SERVICE" -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" --show-labels
kubectl describe pods -n "$NAMESPACE"
```

Correct the Service selector or workload labels in the source manifest so they agree. If Pods are not Ready, fix their scheduling, startup, or readiness-probe issue; an Ingress change cannot make an unready backend healthy.

### TLS secret or hostname mismatch

The referenced TLS Secret must exist in the Ingress namespace and normally be type `kubernetes.io/tls`. Inspect only its metadata/type; do not print `.data`:

```bash
TLS_SECRET=web-tls
kubectl get ingress "$INGRESS" -n "$NAMESPACE" \
  -o jsonpath='{.spec.tls[*].secretName}{"\n"}'
kubectl get secret "$TLS_SECRET" -n "$NAMESPACE" \
  -o jsonpath='{.type}{"\n"}'
```

Verify the Secret name, certificate coverage for `$HOST`, validity dates, and renewal status using your approved certificate-management process. Do not copy private keys into terminal output or CI logs.

## 5. Verify the fix

After applying a reviewed manifest change through the normal deployment process:

```bash
kubectl get ingress "$INGRESS" -n "$NAMESPACE" -o wide
kubectl describe ingress "$INGRESS" -n "$NAMESPACE"
kubectl get endpointslice -n "$NAMESPACE" \
  -l kubernetes.io/service-name="$SERVICE" -o wide
curl -v --connect-timeout 5 --max-time 15 "https://$HOST$PATH_TO_TEST"
```

Confirm the expected DNS/address, host/path response, TLS certificate, and application response. If the change makes the issue worse, use the reviewed rollback procedure for the deployment/GitOps source rather than deleting the Ingress or restarting the controller as a first response.
