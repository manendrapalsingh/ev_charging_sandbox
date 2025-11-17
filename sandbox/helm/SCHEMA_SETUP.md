# Schema Configuration for All Sandbox Environments

All ONIX adapter deployments require JSON schemas for request validation. Each sandbox environment has its own `populate-schemas.sh` script to populate the schemas ConfigMap.

## Quick Reference

| Sandbox Type | Script Location | Default Release Names | ConfigMap Pattern |
|-------------|----------------|----------------------|-------------------|
| **Monolithic** | `sandbox/helm/api/monolithic/populate-schemas.sh` | `ev-charging-bap`, `ev-charging-bpp` | `{release}-onix-api-monolithic-schemas` |
| **Microservice** | `sandbox/helm/api/microservice/populate-schemas.sh` | `ev-charging-microservice-bap`, `ev-charging-microservice-bpp` | `{release}-onix-api-microservice-schemas` |
| **Kafka** | `sandbox/helm/kafka/populate-schemas.sh` | `ev-charging-kafka-bap`, `ev-charging-kafka-bpp` | `{release}-onix-kafka-schemas` |
| **RabbitMQ** | `sandbox/helm/rabbitmq/populate-schemas.sh` | `ev-charging-rabbitmq-bap`, `ev-charging-rabbitmq-bpp` | `{release}-onix-rabbitmq-schemas` |

## Usage

### Basic Usage (Default Release Names)

```bash
# Navigate to the appropriate sandbox directory
cd sandbox/helm/api/monolithic  # or microservice/kafka/rabbitmq

# Run the populate script
./populate-schemas.sh
```

### Custom Release Names

If you used different release names during Helm installation:

```bash
# Set custom release names
RELEASE_BAP=my-custom-bap-name \
RELEASE_BPP=my-custom-bpp-name \
./populate-schemas.sh
```

### Custom Namespace

```bash
# Use a different namespace
NAMESPACE=my-namespace ./populate-schemas.sh
```

### Custom Schema Directory

```bash
# Use a different schema directory
SCHEMAS_DIR=/path/to/schemas/beckn.one_deg_ev-charging/v2.0.0 \
./populate-schemas.sh
```

## When to Run

1. **After initial Helm installation** - Before pods start, populate schemas
2. **After schema updates** - When schema files change in the repository
3. **After upgrading Helm releases** - Ensure ConfigMaps are up-to-date
4. **When deploying to new namespaces** - Each namespace needs its own ConfigMaps

## What the Script Does

1. Validates that the schema directory exists
2. Creates or updates the BAP schemas ConfigMap
3. Creates or updates the BPP schemas ConfigMap
4. Provides commands to restart pods (to pick up new schemas)

## Restarting Pods

After populating schemas, restart the pods to pick up the changes:

```bash
# For monolithic
kubectl delete pod -n ev-charging-sandbox -l component=bap,app.kubernetes.io/instance=ev-charging-bap
kubectl delete pod -n ev-charging-sandbox -l component=bpp,app.kubernetes.io/instance=ev-charging-bpp

# Or use the commands printed by the script
```

## Troubleshooting

### Schema Validation Errors

If you see errors like `schema validation failed: schema not found for domain: beckn.one_deg_ev-charging`:

1. **Verify ConfigMap exists:**
   ```bash
   kubectl get configmap -n ev-charging-sandbox | grep schemas
   ```

2. **Check ConfigMap contents:**
   ```bash
   kubectl get configmap <release-name>-onix-<chart-type>-schemas \
     -n ev-charging-sandbox -o jsonpath='{.data}' | jq 'keys'
   ```

3. **Repopulate schemas:**
   ```bash
   ./populate-schemas.sh
   kubectl delete pod -n ev-charging-sandbox -l component=bap
   kubectl delete pod -n ev-charging-sandbox -l component=bpp
   ```

### Wrong Release Name

If you get errors about ConfigMaps not found, check your Helm release names:

```bash
# List all Helm releases
helm list -n ev-charging-sandbox

# Use the correct release names
RELEASE_BAP=<actual-release-name> \
RELEASE_BPP=<actual-release-name> \
./populate-schemas.sh
```

## Notes

- All sandbox environments use the same schema files from `schemas/beckn.one_deg_ev-charging/v2.0.0/`
- Each Helm release gets its own ConfigMap (even if they use the same schemas)
- ConfigMaps are namespace-scoped, so you need to populate them for each namespace
- The scripts are idempotent - safe to run multiple times

