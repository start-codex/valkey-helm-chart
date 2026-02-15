# Valkey Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/valkey-redis)](https://artifacthub.io/packages/helm/valkey-redis/valkey)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Helm](https://img.shields.io/badge/Helm-3.x-blue)](https://helm.sh)
[![Auto-Update](https://img.shields.io/badge/Auto--Update-Weekly-green)](https://github.com/start-codex/valkey-helm-chart/actions/workflows/update-valkey-version.yml)
[![Chainguard](https://img.shields.io/badge/Images-Chainguard%20%7C%20Zero%20CVE-brightgreen)](https://www.chainguard.dev/)

<p align="center">
  <img src="https://valkey.io/img/valkey-logo-og.png" alt="Valkey Logo" width="300">
</p>

Helm chart for deploying [Valkey](https://valkey.io/) on Kubernetes. Valkey is a high-performance, open-source data structure server compatible with Redis.

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Image Versioning Strategy](#image-versioning-strategy)
- [Quick Start](#quick-start)
- [Architectures](#architectures)
- [Configuration](#configuration)
- [Examples](#examples)
- [Connecting to Valkey](#connecting-to-valkey)
- [Upgrades](#upgrades)
- [Monitoring](#monitoring)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Features

| Feature | Description |
|---------|-------------|
| **Standalone Mode** | Simple single-instance deployment |
| **Sentinel Mode** | High availability with automatic failover |
| **Authentication** | Support for password and existing secrets |
| **Persistence** | Configurable persistent volumes |
| **Metrics** | Built-in Prometheus exporter |
| **Security** | SecurityContext, NetworkPolicies, RBAC |
| **Automatic Upgrades** | Pre-upgrade hooks for zero-downtime migrations |
| **TLS** | Support for encrypted connections |

## Requirements

| Component | Version |
|-----------|---------|
| Kubernetes | >= 1.23 |
| Helm | >= 3.8 |

## Image Versioning Strategy

This chart uses **Chainguard's zero-CVE Valkey images** for enhanced security.

### Why `latest` tag?

- **Free tier limitation**: Chainguard's free tier only provides the `latest` tag
- **Automatic updates**: Using `latest` ensures you always get the most recent security patches
- **Zero CVEs**: Chainguard images are rebuilt continuously to maintain zero known vulnerabilities

### Version tracking

- **appVersion in Chart.yaml**: Reflects the current Valkey version available in `cgr.dev/chainguard/valkey:latest`
- **Automated updates**: A GitHub Action checks weekly for version updates and creates PRs automatically
- **Transparency**: Every version change is tracked via pull requests and changelog entries

### For production use

If you require **version pinning** for production:

```yaml
# Override with a specific version (requires Chainguard Pro or alternative registry)
image:
  repository: valkey/valkey  # Official Valkey images
  tag: "9.0.0"               # Specific version tag
```

> **Note**: Using `latest` provides continuous security updates but means deployments may pull different versions over time. For strict reproducibility, consider using image digests or switching to a registry that provides versioned tags.

## Quick Start

```bash
# Add the repository
helm repo add valkey https://start-codex.github.io/valkey-helm-chart
helm repo update

# Install with default values
helm install my-valkey valkey/valkey

# Install with authentication
helm install my-valkey valkey/valkey \
  --set auth.enabled=true \
  --set auth.password="your-secure-password"
```

## Architectures

### Standalone (Default)

Simple deployment with a single Valkey instance. Ideal for development and workloads that don't require high availability.

```
+------------------+
|     Valkey       |
|   (standalone)   |
+------------------+
        |
+------------------+
|       PVC        |
+------------------+
```

```bash
helm install my-valkey valkey/valkey
```

### Sentinel (High Availability)

Architecture with master, replicas, and sentinels for automatic failover. Recommended for production.

```
+------------------+     +------------------+     +------------------+
|     Sentinel     |     |     Sentinel     |     |     Sentinel     |
+------------------+     +------------------+     +------------------+
         |                       |                        |
         +-------------+---------+------------------------+
                       |
         +-------------+-------------+
         |             |             |
+--------v---+  +------v-----+  +----v-------+
|   Master   |  |  Replica   |  |  Replica   |
+------------+  +------------+  +------------+
      |              |               |
+-----v----+  +------v-----+  +-----v------+
|   PVC    |  |    PVC     |  |    PVC     |
+----------+  +------------+  +------------+
```

```bash
helm install my-valkey valkey/valkey \
  --set architecture=sentinel \
  --set sentinel.replicaCount=3 \
  --set replica.replicaCount=2
```

## Configuration

### Global Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `architecture` | Deployment mode: `standalone` or `sentinel` | `standalone` |
| `global.imageRegistry` | Global registry for all images | `""` |
| `global.storageClass` | Global StorageClass | `""` |
| `clusterDomain` | Kubernetes cluster domain | `cluster.local` |

### Image

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.registry` | Image registry | `docker.io` |
| `image.repository` | Image repository | `valkey/valkey` |
| `image.tag` | Image tag | `9.0.0` |
| `image.pullPolicy` | Pull policy | `IfNotPresent` |

### Authentication

| Parameter | Description | Default |
|-----------|-------------|---------|
| `auth.enabled` | Enable authentication | `false` |
| `auth.password` | Password (not recommended for production) | `""` |
| `auth.existingSecret` | Name of existing Secret | `""` |
| `auth.existingSecretPasswordKey` | Password key in Secret | `password` |

### Standalone

| Parameter | Description | Default |
|-----------|-------------|---------|
| `standalone.replicaCount` | Number of replicas | `1` |
| `standalone.persistence.enabled` | Enable persistence | `true` |
| `standalone.persistence.size` | Volume size | `8Gi` |
| `standalone.persistence.storageClass` | StorageClass | `""` |
| `standalone.service.type` | Service type | `ClusterIP` |
| `standalone.service.port` | Service port | `6379` |
| `standalone.resources.requests.memory` | Memory request | `128Mi` |
| `standalone.resources.requests.cpu` | CPU request | `100m` |
| `standalone.resources.limits.memory` | Memory limit | `256Mi` |

### Sentinel

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sentinel.replicaCount` | Number of sentinels | `3` |
| `sentinel.quorum` | Quorum for failover | `2` |
| `sentinel.downAfterMilliseconds` | Time to detect failure | `30000` |
| `sentinel.failoverTimeout` | Failover timeout | `180000` |
| `master.replicaCount` | Number of masters | `1` |
| `replica.replicaCount` | Number of replicas | `2` |

### Metrics

| Parameter | Description | Default |
|-----------|-------------|---------|
| `metrics.enabled` | Enable Prometheus exporter | `false` |
| `metrics.image.repository` | Exporter image | `oliver006/redis_exporter` |
| `metrics.image.tag` | Exporter tag | `v1.81.0` |
| `metrics.serviceMonitor.enabled` | Create ServiceMonitor | `false` |
| `metrics.podMonitor.enabled` | Create PodMonitor | `false` |

### Security

| Parameter | Description | Default |
|-----------|-------------|---------|
| `podSecurityContext.fsGroup` | Filesystem group | `999` |
| `podSecurityContext.runAsUser` | Container user | `999` |
| `securityContext.runAsNonRoot` | Run as non-root | `true` |
| `securityContext.readOnlyRootFilesystem` | Read-only filesystem | `true` |
| `networkPolicy.enabled` | Enable NetworkPolicy | `false` |

## Examples

### Local Development

```yaml
# values-dev.yaml
architecture: standalone
auth:
  enabled: false
standalone:
  persistence:
    enabled: false
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
    limits:
      memory: 128Mi
```

```bash
helm install valkey-dev valkey/valkey -f values-dev.yaml
```

### Production with Authentication

```yaml
# values-prod.yaml
architecture: standalone
auth:
  enabled: true
  existingSecret: valkey-secret
  existingSecretPasswordKey: password
standalone:
  persistence:
    enabled: true
    storageClass: fast-ssd
    size: 50Gi
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

```bash
# Create the secret first
kubectl create secret generic valkey-secret \
  --from-literal=password="your-super-secure-password"

# Install
helm install valkey-prod valkey/valkey -f values-prod.yaml
```

### High Availability with Sentinel

```yaml
# values-ha.yaml
architecture: sentinel
auth:
  enabled: true
  password: "ha-password"

sentinel:
  replicaCount: 3
  quorum: 2
  resources:
    requests:
      memory: 128Mi
      cpu: 100m

master:
  persistence:
    enabled: true
    size: 20Gi
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi

replica:
  replicaCount: 2
  persistence:
    enabled: true
    size: 20Gi
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

```bash
helm install valkey-ha valkey/valkey -f values-ha.yaml
```

### With Network Policies

```yaml
# values-secure.yaml
architecture: standalone
auth:
  enabled: true
  existingSecret: valkey-secret

networkPolicy:
  enabled: true
  allowExternal: false
  ingressNSMatchLabels:
    app: my-app
```

## Connecting to Valkey

### From Inside the Cluster

```bash
# Temporary pod for testing
kubectl run valkey-client --rm -it \
  --image=valkey/valkey:9.0.0 \
  -- valkey-cli -h my-valkey

# With authentication
kubectl run valkey-client --rm -it \
  --image=valkey/valkey:9.0.0 \
  -- valkey-cli -h my-valkey -a "your-password"
```

### Port-forward for Local Access

```bash
kubectl port-forward svc/my-valkey 6379:6379

# In another terminal
valkey-cli -h localhost -p 6379
```

### Sentinel - Get Current Master

```bash
kubectl run valkey-client --rm -it \
  --image=valkey/valkey:9.0.0 \
  -- valkey-cli -h my-valkey-sentinel -p 26379

# Useful sentinel commands
> SENTINEL masters
> SENTINEL get-master-addr-by-name mymaster
> SENTINEL replicas mymaster
```

## Upgrades

This chart includes an automatic upgrade mechanism that handles StatefulSets transparently, avoiding immutable field errors.

### How It Works

1. **Pre-upgrade Hook**: Deletes StatefulSets with `--cascade=orphan`, preserving pods and PVCs
2. **Recreation**: Helm recreates StatefulSets with new configuration
3. **Rolling Update**: Pods are updated gradually

### Upgrading the Chart

```bash
# Update repository
helm repo update

# View available versions
helm search repo valkey/valkey --versions

# Upgrade to latest version
helm upgrade my-valkey valkey/valkey

# Upgrade with new values
helm upgrade my-valkey valkey/valkey -f new-values.yaml

# Upgrade Valkey image
helm upgrade my-valkey valkey/valkey --set image.tag=9.0.0
```

### Hook Configuration

```yaml
preUpgradeHook:
  image:
    registry: docker.io
    repository: alpine/k8s
    tag: "1.31.13"
  resources:
    limits:
      memory: 128Mi
    requests:
      cpu: 50m
      memory: 64Mi
```

## Monitoring

### Enable Prometheus Metrics

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s
    scrapeTimeout: 10s
```

### Available Metrics

| Metric | Description |
|--------|-------------|
| `redis_up` | Server status |
| `redis_connected_clients` | Connected clients |
| `redis_memory_used_bytes` | Memory used |
| `redis_commands_processed_total` | Processed commands |
| `redis_keyspace_hits_total` | Cache hits |
| `redis_keyspace_misses_total` | Cache misses |

### Grafana Dashboard

You can use the official Redis Exporter dashboard: [Grafana Dashboard 763](https://grafana.com/grafana/dashboards/763)

## Security

### Production Recommendations

1. **Use external Secrets** for passwords:
   ```yaml
   auth:
     enabled: true
     existingSecret: my-valkey-secret
   ```

2. **Enable Network Policies**:
   ```yaml
   networkPolicy:
     enabled: true
     allowExternal: false
   ```

3. **Configure resources**:
   ```yaml
   standalone:
     resources:
       limits:
         memory: 2Gi
       requests:
         memory: 1Gi
   ```

4. **Enable TLS** (if needed):
   ```yaml
   tls:
     enabled: true
     existingSecret: valkey-tls-secret
   ```

### Default Security Context

```yaml
podSecurityContext:
  fsGroup: 999
  runAsUser: 999
  runAsGroup: 999

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

## Troubleshooting

### Pod Won't Start

```bash
# View events
kubectl describe pod my-valkey-0

# View logs
kubectl logs my-valkey-0

# Check PVC
kubectl get pvc
```

### Connection Error

```bash
# Check service
kubectl get svc my-valkey

# Check endpoints
kubectl get endpoints my-valkey

# Connectivity test
kubectl run test --rm -it --image=busybox -- nc -zv my-valkey 6379
```

### StatefulSet Upgrade Error

If the pre-upgrade hook fails, you can delete manually:

```bash
kubectl delete statefulset my-valkey --cascade=orphan
helm upgrade my-valkey valkey/valkey
```

### Check Sentinel Status

```bash
kubectl exec -it my-valkey-sentinel-0 -- valkey-cli -p 26379 SENTINEL masters
```

## Uninstallation

```bash
# Uninstall release
helm uninstall my-valkey

# Delete PVCs (WARNING: this deletes data)
kubectl delete pvc -l app.kubernetes.io/instance=my-valkey
```

## Development

### Test Locally

```bash
# Validate syntax
helm lint .

# Render templates
helm template test . --debug

# Dry-run
helm install test . --dry-run --debug

# Install in test namespace
helm install test . -n valkey-test --create-namespace
```

### Run Tests

```bash
helm test my-valkey
```

## Contributing

1. Fork the repository
2. Create a branch (`git checkout -b feature/new-feature`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to branch (`git push origin feature/new-feature`)
5. Create a Pull Request

## Links

- [Valkey Official](https://valkey.io/)
- [Valkey GitHub](https://github.com/valkey-io/valkey)
- [Artifact Hub](https://artifacthub.io/packages/helm/valkey/valkey)
- [Chart Repository](https://github.com/start-codex/valkey-helm-chart)

## License

This project is licensed under [Apache 2.0](LICENSE).

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/start-codex">StartCodex</a>
</p>
