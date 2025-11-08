# Helm Chart - Monolithic Architecture - API Integration

This guide demonstrates how to deploy the **onix-adapter** using **Helm charts** in a **monolithic architecture** with **REST API** communication on **Kubernetes**.

## Architecture Overview

Deploy onix-adapter as a monolithic application on Kubernetes using Helm charts. In this architecture, the onix-adapter runs as a single pod/service that handles both incoming and outgoing API requests. All communication happens via HTTP/REST API endpoints.

### Components

- **Redis**: Used for caching and state management (deployed as a separate service)
- **Onix-Adapter**: Single pod handling all BAP/BPP operations
- **API Communication**: Direct HTTP/REST API calls between services
- **Kubernetes Services**: Service objects for service discovery and load balancing

## Prerequisites

- Kubernetes cluster (v1.20+)
- Helm 3.x installed
- kubectl configured to access your cluster
- Access to onix-adapter Docker images:
  - `manendrapalsingh/onix-bap-plugin:latest`
  - `manendrapalsingh/onix-bpp-plugin:latest`
- Persistent volume support (for Redis if using persistent storage)

## Directory Structure

```
helm/monolithic/api/
├── Chart.yaml                    # Helm chart metadata
├── values.yaml                   # Default configuration values
├── values-bap.yaml              # BAP-specific values
├── values-bpp.yaml              # BPP-specific values
├── templates/
│   ├── deployment.yaml          # Kubernetes deployment
│   ├── service.yaml             # Kubernetes service
│   ├── configmap.yaml           # Configuration files
│   ├── secret.yaml              # Secrets (if needed)
│   └── redis/                   # Redis deployment (optional)
│       ├── deployment.yaml
│       └── service.yaml
└── README.md                    # This file
```

## Quick Start

### Install BAP Adapter

```bash
# Install with default values
helm install onix-bap ./helm/monolithic/api -f ./helm/monolithic/api/values-bap.yaml

# Or install with custom values
helm install onix-bap ./helm/monolithic/api -f ./helm/monolithic/api/values-bap.yaml --set image.tag=v1.0.0
```

### Install BPP Adapter

```bash
# Install with default values
helm install onix-bpp ./helm/monolithic/api -f ./helm/monolithic/api/values-bpp.yaml

# Or install with custom values
helm install onix-bpp ./helm/monolithic/api -f ./helm/monolithic/api/values-bpp.yaml --set image.tag=v1.0.0
```

### Check Status

```bash
# Check Helm release status
helm status onix-bap
helm status onix-bpp

# Check pod status
kubectl get pods -l app=onix-bap
kubectl get pods -l app=onix-bpp

# Check services
kubectl get svc -l app=onix-bap
kubectl get svc -l app=onix-bpp
```

## Configuration

### Values File Structure

The Helm chart uses `values.yaml` files to configure the deployment. Key configuration areas include:

#### Application Configuration

```yaml
appName: onix-ev-charging
image:
  repository: manendrapalsingh/onix-bap-plugin
  tag: latest
  pullPolicy: IfNotPresent

replicaCount: 1

service:
  type: ClusterIP
  port: 8001
  targetPort: 8001
```

#### Redis Configuration

```yaml
redis:
  enabled: true
  image:
    repository: redis
    tag: 7-alpine
  service:
    port: 6379
  persistence:
    enabled: false
```

#### Adapter Configuration

```yaml
config:
  adapter:
    # Adapter configuration (mounted as ConfigMap)
    appName: onix-ev-charging
    http:
      port: 8001
  routing:
    # Routing configuration files
    caller: |
      # Routing rules YAML
    receiver: |
      # Routing rules YAML
```

### Environment Variables

```yaml
env:
  - name: CONFIG_FILE
    value: /app/config/adapter.yaml
  - name: LOG_LEVEL
    value: info
```

### Resource Limits

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

## Deployment Options

### Option 1: Deploy with Embedded Redis

Redis is deployed as part of the Helm chart:

```bash
helm install onix-bap ./helm/monolithic/api \
  -f ./helm/monolithic/api/values-bap.yaml \
  --set redis.enabled=true
```

### Option 2: Deploy with External Redis

