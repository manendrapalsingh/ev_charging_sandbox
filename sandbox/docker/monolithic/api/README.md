# EV Charging Sandbox - Unified Docker Compose Setup

This directory contains a unified Docker Compose configuration that sets up a complete EV Charging sandbox environment with all necessary services: API adapters (BAP and BPP), mock services (CDS, Registry, BAP, BPP), and supporting infrastructure.

## Architecture Overview

This setup creates a fully functional sandbox environment for testing and developing EV Charging network integrations using the ONIX protocol. The architecture includes:

- **ONIX Adapters**: Protocol adapters for BAP (Buyer App Provider) and BPP (Buyer Platform Provider)
- **Mock Services**: Simulated services for testing without real implementations
- **Supporting Services**: Redis for caching and state management

## Services

### Core Services

1. **redis-onix-bap** (Port: 6379)
   - Redis cache for the BAP adapter
   - Used for storing transaction state, caching registry lookups, and session management

2. **redis-onix-bpp** (Port: 6380)
   - Redis cache for the BPP adapter
   - Used for storing transaction state, caching registry lookups, and session management

3. **onix-bap-plugin** (Port: 8001)
   - ONIX protocol adapter for BAP (Buyer App Provider)
   - Handles protocol compliance, signing, validation, and routing for BAP transactions
   - **Caller Endpoint**: `/bap/caller/` - Entry point for requests from BAP application
   - **Receiver Endpoint**: `/bap/receiver/` - Receives callbacks from CDS and BPPs

4. **onix-bpp-plugin** (Port: 8002)
   - ONIX protocol adapter for BPP (Buyer Platform Provider)
   - Handles protocol compliance, signing, validation, and routing for BPP transactions
   - **Caller Endpoint**: `/bpp/caller/` - Sends responses to CDS and BAPs
   - **Receiver Endpoint**: `/bpp/receiver/` - Receives requests from CDS and BAPs

### Mock Services

5. **mock-registry** (Port: 3030)
   - Mock implementation of the network registry service
   - Maintains a registry of all BAPs, BPPs, and CDS services on the network
   - Provides subscriber lookup and key management functionality

6. **mock-cds** (Port: 8082)
   - Mock Catalog Discovery Service (CDS)
   - Aggregates discover requests from BAPs and broadcasts to registered BPPs
   - Collects and aggregates responses from multiple BPPs
   - Handles signature verification and signing

7. **mock-bap** (Port: 9001)
   - Mock BAP backend service
   - Simulates a Buyer App Provider application
   - Receives callbacks from the ONIX adapter

8. **mock-bpp** (Port: 9002)
   - Mock BPP backend service
   - Simulates a Buyer Platform Provider application
   - Handles requests from the ONIX adapter

## Configuration Files

Each service has its own configuration file with a service name prefix. This section explains what each config file is used for:

### 1. `onix-bap_config.yml`

**Purpose**: Complete adapter configuration for the ONIX BAP plugin.

**Usage**: This is a reference copy of the adapter configuration. The actual configuration is mounted from `../../../../docker/monolithic/api/config/onix-bap/adapter.yaml`.

**Key Sections**:
- **`appName`**: Application identifier ("onix-ev-charging")
- **`log`**: Logging configuration (level, destinations, context keys)
- **`http`**: HTTP server settings (port: 8001, timeouts)
- **`modules`**: Two main modules:
  - **`bapTxnReceiver`**: Handles incoming callbacks from CDS/BPPs
    - Validates signatures
    - Routes to backend (mock-bap)
    - Validates schemas
  - **`bapTxnCaller`**: Handles outgoing requests from BAP application
    - Routes requests (discover → CDS, others → BPP)
    - Signs requests
- **`plugins`**: Plugin configuration including:
  - Registry connection to mock-registry
  - Key manager for signing/encryption
  - Redis cache configuration
  - Schema validator
  - Router for request routing

**Note**: This file is informational. The actual config is in `docker/monolithic/api/config/onix-bap/`.

### 2. `onix-bpp_config.yml`

**Purpose**: Complete adapter configuration for the ONIX BPP plugin.

**Usage**: This is a reference copy of the adapter configuration. The actual configuration is mounted from `../../../../docker/monolithic/api/config/onix-bpp/adapter.yaml`.

**Key Sections**:
- **`appName`**: Application identifier ("bpp-ev-charging")
- **`log`**: Logging configuration
- **`http`**: HTTP server settings (port: 8002, timeouts)
- **`modules`**: Two main modules:
  - **`bppTxnReceiver`**: Handles incoming requests from CDS/BAPs
    - Validates signatures
    - Routes to backend (mock-bpp)
    - Validates schemas
  - **`bppTxnCaller`**: Handles outgoing responses to CDS/BAPs
    - Routes responses (on_discover → CDS, others → BAP)
    - Signs responses
