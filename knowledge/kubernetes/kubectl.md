# kubectl Command Reference

Comprehensive documentation for kubectl commands.

**Official Documentation:** https://kubernetes.io/docs/reference/kubectl/

---

## Table of Contents

1. [Basic Commands](#basic-commands)
2. [Resource Management](#resource-management)
3. [Debugging](#debugging)
4. [Configuration](#configuration)
5. [Context Management](#context-management)
6. [Advanced Operations](#advanced-operations)

---

## Basic Commands

### Viewing Resources

```bash
# List resources
kubectl get pods
kubectl get pods -o wide                    # More details
kubectl get pods -o yaml                    # YAML output
kubectl get pods -o json                    # JSON output
kubectl get pods --show-labels              # Show labels
kubectl get pods -l app=nginx               # Filter by label
kubectl get pods --field-selector status.phase=Running
kubectl get pods --all-namespaces           # All namespaces
kubectl get pods -A                         # Short form

# Common resources
kubectl get nodes
kubectl get deployments
kubectl get services
kubectl get configmaps
kubectl get secrets
kubectl get ingress
kubectl get pvc
kubectl get events --sort-by='.lastTimestamp'

# Multiple resources
kubectl get pods,services,deployments

# Watch mode
kubectl get pods -w
kubectl get pods --watch
```

### Describing Resources

```bash
# Detailed info
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe deployment <deployment-name>
kubectl describe service <service-name>

# Events for troubleshooting
kubectl describe pod <pod-name> | grep -A 20 Events
```

### Creating Resources

```bash
# From file
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/              # Directory
kubectl apply -f https://example.com/manifest.yaml

# Create vs Apply
kubectl create -f deployment.yaml          # Fails if exists
kubectl apply -f deployment.yaml           # Creates or updates

# Quick create
kubectl create deployment nginx --image=nginx
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap my-config --from-literal=key=value
kubectl create secret generic my-secret --from-literal=password=secret

# Dry run
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server
```

### Deleting Resources

```bash
# Delete specific resource
kubectl delete pod <pod-name>
kubectl delete deployment <deployment-name>
kubectl delete -f deployment.yaml

# Delete multiple
kubectl delete pods pod1 pod2 pod3
kubectl delete pods -l app=nginx
kubectl delete pods --all

# Force delete
kubectl delete pod <pod-name> --force --grace-period=0

# Delete namespace (deletes all resources in it)
kubectl delete namespace <namespace>
```

---

## Resource Management

### Editing Resources

```bash
# Edit in editor
kubectl edit deployment <deployment-name>
kubectl edit configmap <configmap-name>

# Patch
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'
kubectl patch deployment nginx --type='json' -p='[{"op":"replace","path":"/spec/replicas","value":5}]'
```

### Scaling

```bash
# Scale deployment
kubectl scale deployment nginx --replicas=5

# Auto-scale
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Check HPA
kubectl get hpa
```

### Rolling Updates

```bash
# Update image
kubectl set image deployment/nginx nginx=nginx:1.25

# Check rollout status
kubectl rollout status deployment/nginx

# Rollout history
kubectl rollout history deployment/nginx

# Rollback
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# Restart
kubectl rollout restart deployment/nginx
```

### Labels and Annotations

```bash
# Add label
kubectl label pod <pod-name> env=production

# Update label
kubectl label pod <pod-name> env=staging --overwrite

# Remove label
kubectl label pod <pod-name> env-

# Add annotation
kubectl annotate pod <pod-name> description="My pod"
```

---

## Debugging

### Logs

```bash
# Pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>    # Specific container
kubectl logs <pod-name> --all-containers       # All containers

# Follow logs
kubectl logs -f <pod-name>

# Previous container logs (if crashed)
kubectl logs <pod-name> --previous

# Last N lines
kubectl logs <pod-name> --tail=100

# Since time
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since-time=2024-01-01T00:00:00Z

# Multiple pods
kubectl logs -l app=nginx
kubectl logs -l app=nginx --all-containers
```

### Exec into Container

```bash
# Run command
kubectl exec <pod-name> -- ls /app
kubectl exec <pod-name> -c <container> -- cat /etc/config

# Interactive shell
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/bash
```

### Port Forwarding

```bash
# Forward to pod
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward pod/<pod-name> 8080:80 &

# Forward to service
kubectl port-forward service/<service-name> 8080:80

# Forward to deployment
kubectl port-forward deployment/<deployment-name> 8080:80

# Listen on all interfaces
kubectl port-forward --address 0.0.0.0 pod/<pod-name> 8080:80
```

### Copy Files

```bash
# Copy to pod
kubectl cp ./local-file.txt <pod-name>:/path/to/file

# Copy from pod
kubectl cp <pod-name>:/path/to/file ./local-file.txt

# With container
kubectl cp ./file <pod-name>:/path -c <container>
```

### Debug Pods

```bash
# Run debug container
kubectl debug <pod-name> -it --image=busybox

# Copy pod with debug container
kubectl debug <pod-name> -it --image=busybox --copy-to=debug-pod

# Debug node
kubectl debug node/<node-name> -it --image=busybox
```

### Resource Usage

```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods
kubectl top pods --containers
kubectl top pods -l app=nginx
```

---

## Configuration

### ConfigMaps

```bash
# Create from literal
kubectl create configmap my-config \
  --from-literal=key1=value1 \
  --from-literal=key2=value2

# Create from file
kubectl create configmap my-config --from-file=config.properties
kubectl create configmap my-config --from-file=key=path/to/file

# Create from directory
kubectl create configmap my-config --from-file=./config-dir/

# Create from env file
kubectl create configmap my-config --from-env-file=.env

# View
kubectl get configmap my-config -o yaml
kubectl describe configmap my-config
```

### Secrets

```bash
# Create generic secret
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret

# Create from file
kubectl create secret generic my-secret --from-file=./credentials

# Create docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>

# Create TLS secret
kubectl create secret tls my-tls \
  --cert=path/to/cert.crt \
  --key=path/to/key.key

# View (base64 encoded)
kubectl get secret my-secret -o yaml

# Decode secret
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## Context Management

### Contexts and Clusters

```bash
# View config
kubectl config view
kubectl config view --minify    # Current context only

# List contexts
kubectl config get-contexts

# Current context
kubectl config current-context

# Switch context
kubectl config use-context <context-name>

# Set default namespace
kubectl config set-context --current --namespace=<namespace>

# Create new context
kubectl config set-context <context-name> \
  --cluster=<cluster> \
  --user=<user> \
  --namespace=<namespace>
```

### Namespaces

```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace <namespace>

# Set default namespace
kubectl config set-context --current --namespace=<namespace>

# Run command in namespace
kubectl get pods -n <namespace>
kubectl get pods --namespace=<namespace>
```

---

## Advanced Operations

### JSONPath Queries

```bash
# Get specific field
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get multiple fields
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
```

### Custom Columns

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# From file
kubectl get pods -o custom-columns-file=columns.txt
```

### Resource Diff

```bash
# Diff before apply
kubectl diff -f deployment.yaml
```

### Wait

```bash
# Wait for condition
kubectl wait --for=condition=ready pod/nginx --timeout=60s
kubectl wait --for=condition=available deployment/nginx --timeout=120s
kubectl wait --for=delete pod/nginx --timeout=60s
```

### API Resources

```bash
# List all API resources
kubectl api-resources

# List with specific API version
kubectl api-resources --api-group=apps

# Explain resource
kubectl explain pods
kubectl explain pods.spec.containers
kubectl explain deployment.spec.strategy
```

### Proxy

```bash
# Start API proxy
kubectl proxy

# Access API
curl http://localhost:8001/api/v1/namespaces/default/pods
```

### Run Temporary Pod

```bash
# Run and delete
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh

# Run with specific command
kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- curl http://service:80

# DNS lookup
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup kubernetes

# Network debugging
kubectl run tmp --image=nicolaka/netshoot --rm -it --restart=Never -- /bin/bash
```

---

## Quick Reference

### Common Flags

| Flag | Description |
|------|-------------|
| `-n, --namespace` | Specify namespace |
| `-A, --all-namespaces` | All namespaces |
| `-o, --output` | Output format |
| `-l, --selector` | Label selector |
| `-f, --filename` | File/directory |
| `--dry-run=client` | Preview changes |
| `-w, --watch` | Watch for changes |
| `-v` | Verbosity level |

### Output Formats

| Format | Description |
|--------|-------------|
| `-o wide` | Additional columns |
| `-o yaml` | YAML format |
| `-o json` | JSON format |
| `-o name` | Resource name only |
| `-o jsonpath` | JSONPath expression |
| `-o custom-columns` | Custom columns |

### Resource Shortcuts

| Shortcut | Resource |
|----------|----------|
| `po` | pods |
| `svc` | services |
| `deploy` | deployments |
| `ds` | daemonsets |
| `sts` | statefulsets |
| `cm` | configmaps |
| `ns` | namespaces |
| `no` | nodes |
| `pv` | persistentvolumes |
| `pvc` | persistentvolumeclaims |
| `ing` | ingresses |
| `rs` | replicasets |
| `hpa` | horizontalpodautoscalers |