Use an existing Redis instance:

```bash
helm install onix-bap ./helm/monolithic/api \
  -f ./helm/monolithic/api/values-bap.yaml \
  --set redis.enabled=false \
  --set config.redis.host=external-redis-service \
  --set config.redis.port=6379
```

### Option 3: Deploy with Custom Configuration

Override specific values:

```bash
helm install onix-bap ./helm/monolithic/api \
  -f ./helm/monolithic/api/values-bap.yaml \
  --set image.tag=v1.0.0 \
  --set replicaCount=3 \
  --set service.type=LoadBalancer
```

## Service Endpoints

Once deployed, the services are accessible via Kubernetes services:

### BAP Service

- **Service Name**: `onix-bap-service`
- **Port**: 8001
- **Caller Endpoint**: `http://onix-bap-service:8001/bap/caller/{action}`
- **Receiver Endpoint**: `http://onix-bap-service:8001/bap/receiver/{action}`

### BPP Service

- **Service Name**: `onix-bpp-service`
- **Port**: 8002
- **Caller Endpoint**: `http://onix-bpp-service:8002/bpp/caller/{action}`
- **Receiver Endpoint**: `http://onix-bpp-service:8002/bpp/receiver/{action}`

### Access from Outside Cluster

If using `LoadBalancer` or `NodePort` service type:

```bash
# Get external IP (LoadBalancer)
kubectl get svc onix-bap-service

# Get NodePort (NodePort)
kubectl get svc onix-bap-service -o jsonpath='{.spec.ports[0].nodePort}'
```

## Upgrading

### Upgrade Release

```bash
# Upgrade with new values
helm upgrade onix-bap ./helm/monolithic/api \
  -f ./helm/monolithic/api/values-bap.yaml \
  --set image.tag=v1.1.0

# Check upgrade status
helm status onix-bap
```

### Rollback

```bash
# List release history
helm history onix-bap

# Rollback to previous version
helm rollback onix-bap 1
```

## Uninstalling

```bash
# Uninstall BAP
helm uninstall onix-bap

# Uninstall BPP
helm uninstall onix-bpp

# Remove associated resources (if needed)
kubectl delete pvc -l app=onix-bap
```

## Troubleshooting

### Pod Won't Start

1. **Check pod status:**
   ```bash
   kubectl get pods -l app=onix-bap
   kubectl describe pod <pod-name>
   ```

2. **Check pod logs:**
   ```bash
   kubectl logs <pod-name>
   kubectl logs <pod-name> --previous  # Previous container logs
   ```

3. **Check events:**
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   ```

### Configuration Issues

1. **Verify ConfigMap:**
   ```bash
   kubectl get configmap onix-bap-config -o yaml
   ```

2. **Check mounted files:**
   ```bash
   kubectl exec <pod-name> -- ls -la /app/config/
   kubectl exec <pod-name> -- cat /app/config/adapter.yaml
   ```

### Redis Connection Issues

1. **Verify Redis service:**
   ```bash
   kubectl get svc redis-onix-bap
   kubectl get pods -l app=redis-onix-bap
   ```

2. **Test connectivity:**
   ```bash
   kubectl exec <pod-name> -- redis-cli -h redis-onix-bap ping
   ```

### Service Discovery Issues

1. **Verify service endpoints:**
   ```bash
   kubectl get endpoints onix-bap-service
   ```

2. **Test service connectivity:**
   ```bash
   kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://onix-bap-service:8001/health
   ```

## Customization

### Adding Custom Environment Variables

Edit `values.yaml`:

```yaml
env:
  - name: CUSTOM_VAR
    value: custom-value
  - name: SECRET_VAR
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: secret-key
```

### Configuring Persistent Volumes

For Redis persistence:

```yaml
redis:
  persistence:
    enabled: true
    size: 10Gi
    storageClass: standard
```

### Setting Resource Limits

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

## Next Steps

- For RabbitMQ integration: See [Helm Monolithic RabbitMQ](./../rabbitmq/README.md)
- For Kafka integration: See [Helm Monolithic Kafka](./../kafka/README.md)
- For microservice architecture: See [Helm Microservice API](./../../microservice/api/README.md)

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