- **`plugins`**: Similar plugin configuration as BAP, but for BPP role

**Note**: This file is informational. The actual config is in `docker/monolithic/api/config/onix-bpp/`.

### 3. `mock-registry_config.yml`

**Purpose**: Configuration for the mock registry service that maintains the network participant registry.

**Usage**: Mounted into the mock-registry container at `/app/config/config.yaml`.

**Key Sections**:
- **`server.port`**: Server port (3030)
- **`subscribers`**: List of all network participants:
  - **BAPs**: mock-bap, onix-bap, example-bap.com
  - **BPPs**: mock-bpp, onix-bpp, example-bpp.com, chargezone-energy-bpp.com
  - **CDS**: mock-cds
  - Each subscriber has:
    - `subscriber_id`: Unique identifier
    - `subscriber_uri`: Endpoint URL
    - `type`: BAP, BPP, or CDS
    - `signing_public_key` / `encr_public_key`: For signature verification (external subscribers)
    - `key_id`: Key identifier
- **`defaults`**: Validity period for registry entries

**When to Modify**:
- Add new BAPs, BPPs, or CDS services to the network
- Update subscriber URIs if services move
- Update keys when rotating encryption/signing keys
- Modify validity periods

### 4. `mock-cds_config.yml`

**Purpose**: Configuration for the mock Catalog Discovery Service that aggregates discover requests.

**Usage**: Mounted into the mock-cds container at `/app/config/config.yaml`.

**Key Sections**:
- **`server.port`**: Server port (8082)
- **`registry`**: Registry service connection:
  - `host`: mock-registry
  - `port`: 3030
  - `lookup_path`: /lookup
- **`cds.signing`**: CDS signing keys:
  - `private_key`: For signing aggregated responses
  - `public_key`: For signature verification
  - `subscriber_id`: "mock-cds"
  - `key_id`: "cds-key-1"
  - `verify_signatures`: Enable signature verification
- **`endpoints.bpps`**: List of BPP endpoints to broadcast discover requests to:
  - `host`: BPP service hostname
  - `port`: BPP service port
  - `discover_path`: Path for discover endpoint
  - `subscriber_id`: BPP subscriber ID
- **`timing`**: Timing configuration:
  - `broadcast_discover_delay_ms`: Delay before broadcasting (500ms)
  - `aggregate_response_delay_ms`: Delay before aggregating responses (1000ms)
  - `signature_expiry_minutes`: Signature expiry time (5 minutes)
- **`defaults`**: Default values for requests:
  - `version`: Protocol version ("1.0.0")
  - `country_code`: "IND"
  - `city_code`: "std:080"
  - `default_gps`: Default GPS coordinates

**When to Modify**:
- Add or remove BPP endpoints to broadcast to
- Update signing keys
- Adjust timing delays for testing
- Change default location/version values

### 5. `mock-bap_config.yml`

**Purpose**: Configuration for the mock BAP backend service.

**Usage**: Mounted into the mock-bap container at `/app/config/config.yaml`.

**Key Sections**:
- **`server.port`**: Server port (9001)
- **`defaults`**: Default values:
  - `version`: Protocol version ("1.0.0")
  - `country_code`: "IND"
  - `city_code`: "std:080"

**When to Modify**:
- Change server port
- Update default version or location codes

### 6. `mock-bpp_config.yml`

**Purpose**: Configuration for the mock BPP backend service.

**Usage**: Mounted into the mock-bpp container at `/app/config/config.yaml`.

**Key Sections**:
- **`server.port`**: Server port (9002)
- **`defaults`**: Default values:
  - `version`: Protocol version ("1.0.0")
  - `country_code`: "IND"
  - `city_code`: "std:080"

**When to Modify**:
- Change server port
- Update default version or location codes

## Quick Start

### Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to Docker images (pulled automatically)

### Starting All Services

```bash
# Navigate to this directory
cd sandbox/docker/monolithic/api

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Check service status
docker-compose ps
```

### Stopping Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

## Service Endpoints

Once all services are running, you can access:

