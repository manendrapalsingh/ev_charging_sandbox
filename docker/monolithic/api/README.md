# Monolithic Architecture - API Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **monolithic architecture** with **REST API** communication.

## Architecture Overview

In a monolithic architecture, the onix-adapter runs as a single container service that handles both incoming and outgoing API requests. All communication happens via HTTP/REST API endpoints.

### Components

- **Redis**: Used for caching and state management
- **Onix-Adapter**: Single container handling all BAP/BPP operations
- **API Communication**: Direct HTTP/REST API calls between services

## Directory Structure

```
docker/monolithic/api/
├── docker-compose-bap.yml          # BAP service configuration
├── docker-compose-bpp.yml          # BPP service configuration
├── config/
│   ├── onix-bap/
│   │   ├── adapter.yaml            # BAP adapter configuration
│   │   ├── bap_caller_routing.yaml # BAP caller routing rules
│   │   └── bap_receiver_routing.yaml # BAP receiver routing rules
│   └── onix-bpp/
│       ├── adapter.yaml            # BPP adapter configuration
│       ├── bpp_caller_routing.yaml  # BPP caller routing rules
│       └── bpp_receiver_routing.yaml # BPP receiver routing rules
└── README.md                       # This file
```

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to onix-adapter Docker images:
  - `manendrapalsingh/onix-bap-plugin:latest`
  - `manendrapalsingh/onix-bpp-plugin:latest`
- Schema files mounted at `../schema` (read-only)

## Quick Start

### For BAP (Buyer App Provider)

1. **Start the BAP services:**
   ```bash
   docker-compose -f docker-compose-bap.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bap|onix-bap-plugin)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-bap.yml logs -f onix-bap-plugin
   ```

4. **Access the BAP adapter:**
   - Caller endpoint: `http://localhost:8001/bap/caller/`
   - Receiver endpoint: `http://localhost:8001/bap/receiver/`

### For BPP (Buyer App Provider)

1. **Start the BPP services:**
   ```bash
   docker-compose -f docker-compose-bpp.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bpp|onix-bpp-plugin)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-bpp.yml logs -f onix-bpp-plugin
   ```

4. **Access the BPP adapter:**
   - Caller endpoint: `http://localhost:8002/bpp/caller/`
   - Receiver endpoint: `http://localhost:8002/bpp/receiver/`

## Configuration Details

### BAP Configuration

#### Adapter Configuration (`config/onix-bap/adapter.yaml`)

- **Application**: `onix-ev-charging`
- **HTTP Port**: `8001`
- **Modules**:
  - `bapTxnReceiver`: Receives callbacks from CDS (Phase 1) and BPPs (Phase 2+)
  - `bapTxnCaller`: Entry point for requests from BAP application

#### Routing Configuration

**BAP Caller Routing** (`bap_caller_routing.yaml`):
- Phase 1: `discover` → Routes to CDS for aggregation
- Phase 2+: Other actions (`select`, `init`, `confirm`, etc.) → Routes directly to BPP

**BAP Receiver Routing** (`bap_receiver_routing.yaml`):
- Phase 1: `on_discover` → Routes callbacks to BAP backend
- Phase 2+: Other callbacks → Routes to BAP backend

### BPP Configuration

#### Adapter Configuration (`config/onix-bpp/adapter.yaml`)

- **Application**: `bpp-ev-charging`
- **HTTP Port**: `8002`
- **Modules**:
  - `bppTxnReceiver`: Receives requests from CDS (Phase 1) and BAP-ONIX (Phase 2+)
  - `bppTxnCaller`: Sends responses to CDS/ONIX

#### Routing Configuration

**BPP Caller Routing** (`bpp_caller_routing.yaml`):
- Phase 1: `on_discover` → Routes to CDS for aggregation
- Phase 2+: Other responses → Routes directly to BAP-ONIX

**BPP Receiver Routing** (`bpp_receiver_routing.yaml`):
- Phase 1: `discover` → Routes to BPP backend service
- Phase 2+: Other requests → Routes to BPP backend service

## API Endpoints

### BAP Endpoints

