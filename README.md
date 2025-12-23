[![Go Report Card](https://goreportcard.com/badge/github.com/mkihr/pvc-autoscaler)](https://goreportcard.com/report/github.com/mkihr/pvc-autoscaler)
[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/mkihr/pvc-autoscaler/go.yml?logo=go)](https://github.com/mkihr/pvc-autoscaler/actions)
[![GitHub release](https://img.shields.io/github/v/release/mkihr/pvc-autoscaler?logo=github)](https://github.com/mkihr/pvc-autoscaler/releases)
[![Helm Workflow Status](https://img.shields.io/github/actions/workflow/status/mkihr/pvc-autoscaler/helm-release.yaml?logo=helm&label=Helm)](https://github.com/mkihr/pvc-autoscaler/actions)
![GitHub release (with filter)](https://img.shields.io/github/v/release/mkihr/pvc-autoscaler?filter=pvcautoscaler-*&logo=Helm&label=Helm%20release)
![GitHub](https://img.shields.io/github/license/mkihr/pvc-autoscaler)

# PVC autoscaler for Kubernetes

PVC Autoscaler is an open-source project aimed at providing autoscaling functionality to Persistent Volume Claims (PVCs) in Kubernetes environments. It allows you to automatically scale your PVCs based on your workloads and the metrics collected.

Please note that PVC Autoscaler is currently in a heavy development phase. As such, it's not recommended for production usage at this point.

## Table of Contents

- [Motivation](#motivation)
- [Features](#features)
- [How it works](#how-it-works)
- [Limitations](#limitations)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Standard Installation](#standard-installation)
  - [OpenShift Installation](#openshift-installation)
- [Configuration Reference](#configuration-reference)
  - [CLI Flags](#cli-flags)
  - [PVC Annotations](#pvc-annotations)
  - [Helm Values](#helm-values)
- [Usage](#usage)
  - [Basic Example (AWS EKS)](#basic-example-aws-eks)
  - [Minimal Configuration](#minimal-configuration)
  - [Azure AKS Example](#azure-aks-example)
  - [Google GKE Example](#google-gke-example)
- [OpenShift Specific Configuration](#openshift-specific-configuration)
- [Architecture](#architecture)
- [Troubleshooting](#troubleshooting)
- [Monitoring and Observability](#monitoring-and-observability)
- [FAQ](#faq)
- [Development](#development)
- [Security Considerations](#security-considerations)
- [Upgrade Guide](#upgrade-guide)
- [Contributions](#contributions)
- [License](#license)

## Motivation

The motivation behind the PVC Autoscaler project is to provide developers with an easy and efficient way of managing storage resources within their Kubernetes clusters: sometimes is difficult to estimate how much storage an application needs. With the PVC Autoscaler, there's no need to manually adjust the size of your PVCs as your storage needs change. The Autoscaler handles this for you, freeing you up to focus on other areas of your development work.

## Features

- **Automatic PVC Scaling**: Monitors volume usage and automatically expands PVCs when thresholds are exceeded
- **Prometheus Integration**: Leverages existing Prometheus metrics for accurate volume usage tracking
- **Flexible Configuration**: Customizable thresholds, increase percentages, and ceiling limits per PVC
- **Safety Features**:
  - Prevents resize loops with capacity tracking
  - Validates StorageClass expansion capability
  - Respects ceiling limits to prevent runaway scaling
  - Only processes bound PVCs with Filesystem mode
- **Multi-Cloud Support**: Works with any Kubernetes cluster and CSI driver supporting volume expansion (AWS EKS, Azure AKS, Google GKE, etc.)
- **OpenShift Compatible**: Built-in support for OpenShift with bearer token authentication
- **Lightweight**: Minimal resource footprint (default: 10m CPU, 20Mi memory)
- **Easy Deployment**: Simple Helm chart installation with sensible defaults

## How it works

![pvc-autoscaler-architecture](https://raw.githubusercontent.com/mkihr/pvc-autoscaler/main/docs/pvc-autoscaler-architecture.svg)

The PVC Autoscaler operates as a Kubernetes controller with the following workflow:

1. **Polling Loop**: Runs at a configurable interval (default: 30 seconds)
2. **PVC Discovery**: Fetches all PVCs with the `pvc-autoscaler.mkihr.io/enabled: "true"` annotation
3. **Metrics Collection**: Queries Prometheus for `kubelet_volume_stats_used_bytes` and `kubelet_volume_stats_capacity_bytes`
4. **Validation**: Ensures the PVC is bound, uses Filesystem mode, and has an expandable StorageClass
5. **Threshold Check**: Compares current usage against the configured threshold percentage
6. **Resize Decision**: If usage exceeds threshold:
   - Calculates new size based on increase percentage
   - Rounds up to the nearest GiB
   - Caps at ceiling limit if specified
   - Updates PVC spec and sets `previous_capacity` annotation to prevent repeated attempts
7. **Safety Tracking**: Uses `previous_capacity` annotation to avoid resize loops while expansion is in progress

## Limitations

- Currently only supports Prometheus for collecting metrics
- Only Filesystem volume mode is supported (Block storage volumes are not supported)
- Requires CSI driver with VolumeExpansion capability
- Cannot shrink volumes (Kubernetes limitation)

## Requirements

Before installing PVC Autoscaler, ensure you have:

- [ ] Managed Kubernetes cluster (EKS, AKS, GKE, OpenShift, etc.) or self-managed cluster with CSI support
- [ ] CSI driver that supports [`VolumeExpansion`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#csi-volume-expansion)
- [ ] StorageClass(es) with `allowVolumeExpansion: true`
- [ ] PVCs with `volumeMode: Filesystem` (default)
- [ ] Prometheus collecting kubelet volume metrics (`kubelet_volume_stats_used_bytes` and `kubelet_volume_stats_capacity_bytes`)
- [ ] Network connectivity from PVC Autoscaler to Prometheus endpoint

## Installation

PVC Autoscaler comes with a Helm chart for easy deployment in a Kubernetes cluster.

### Standard Installation

First add the Helm repository:

```console
helm repo add pvc-autoscaler https://mkihr.github.io/pvc-autoscaler
helm repo update
```

Install the chart (adopt values to your needs):

```console
helm install pvc-autoscaler pvc-autoscaler/pvcautoscaler -n kube-system \
  --set pvcAutoscaler.args.metricsClientURL=http://prometheus-operated.prom.svc.cluster.local:9090
```

Replace the `metricsClientURL` with your Prometheus service endpoint.

### OpenShift Installation

For OpenShift environments with Thanos Querier, use the OpenShift-specific values:

```console
helm install pvc-autoscaler pvc-autoscaler/pvcautoscaler -n kube-system \
  --set pvcAutoscaler.args.metricsClientURL=https://thanos-querier.openshift-monitoring.svc.cluster.local:9091 \
  --set pvcAutoscaler.args.bearerTokenFile=/var/run/secrets/kubernetes.io/serviceaccount/token \
  --set pvcAutoscaler.args.insecureSkipVerify=true \
  --set openshift.enabled=true
```

Or use the pre-configured OpenShift values file:

```console
helm install pvc-autoscaler ./charts/pvc-autoscaler \
  -f charts/pvc-autoscaler/values-openshift.yaml -n kube-system
```

## Configuration Reference

### CLI Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--metrics-client` | string | `prometheus` | Metrics client type to use |
| `--metrics-client-url` | string | (required) | Metrics client URL to query volume stats |
| `--polling-interval` | duration | `30s` | How often to check PVC stats |
| `--reconcile-timeout` | duration | `1m` | Timeout for reconciliation loop |
| `--log-level` | string | `INFO` | Log level (`INFO` or `DEBUG`) |
| `--insecure-skip-verify` | bool | `false` | Skip TLS certificate verification |
| `--bearer-token-file` | string | (empty) | Path to bearer token file for authentication |

### PVC Annotations

All annotations use the prefix `pvc-autoscaler.mkihr.io/`:

| Annotation | Type | Default | Description | Example |
|------------|------|---------|-------------|---------|
| `enabled` | string | (required) | Enable autoscaling for this PVC | `"true"` |
| `threshold` | percentage | `80%` | Usage percentage to trigger resize | `"85%"` |
| `increase` | percentage | `20%` | How much to increase size by | `"25%"` |
| `ceiling` | quantity | (none) | Maximum PVC size limit | `"100Gi"` |
| `previous_capacity` | quantity | (internal) | Tracks previous capacity (managed by controller) | `"10Gi"` |

### Helm Values

Key values from `values.yaml`:

```yaml
image:
  repository: mkihr1/pvc-autoscaler
  tag: "latest"
  pullPolicy: Always

pvcAutoscaler:
  args:
    metricsClient: "prometheus"
    metricsClientURL: http://prometheus-operated.prom.svc.cluster.local:9090
    pollingInterval: 30s
    reconcileTimeout: 30s
    insecureSkipVerify: false
    bearerTokenFile: ""
    logger:
      logLevel: "INFO"

  resources:
    requestCPU: "10m"
    requestMemory: "20Mi"

openshift:
  enabled: false  # Enable for OpenShift-specific RBAC
```

See the full [values.yaml](charts/pvc-autoscaler/values.yaml) for all options.

## Usage

### Basic Example (AWS EKS)

Using `pvc-autoscaler` requires a `StorageClass` that allows volume expansion, i.e. with the `allowVolumeExpansion` field set to `true`. In case of `EKS` you can define:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-expandable
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Then set up the `PersistentVolumeClaim` based on the following example:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pvc
  annotations:
    pvc-autoscaler.mkihr.io/enabled: "true"
    pvc-autoscaler.mkihr.io/threshold: 80%
    pvc-autoscaler.mkihr.io/ceiling: 20Gi
    pvc-autoscaler.mkihr.io/increase: 20%
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-expandable
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
```

Configuration notes:
* Set `spec.storageClassName` to the name of the expandable `StorageClass` defined above
* Make sure `spec.volumeMode` is set to `Filesystem` (if you have a block storage this won't work)
* Set `metadata.annotations.pvc-autoscaler.mkihr.io/enabled` to `"true"` to enable autoscaling
* The `threshold` annotation fixes the volume usage above which the resizing will be triggered (default: 80%)
* Set how much to increase via the `increase` annotation (default 20%)
* To avoid infinite scaling you can set a maximum size for your volume via the `ceiling` annotation

### Minimal Configuration

For a minimal setup using all defaults (80% threshold, 20% increase, no ceiling):

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-minimal-pvc
  annotations:
    pvc-autoscaler.mkihr.io/enabled: "true"
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-expandable
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
```

### Azure AKS Example

For Azure AKS with managed disks:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-expandable
provisioner: disk.csi.azure.com
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed
reclaimPolicy: Delete
allowVolumeExpansion: true
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: azure-pvc
  annotations:
    pvc-autoscaler.mkihr.io/enabled: "true"
    pvc-autoscaler.mkihr.io/threshold: 75%
    pvc-autoscaler.mkihr.io/ceiling: 50Gi
    pvc-autoscaler.mkihr.io/increase: 25%
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium-expandable
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
```

### Google GKE Example

For Google GKE with Persistent Disk:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd-expandable
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gke-pvc
  annotations:
    pvc-autoscaler.mkihr.io/enabled: "true"
    pvc-autoscaler.mkihr.io/threshold: 85%
    pvc-autoscaler.mkihr.io/ceiling: 100Gi
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pd-ssd-expandable
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi
```

## OpenShift Specific Configuration

OpenShift environments typically use Thanos Querier for Prometheus metrics. Here's the recommended configuration:

### 1. Install with OpenShift values

```bash
helm install pvc-autoscaler pvc-autoscaler/pvcautoscaler \
  --set openshift.enabled=true \
  --set pvcAutoscaler.args.metricsClientURL=https://thanos-querier.openshift-monitoring.svc.cluster.local:9091 \
  --set pvcAutoscaler.args.bearerTokenFile=/var/run/secrets/kubernetes.io/serviceaccount/token \
  --set pvcAutoscaler.args.insecureSkipVerify=true \
  -n kube-system
```

### 2. Required RBAC

When `openshift.enabled=true`, the Helm chart automatically creates:
- Standard ClusterRole for PVC and StorageClass access
- Additional ClusterRoleBinding to `cluster-monitoring-view` role for Prometheus access

### 3. Bearer Token Authentication

The controller uses the ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` to authenticate with Thanos Querier.

### 4. TLS Configuration

If using self-signed certificates (common in OpenShift):
```bash
--set pvcAutoscaler.args.insecureSkipVerify=true
```

## Architecture

The PVC Autoscaler consists of the following components:

### Controller Components

1. **Main Controller** (`cmd/main.go`)
   - Initializes Kubernetes and metrics clients
   - Runs a ticker-based polling loop
   - Handles configuration via CLI flags

2. **Reconciliation Loop** (`cmd/reconcile.go`)
   - Fetches all annotated PVCs
   - Queries metrics for each PVC
   - Validates expansion eligibility
   - Calculates and applies new sizes

3. **Metrics Client Layer** (`internal/metrics_clients/`)
   - Abstraction interface for metrics providers
   - Prometheus implementation with bearer token support
   - Queries kubelet volume stats

4. **Kubernetes Client** (`cmd/kubeclient.go`)
   - In-cluster authentication
   - PVC and StorageClass operations

### Data Flow

```
┌─────────────────┐
│   Controller    │
│   (Ticker Loop) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────┐
│  Reconcile Loop │─────▶│  Kubernetes  │
└────────┬────────┘      │     API      │
         │               └──────────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────┐
│ Metrics Client  │─────▶│  Prometheus  │
└─────────────────┘      └──────────────┘
```

### Resize Decision Logic

```
1. Is PVC annotated with enabled=true? → No: Skip
                                      → Yes: Continue
2. Is StorageClass expandable? → No: Skip
                                → Yes: Continue
3. Is PVC in Bound state? → No: Skip
                           → Yes: Continue
4. Is volumeMode Filesystem? → No: Skip
                              → Yes: Continue
5. Are metrics available? → No: Log warning, skip
                          → Yes: Continue
6. Is usage ≥ threshold? → No: Skip
                         → Yes: Calculate new size
7. Has capacity changed since last resize? → No: Skip (resize in progress)
                                            → Yes: Apply resize
8. Is new size ≤ ceiling? → No: Cap at ceiling
                          → Yes: Use calculated size
9. Update PVC spec and previous_capacity annotation
```

## Troubleshooting

### PVC Not Scaling

**Symptoms**: PVC usage is above threshold but no resize occurs.

**Checks**:

1. **Verify PVC annotations**:
   ```bash
   kubectl get pvc <pvc-name> -o yaml | grep -A 5 annotations
   ```
   Ensure `pvc-autoscaler.mkihr.io/enabled: "true"` is set.

2. **Check StorageClass**:
   ```bash
   kubectl get storageclass <class-name> -o yaml | grep allowVolumeExpansion
   ```
   Must be `allowVolumeExpansion: true`.

3. **Verify PVC is Bound**:
   ```bash
   kubectl get pvc <pvc-name>
   ```
   Status must be `Bound`.

4. **Check controller logs**:
   ```bash
   kubectl logs -n kube-system deployment/pvc-autoscaler
   ```
   Look for errors related to the PVC.

5. **Enable debug logging**:
   ```bash
   helm upgrade pvc-autoscaler pvc-autoscaler/pvcautoscaler \
     --set pvcAutoscaler.args.logger.logLevel=DEBUG \
     --reuse-values
   ```

### Metrics Not Available

**Symptoms**: Logs show "metrics not available" or "metriken wieder verfügbar" messages.

**Checks**:

1. **Test Prometheus connectivity**:
   ```bash
   kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
     curl http://prometheus-operated.prom.svc.cluster.local:9090/api/v1/query?query=up
   ```

2. **Verify kubelet metrics exist**:
   ```bash
   # Query for your PVC's metrics
   curl 'http://prometheus:9090/api/v1/query?query=kubelet_volume_stats_used_bytes{persistentvolumeclaim="my-pvc"}'
   ```

3. **For OpenShift, verify bearer token**:
   ```bash
   kubectl exec -n kube-system deployment/pvc-autoscaler -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
   ```

4. **Check network policies**: Ensure PVC Autoscaler pod can reach Prometheus.

### Resize Stuck in Progress

**Symptoms**: PVC shows increased `spec.resources.requests.storage` but `status.capacity` hasn't updated.

**Explanation**: This is normal Kubernetes behavior. The resize is handled by the CSI driver and may take time.

**Checks**:

1. **Check PVC conditions**:
   ```bash
   kubectl describe pvc <pvc-name>
   ```
   Look for `FileSystemResizePending` or `Resizing` conditions.

2. **Check pod using the PVC**: Some CSI drivers require pod restart:
   ```bash
   kubectl delete pod <pod-using-pvc>
   ```

3. **Check CSI driver logs**: Look for resize operations in CSI driver pods.

### Permission Errors

**Symptoms**: Logs show 403 Forbidden or RBAC errors.

**Checks**:

1. **Verify ClusterRole**:
   ```bash
   kubectl get clusterrole pvc-autoscaler -o yaml
   ```

2. **Verify ClusterRoleBinding**:
   ```bash
   kubectl get clusterrolebinding pvc-autoscaler -o yaml
   ```

3. **For OpenShift, verify Prometheus RBAC**:
   ```bash
   kubectl get clusterrolebinding pvc-autoscaler-prometheus -o yaml
   ```

### Ceiling Reached

**Symptoms**: PVC stops growing even though usage is high.

**Check**: Review the `ceiling` annotation:
```bash
kubectl get pvc <pvc-name> -o jsonpath='{.metadata.annotations.pvc-autoscaler\.mkihr\.io/ceiling}'
```

**Solution**: Increase or remove the ceiling annotation:
```bash
kubectl annotate pvc <pvc-name> pvc-autoscaler.mkihr.io/ceiling=100Gi --overwrite
```

## Monitoring and Observability

### Verify Autoscaler is Running

```bash
kubectl get pods -n kube-system -l app=pvc-autoscaler
kubectl logs -n kube-system -l app=pvc-autoscaler --tail=50
```

Expected logs:
```
INFO pvc-autoscaler mkihr version
INFO Build tag: 1.0.1
INFO kubernetes client ready
INFO metrics client (prometheus) ready at address http://prometheus:9090
INFO pvc-autoscaler ready
```

### Monitor PVC Resizes

Watch for resize events:
```bash
kubectl get events --sort-by='.lastTimestamp' | grep ExpandVolume
```

Check PVC history:
```bash
kubectl describe pvc <pvc-name> | grep -A 10 Events
```

### Track PVC Growth Over Time

Query Prometheus for PVC capacity changes:
```promql
kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="my-pvc"}
```

Query for usage percentage:
```promql
100 * kubelet_volume_stats_used_bytes{persistentvolumeclaim="my-pvc"}
/ kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="my-pvc"}
```

### Recommended Alerts

Set up Prometheus alerts for:

1. **High usage not scaling**:
   ```yaml
   - alert: PVCHighUsageNotScaling
     expr: |
       (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.90
     for: 15m
     annotations:
       summary: "PVC {{ $labels.persistentvolumeclaim }} is at {{ $value }}% and may not be autoscaling"
   ```

2. **Ceiling reached**:
   Monitor PVCs that have hit their ceiling limit (requires custom metric from controller - not currently implemented).

## FAQ

### Q: How often does the controller check PVC usage?

**A**: By default, every 30 seconds (configurable via `--polling-interval` flag).

### Q: What happens if my PVC hits the ceiling limit?

**A**: The controller will stop resizing the PVC. You'll need to manually increase the ceiling annotation or remove it to allow further growth.

### Q: Can I temporarily disable autoscaling for a PVC?

**A**: Yes, set the annotation to `pvc-autoscaler.mkihr.io/enabled: "false"` or remove it entirely.

```bash
kubectl annotate pvc <pvc-name> pvc-autoscaler.mkihr.io/enabled=false --overwrite
```

### Q: Why is my PVC not shrinking when usage decreases?

**A**: Kubernetes does not support volume shrinking. This is a platform limitation, not a controller limitation.

### Q: What happens if Prometheus is temporarily unavailable?

**A**: The controller will log a warning and skip that reconciliation cycle. It will retry on the next polling interval. Once Prometheus is available again, it will log "metriken wieder verfügbar" (metrics available again).

### Q: Can I use different thresholds for different PVCs?

**A**: Yes, each PVC can have its own `threshold`, `increase`, and `ceiling` annotations.

### Q: Does the controller support multiple Prometheus instances?

**A**: No, currently only one metrics endpoint can be configured per controller instance.

### Q: What's the minimum increase size?

**A**: Sizes are rounded up to the nearest GiB (1073741824 bytes). The minimum practical increase is 1Gi.

### Q: Can I scale PVCs in specific namespaces only?

**A**: Currently, no. The controller watches all namespaces. You control which PVCs are scaled using the `enabled` annotation.

### Q: Does this work with StatefulSets?

**A**: Yes, it works with any PVC regardless of how it's created (manual, Deployment, StatefulSet, etc.).

### Q: What permissions does the controller need?

**A**: The controller needs:
- `get`, `list`, `update`, `patch` on PVCs cluster-wide
- `get`, `list` on StorageClasses
- For OpenShift: `cluster-monitoring-view` role to access Prometheus

## Development

### Prerequisites

- Go 1.22 or later
- Docker (for image builds)
- Kubernetes cluster (for testing)
- Make

### Building

```bash
# Format code
make fmt

# Run linter
make vet

# Run tests
make test

# Run tests with coverage
make cov

# View coverage in browser
make cov-html

# Build binary
make build
# Output: bin/pvc-autoscaler
```

### Running Tests

```bash
# All tests
go test ./...

# Single package with verbose output
go test ./internal/metrics_clients/prometheus -v

# With coverage
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

### Docker

```bash
# Build image
make docker-build

# Build with custom image name
make docker-build IMG=myrepo/pvc-autoscaler:dev

# Push image
make docker-push IMG=myrepo/pvc-autoscaler:dev

# Run container
make docker-run IMG=myrepo/pvc-autoscaler:dev
```

### Helm Chart Development

Chart location: `charts/pvc-autoscaler/`

```bash
# Lint chart
helm lint charts/pvc-autoscaler/

# Template chart (dry-run)
helm template pvc-autoscaler charts/pvc-autoscaler/ \
  --set pvcAutoscaler.args.metricsClientURL=http://prometheus:9090

# Install from local chart
helm install pvc-autoscaler charts/pvc-autoscaler/ \
  --set pvcAutoscaler.args.metricsClientURL=http://prometheus:9090 \
  -n kube-system
```

### Project Structure

```
cmd/                         # Main application code
  main.go                    # Entry point, CLI flags, ticker loop
  reconcile.go               # Core reconciliation logic
  kubeclient.go              # Kubernetes client initialization
  metrics.go                 # Metrics client factory
  utils.go                   # Helper functions

internal/
  metrics_clients/
    clients/                 # MetricsClient interface
    prometheus/              # Prometheus implementation + tests
  logger/                    # Logrus logger initialization

charts/pvc-autoscaler/       # Helm chart
  templates/                 # K8s manifests
  values.yaml               # Default configuration
  values-openshift.yaml     # OpenShift overrides
```

### Running Locally

```bash
# Build the binary
make build

# Run against your kubeconfig cluster
./bin/pvc-autoscaler \
  --metrics-client=prometheus \
  --metrics-client-url=http://localhost:9090 \
  --polling-interval=30s \
  --log-level=DEBUG
```

For more detailed development guidelines, see [CLAUDE.md](CLAUDE.md).

## Security Considerations

### RBAC Least Privilege

The controller requires cluster-wide permissions for PVCs and StorageClasses. Review the ClusterRole in `charts/pvc-autoscaler/templates/clusterrole.yaml` to ensure it meets your security requirements.

### Bearer Token Security

When using bearer token authentication (especially on OpenShift):
- Tokens are read from the filesystem (ServiceAccount token)
- Tokens are sent with every Prometheus API request
- Use TLS when possible (`insecureSkipVerify=false`)
- Rotate ServiceAccount tokens according to your security policy

### Network Policies

Consider creating NetworkPolicies to restrict:
- Ingress: Controller doesn't need to accept connections
- Egress: Allow only to Kubernetes API and Prometheus

Example egress policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pvc-autoscaler-egress
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      app: pvc-autoscaler
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: prom  # Your Prometheus namespace
    ports:
    - protocol: TCP
      port: 9090
  - to:  # Kubernetes API
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          component: apiserver
```

### Image Security

- Official images are published to `mkihr1/pvc-autoscaler` on Docker Hub
- Multi-arch builds (amd64, arm64)
- Review the [Dockerfile](Dockerfile) for security practices
- Consider using image vulnerability scanning in your CI/CD

### Audit Logging

Enable Kubernetes audit logging to track PVC modifications made by the controller.

## Upgrade Guide

### Upgrading the Helm Chart

```bash
# Update Helm repository
helm repo update

# Check for new versions
helm search repo pvc-autoscaler/pvcautoscaler --versions

# Upgrade to latest
helm upgrade pvc-autoscaler pvc-autoscaler/pvcautoscaler \
  -n kube-system \
  --reuse-values
```

### Version Compatibility

| Chart Version | App Version | Kubernetes Version | Notes |
|---------------|-------------|-------------------|-------|
| 1.0.3 | 1.0.1 | 1.24+ | Current version |

### Breaking Changes

**1.0.0 → 1.0.1+**
- No breaking changes

### Migration Notes

- The controller maintains state via PVC annotations (`previous_capacity`)
- No data migration required between versions
- Helm upgrade will restart the controller pod
- In-progress resizes will continue normally

## Contributions

Contributions to PVC Autoscaler are more than welcome! Whether you want to help me improve the code, add new features, fix bugs, or improve our documentation, I would be glad to receive your pull requests and issues.

### How to Contribute

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Run tests (`make test`)
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

### Reporting Issues

Please use GitHub Issues to report bugs or request features. Include:
- PVC Autoscaler version
- Kubernetes version and platform (EKS, AKS, GKE, OpenShift, etc.)
- Steps to reproduce
- Relevant logs (use `--log-level=DEBUG`)

## License

This project is licensed under the MIT License - see the LICENSE file for details.