| Service | Endpoint | Description |
|---------|----------|-------------|
| **ONIX BAP** | `http://localhost:8001` | BAP adapter endpoints |
| | `http://localhost:8001/bap/caller/{action}` | Send requests from BAP |
| | `http://localhost:8001/bap/receiver/{action}` | Receive callbacks |
| **ONIX BPP** | `http://localhost:8002` | BPP adapter endpoints |
| | `http://localhost:8002/bpp/caller/{action}` | Send responses |
| | `http://localhost:8002/bpp/receiver/{action}` | Receive requests |
| **Mock Registry** | `http://localhost:3030` | Registry service |
| | `http://localhost:3030/lookup` | Lookup subscribers |
| **Mock CDS** | `http://localhost:8082` | Catalog Discovery Service |
| | `http://localhost:8082/csd` | CDS aggregation endpoint |
| **Mock BAP** | `http://localhost:9001` | Mock BAP backend |
| **Mock BPP** | `http://localhost:9002` | Mock BPP backend |

## Configuration Workflow

1. **Service Discovery Flow**:
   - BAP sends discover request → ONIX BAP adapter
   - ONIX BAP routes to → Mock CDS
   - Mock CDS broadcasts to → All registered BPPs
   - BPPs respond → Mock CDS aggregates
   - Mock CDS sends aggregated response → ONIX BAP → Mock BAP

2. **Transaction Flow** (Phase 2+):
   - BAP sends select/init/confirm → ONIX BAP adapter
   - ONIX BAP routes directly to → ONIX BPP (bypasses CDS)
   - ONIX BPP forwards to → Mock BPP backend
   - Mock BPP responds → ONIX BPP
   - ONIX BPP routes callback → ONIX BAP → Mock BAP

## Network Architecture

All services run on a shared Docker network (`onix-network`) allowing:
- Service-to-service communication using container names as hostnames
- Isolated networking from other Docker containers
- Easy service discovery without IP address management

## Customization

### Adding New BPP Endpoints

1. Edit `mock-cds_config.yml`:
   ```yaml
   endpoints:
     bpps:
       - host: "new-bpp-service"
         port: "8002"
         discover_path: "/bpp/receiver/discover"
         subscriber_id: "new-bpp-id"
   ```

2. Add the BPP to `mock-registry_config.yml`:
   ```yaml
   subscribers:
     - subscriber_id: "new-bpp-id"
       subscriber_uri: "http://new-bpp-service:8002"
       type: "BPP"
   ```

3. Restart services:
   ```bash
   docker-compose restart mock-cds mock-registry
   ```

### Changing Service Ports

Edit `docker-compose.yml` and update the port mappings:
```yaml
ports:
  - "NEW_PORT:CONTAINER_PORT"
```

**Important**: Also update the corresponding config files if services reference each other's ports.

### Updating Signing Keys

1. Generate new key pairs for the service
2. Update the service's config file (e.g., `mock-cds_config.yml`)
3. Update `mock-registry_config.yml` with the new public key
4. Restart affected services

## Troubleshooting

### Service Won't Start

1. **Check ports are available**:
   ```bash
   lsof -i :8001  # BAP
   lsof -i :8002  # BPP
   lsof -i :3030  # Registry
   lsof -i :8082  # CDS
   ```

2. **Check container logs**:
   ```bash
   docker-compose logs <service-name>
   ```

3. **Verify health checks**:
   ```bash
   docker-compose ps
   ```

### Configuration Issues

1. **Verify config files are mounted**:
   ```bash
   docker exec <container-name> ls -la /app/config/
   ```

2. **Check config file syntax**:
   ```bash
   # YAML validation
   docker exec <container-name> cat /app/config/config.yaml
   ```

### Registry Lookup Failures

1. **Verify registry is running**:
   ```bash
   curl http://localhost:3030/health
   ```

2. **Check subscriber registration**:
   ```bash
   curl http://localhost:3030/lookup?subscriber_id=mock-bap
   ```

3. **Verify subscriber URIs are reachable from registry**:
   - Check network connectivity
   - Verify service names match container names

## File Structure

```
sandbox/docker/monolithic/api/
├── docker-compose.yml              # Unified compose file for all services
├── README.md                        # This file
├── onix-bap_config.yml              # Reference config for ONIX BAP (informational)
├── onix-bpp_config.yml              # Reference config for ONIX BPP (informational)
├── mock-registry_config.yml         # Mock registry configuration
├── mock-cds_config.yml              # Mock CDS configuration
├── mock-bap_config.yml              # Mock BAP configuration
└── mock-bpp_config.yml              # Mock BPP configuration
```

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)
- [Main API README](../../../../../docker/monolithic/api/README.md) - Detailed ONIX adapter documentation

## Notes

- The `onix-bap_config.yml` and `onix-bpp_config.yml` files are **reference copies** for documentation purposes. The actual configurations used by the containers are mounted from `../../../../docker/monolithic/api/config/onix-{bap|bpp}/`.
- All mock services use simplified configurations suitable for testing.
- Production deployments should use proper key management and secure configurations.
- Network service discovery uses Docker container names, which must match the service names in configuration files.