| Endpoint | Purpose | Path |
|----------|---------|------|
| Caller | Send requests to BPP/CDS | `/bap/caller/{action}` |
| Receiver | Receive callbacks from BPP/CDS | `/bap/receiver/{action}` |

**Example:**
- Send discover request: `POST http://localhost:8001/bap/caller/discover`
- Receive callback: `POST http://localhost:8001/bap/receiver/on_discover`

### BPP Endpoints

| Endpoint | Purpose | Path |
|----------|---------|------|
| Caller | Send responses to BAP/CDS | `/bpp/caller/{action}` |
| Receiver | Receive requests from BAP/CDS | `/bpp/receiver/{action}` |

**Example:**
- Receive discover request: `POST http://localhost:8002/bpp/receiver/discover`
- Send response: `POST http://localhost:8002/bpp/caller/on_discover`

## Environment Variables

The adapter uses the following environment variables:

- `CONFIG_FILE`: Path to the adapter configuration file (default: `/app/config/adapter.yaml`)

## Volume Mounts

1. **Config Directory**: `../config/onix-{bap|bpp}:/app/config/onix-{bap|bpp}`
2. **Adapter Config**: `../config/onix-{bap|bpp}/adapter.yaml:/app/config/adapter.yaml:ro`
3. **Schema Directory**: `../schema:/app/schemas:ro`

**Note**: Adjust the relative paths based on your project structure.

## Network Configuration

Both services use the `onix-network` bridge network for inter-container communication:
- Redis services: `redis-onix-bap`, `redis-onix-bpp`
- Onix adapters can communicate with other services on the same network

## Stopping Services

```bash
# Stop BAP services
docker-compose -f docker-compose-bap.yml down

# Stop BPP services
docker-compose -f docker-compose-bpp.yml down

# Stop both and remove volumes
docker-compose -f docker-compose-bap.yml -f docker-compose-bpp.yml down -v
```

## Troubleshooting

### Service Won't Start

1. **Check if ports are available:**
   ```bash
   # Check port 8001 (BAP)
   lsof -i :8001
   
   # Check port 8002 (BPP)
   lsof -i :8002
   
   # Check Redis ports
   lsof -i :6379  # BAP Redis
   lsof -i :6380  # BPP Redis
   ```

2. **Verify Docker images exist:**
   ```bash
   docker images | grep onix
   ```

3. **Check container logs:**
   ```bash
   docker-compose -f docker-compose-bap.yml logs
   docker-compose -f docker-compose-bpp.yml logs
   ```

### Configuration Issues

1. **Verify config files are mounted correctly:**
   ```bash
   docker exec onix-bap-plugin ls -la /app/config/
   docker exec onix-bpp-plugin ls -la /app/config/
   ```

2. **Check adapter configuration:**
   ```bash
   docker exec onix-bap-plugin cat /app/config/adapter.yaml
   ```

### Redis Connection Issues

1. **Verify Redis is healthy:**
   ```bash
   docker exec redis-onix-bap redis-cli ping
   docker exec redis-onix-bpp redis-cli ping
   ```

2. **Check Redis logs:**
   ```bash
   docker-compose -f docker-compose-bap.yml logs redis-onix-bap
   ```

## Customization

### Changing Ports

Edit the `ports` section in `docker-compose-{bap|bpp}.yml`:

```yaml
ports:
  - "YOUR_PORT:8001"  # For BAP
  - "YOUR_PORT:8002"  # For BPP
```

### Updating Configuration

1. Modify the YAML files in `config/onix-{bap|bpp}/`
2. Restart the services:
   ```bash
   docker-compose -f docker-compose-{bap|bpp}.yml restart
   ```

### Using Custom Images

Update the `image` field in `docker-compose-{bap|bpp}.yml`:

```yaml
image: your-registry/onix-{bap|bpp}-plugin:your-tag
```

## Next Steps

- For RabbitMQ integration: See [Monolithic RabbitMQ](./../rabbitmq/README.md)
- For Kafka integration: See [Monolithic Kafka](./../kafka/README.md)
- For microservice architecture: See [Microservice API](./../../microservice/api/README.md)

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)

