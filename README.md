# kubectl Cheatsheet

Common `kubectl` commands, from everyday basics to more advanced workflows. Replace example names, namespaces, images, and file paths with values from your cluster.

> **Caution:** Commands that change or delete resources affect the selected cluster. Check `kubectl config current-context` before running them.

## Basics

### Context and namespace

```bash
kubectl config get-contexts                 # List configured clusters
kubectl config current-context               # Show the selected context
kubectl config use-context my-cluster        # Select a context
kubectl get namespaces                       # List namespaces
kubectl config set-context --current --namespace=staging
```

### Inspect resources

```bash
kubectl get nodes                            # List cluster nodes
kubectl get pods                             # List pods in the current namespace
kubectl get pods -A                          # List pods across all namespaces
kubectl get deployments,services             # List deployments and services
kubectl get pods -o wide                     # Include node and IP details
kubectl describe pod my-pod                  # Show pod details and events
kubectl get pod my-pod -o yaml               # Print the pod manifest
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Create and apply

```bash
kubectl create deployment web --image=nginx  # Create a deployment
kubectl get deployment web                   # Check deployment status
kubectl expose deployment web --port=80      # Create a service for the deployment
kubectl apply -f ./manifests/                # Create or update resources from a directory
kubectl diff -f ./deployment.yaml            # Preview manifest changes
kubectl explain deployment.spec              # Inspect the API fields for a resource
```

### Logs and running containers

```bash
kubectl logs my-pod                          # Print pod logs
kubectl logs my-pod -c app                   # Print logs for a specific container
kubectl logs my-pod --previous               # Print logs from the previous container
kubectl logs -f deployment/web               # Follow deployment logs
kubectl exec my-pod -- printenv              # Run a command in a container
kubectl exec -it my-pod -- /bin/sh           # Open an interactive shell
kubectl port-forward service/web 8080:80     # Forward localhost:8080 to the service
```

## More advanced commands

### Filter, sort, and format output

```bash
kubectl get pods -n staging -l app=web        # Filter by namespace and label
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --sort-by=.metadata.name     # Sort by pod name
kubectl get pods -A --show-labels             # Show labels in every namespace
kubectl get pods -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,IMAGES:.spec.containers[*].image'
```

### Roll out and scale changes

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web         # Wait for the rollout
kubectl rollout history deployment/web        # Review rollout revisions
kubectl rollout undo deployment/web          # Roll back to the previous revision
kubectl scale deployment/web --replicas=3
kubectl autoscale deployment/web --min=2 --max=10 --cpu-percent=70
kubectl rollout restart deployment/web       # Restart pods managed by the deployment
```

### Patch and label resources

```bash
kubectl label pod my-pod tier=frontend
kubectl label pod my-pod tier-                 # Remove a label
kubectl annotate pod my-pod owner=platform
kubectl patch deployment/web \
  -p '{"spec":{"replicas":3}}'
kubectl patch deployment/web --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.27"}]'
```

### Troubleshoot and inspect cluster health

```bash
kubectl get pods -A -o wide                   # Find pod placement and IPs
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe node my-node                 # Inspect node conditions and events
kubectl top nodes                             # Show node resource usage
kubectl top pods -A --sort-by=memory          # Show pod memory usage
kubectl debug my-pod -it --image=busybox:1.36 # Start an ephemeral debug container
kubectl api-resources --namespaced=true       # List namespaced API resources
```

### Safely remove resources

```bash
kubectl delete -f ./manifests/                # Delete resources declared in the directory
kubectl delete pod my-pod -n staging          # Delete one pod
kubectl delete pods -l app=web -n staging     # Delete pods matching a label
```

Review the target namespace and selector before deleting resources. Deleting a pod managed by a Deployment usually causes the Deployment to create a replacement.

## Handy flags

| Flag | Purpose |
| --- | --- |
| `-n <namespace>` | Target one namespace |
| `-A` | Target all namespaces |
| `-l key=value` | Select resources by label |
| `-o yaml` / `-o json` | Choose structured output |
| `--dry-run=client -o yaml` | Generate a manifest without creating the resource |
| `-w` / `--watch` | Watch for changes |

## Reference

Based on the [Kubernetes kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/).
