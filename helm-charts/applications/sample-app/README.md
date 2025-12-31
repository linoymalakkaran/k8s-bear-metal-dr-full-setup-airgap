# Sample Application Helm Chart

This directory contains a template Helm chart for deploying applications on the Kubernetes cluster.

## Chart Structure

```
sample-app/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── values-production.yaml  # Production overrides
├── values-staging.yaml     # Staging overrides
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml            # Horizontal Pod Autoscaler
│   ├── pvc.yaml            # Persistent Volume Claim
│   ├── serviceaccount.yaml
│   └── _helpers.tpl        # Template helpers
└── README.md               # This file
```

## Usage

### Install Chart

```bash
# Install with default values
helm install my-app ./sample-app

# Install with production values
helm install my-app ./sample-app -f values-production.yaml -n production

# Install with custom values
helm install my-app ./sample-app --set image.tag=v2.0.0

# Dry run to see generated manifests
helm install my-app ./sample-app --dry-run --debug
```

### Upgrade Chart

```bash
# Upgrade existing release
helm upgrade my-app ./sample-app -f values-production.yaml

# Upgrade with rollback on failure
helm upgrade my-app ./sample-app --atomic --timeout 5m
```

### Uninstall Chart

```bash
helm uninstall my-app
```

## Configuration

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `3` |
| `image.repository` | Container image repository | `nginx` |
| `image.tag` | Container image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.hosts` | Ingress hostnames | `[]` |
| `resources.limits.cpu` | CPU limit | `500m` |
| `resources.limits.memory` | Memory limit | `512Mi` |
| `autoscaling.enabled` | Enable HPA | `true` |
| `autoscaling.minReplicas` | Minimum replicas | `2` |
| `autoscaling.maxReplicas` | Maximum replicas | `10` |

### Example Values Override

```yaml
# custom-values.yaml
replicaCount: 5

image:
  repository: myapp/frontend
  tag: v1.2.3

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
```

## Development

### Create New Chart

```bash
# Create from this template
cp -r sample-app my-new-app
cd my-new-app

# Update Chart.yaml with your app details
# Update values.yaml with your app configuration
# Customize templates as needed
```

### Lint Chart

```bash
helm lint ./sample-app
```

### Template Rendering

```bash
# Render templates locally
helm template my-app ./sample-app

# Render with specific values
helm template my-app ./sample-app -f values-production.yaml
```

## Best Practices

1. **Version Control**: Always specify exact image tags (avoid `latest`)
2. **Resource Limits**: Set appropriate CPU and memory limits
3. **Health Checks**: Configure liveness and readiness probes
4. **Secrets**: Use Kubernetes secrets or external secret managers
5. **RBAC**: Create service accounts with minimal permissions
6. **Labels**: Use consistent labeling for resources
7. **Documentation**: Keep values.yaml well-documented

## Troubleshooting

```bash
# Check release status
helm status my-app

# View release history
helm history my-app

# Rollback to previous version
helm rollback my-app 1

# Get rendered values
helm get values my-app

# Get all release information
helm get all my-app
```

## Additional Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Development Tips](https://helm.sh/docs/howto/charts_tips_and_tricks/)
