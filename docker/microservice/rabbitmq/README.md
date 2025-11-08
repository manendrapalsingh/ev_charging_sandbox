# Microservice Architecture - RabbitMQ Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **microservice architecture** with **RabbitMQ** message broker communication.

## Architecture Overview

In a microservice architecture with RabbitMQ, the onix-adapter uses RabbitMQ for asynchronous message processing. Each adapter consists of two modules that work together to handle bidirectional communication between BAP and BPP applications.

### Components

- **Redis**: Shared caching service for each adapter
- **RabbitMQ**: Message queue server for asynchronous communication
- **Onix-Adapter Services**: BAP and BPP adapters with RabbitMQ integration
- **Message-Based Communication**: Queue-based consumption and publishing

## Directory Structure

```
docker/microservice/rabbitmq/
├── docker-compose-onix-bap-rabbit-mq-plugin.yml    # BAP service configuration
├── docker-compose-onix-bpp-rabbit-mq-plugin.yml    # BPP service configuration
├── config/
│   └── message-baised/
│       └── rabbit-mq/
│           ├── onix-bap/
│           │   └── adapter.yaml                    # BAP adapter configuration
│           └── onix-bpp/
│               └── adapter.yaml                    # BPP adapter configuration
└── README.md                                        # This file
```

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to onix-adapter Docker images:
  - `manendrapalsingh/onix-bap-plugin-rabbit-mq:latest`
  - `manendrapalsingh/onix-bpp-plugin-rabbit-mq:latest`
- RabbitMQ server (included in docker-compose)
- Redis server (included in docker-compose)

## Architecture

### BAP Adapter Modules

1. **bapTxnCaller**: Queue consumer that consumes requests from BAP Backend and routes them to BPP via HTTP
2. **bapTxnReceiver**: HTTP handler that receives callbacks from BPP and publishes them to BAP Backend via RabbitMQ

### BPP Adapter Modules

1. **bppTxnCaller**: Queue consumer that consumes callbacks from BPP Backend and routes them to BAP/CDS via HTTP
2. **bppTxnReceiver**: HTTP handler that receives requests from BAP adapter and publishes them to BPP Backend via RabbitMQ

## Features

- **Dual-Mode Operation**: 
  - Queue consumer for requests/callbacks from Backend
  - HTTP handler for callbacks/requests from adapter

- **Queue-Based Message Consumption**: Consumes messages from Backend via RabbitMQ

- **HTTP-Based Request/Callback Reception**: Receives requests/callbacks from adapter via HTTP

- **Message Publishing**: Publishes messages to Backend via RabbitMQ

- **Manual ACK/NACK**: Reliable message processing with retry capability

- **Phase 1 Support**: Routes discover requests to CDS for aggregation

- **Phase 2+ Support**: Routes requests directly to BPP and receives callbacks

- **External Configuration**: Config files are mounted from the host

- **Plugin Support**: All required plugins are built and included

## Pre-built Images

Pre-built images are available from Docker Hub:

- `manendrapalsingh/onix-bap-plugin-rabbit-mq:latest`
- `manendrapalsingh/onix-bap-plugin-rabbit-mq:sha-{commit-sha}`
- `manendrapalsingh/onix-bpp-plugin-rabbit-mq:latest`
- `manendrapalsingh/onix-bpp-plugin-rabbit-mq:sha-{commit-sha}`

## Quick Start

### For BAP (Buyer App Provider)

1. **Start the BAP services:**
   ```bash
   docker-compose -f docker-compose-onix-bap-rabbit-mq-plugin.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bap|onix-bap-plugin-rabbitmq|rabbitmq)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-onix-bap-rabbit-mq-plugin.yml logs -f onix-bap-plugin-rabbitmq
   ```

### For BPP (Buyer Platform Provider)

1. **Start the BPP services:**
   ```bash
   docker-compose -f docker-compose-onix-bpp-rabbit-mq-plugin.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bpp|onix-bpp-plugin-rabbitmq|rabbitmq)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-onix-bpp-rabbit-mq-plugin.yml logs -f onix-bpp-plugin-rabbitmq
   ```

## Configuration

### Required Services

1. **RabbitMQ**: Message queue server (default: `rabbitmq:5672`)
2. **Redis**: Cache server 
   - BAP: `redis-onix-bap:6379`
   - BPP: `redis-onix-bpp:6379`
3. **Registry**: Mock registry service (default: `http://mock-registry:3030`)

### Environment Variables

- `CONFIG_FILE`: Path to the adapter configuration file
  - BAP: `/app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml`
  - BPP: `/app/config/message-baised/rabbit-mq/onix-bpp/adapter.yaml`

### RabbitMQ Credentials Configuration

**RabbitMQ credentials are configured in the `adapter.yaml` file**, not via environment variables. Each publisher and consumer plugin requires `username` and `password` in its configuration:

```yaml
# Publisher configuration
publisher:
  id: publisher
  config:
    addr: rabbitmq:5672
    exchange: beckn_exchange
    durable: "true"
    username: admin
    password: admin

# RabbitMQ Consumer configuration
rabbitmqConsumer:
  id: rabbitmqconsumer
  config:
    addr: rabbitmq:5672
    exchange: beckn_exchange
    routingKeys: "bap.discover,bap.select,..."
    queueName: "bap_caller_queue"
    durable: "true"
    autoDelete: "false"
    exclusive: "false"
    noWait: "false"
    autoAck: "false"
    prefetchCount: "10"
    consumerThreads: "2"
    queueArgs: ""
    username: admin
    password: admin
```

**Note**: The default credentials in the provided configuration files are `admin`/`admin` to match the Docker Compose setup. For production deployments, replace these with your actual RabbitMQ credentials and consider using secret management tools to inject credentials securely.

### HTTP Configuration

**HTTP server is required** for the Receiver modules to receive HTTP requests. The HTTP configuration must be enabled in `adapter.yaml`:

**BAP Adapter:**
```yaml
http:
  port: 8001
  timeout:
    read: 30
    write: 30
    idle: 30
```

**BPP Adapter:**
```yaml
http:
  port: 8002
  timeout:
    read: 30
    write: 30
    idle: 30
```

- **BAP Port**: 8001 (must be exposed in Docker Compose)
- **BPP Port**: 8002 (must be exposed in Docker Compose)
- **Purpose**: Enables Receiver modules to receive HTTP requests/callbacks
- **Required**: Yes - The Receiver modules use handler type `std` which requires an HTTP server

### RabbitMQ Configuration

#### Exchange Configuration

- **Exchange Name**: `beckn_exchange`
- **Exchange Type**: Topic exchange (allows routing based on routing keys)
- **Durability**: Durable (survives broker restarts)

#### BAP bapTxnCaller (Queue Consumer)

The adapter consumes requests from BAP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)
- **Queue**: `bap_caller_queue` (durable)
- **Routing Keys** (requests from BAP Backend): 
  - `bap.discover` - Discover request
  - `bap.select`, `bap.init`, `bap.confirm`, `bap.status`, `bap.track`, `bap.cancel`, `bap.update`, `bap.rating`, `bap.support` - Other requests
- **Acknowledgment**: Manual (`autoAck: false`)
- **Prefetch Count**: 10
- **Consumer Threads**: 2

#### BAP bapTxnReceiver (HTTP Handler + Publisher)

The adapter publishes callbacks to BAP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)
- **HTTP Endpoint**: `/bap/receiver/` (receives HTTP requests from BPP adapter)
- **Publishing Routing Keys** (callbacks to BAP Backend):
  - `bap.on_discover` - Phase 1 aggregated search results
  - `bap.on_select`, `bap.on_init`, `bap.on_confirm`, `bap.on_status`, `bap.on_track`, `bap.on_cancel`, `bap.on_update`, `bap.on_rating`, `bap.on_support` - Phase 2+ callbacks

#### BPP bppTxnCaller (Queue Consumer)

The adapter consumes callbacks from BPP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)
- **Queue**: `bpp_caller_queue` (durable)
- **Routing Keys** (callbacks from BPP Backend): 
  - `bpp.on_discover` - Phase 1 on_discover callback
  - `bpp.on_select`, `bpp.on_init`, `bpp.on_confirm`, `bpp.on_status`, `bpp.on_track`, `bpp.on_cancel`, `bpp.on_update`, `bpp.on_rating`, `bpp.on_support` - Phase 2+ callbacks
- **Acknowledgment**: Manual (`autoAck: false`)
- **Prefetch Count**: 10
- **Consumer Threads**: 2

#### BPP bppTxnReceiver (HTTP Handler + Publisher)

The adapter publishes requests to BPP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)
- **HTTP Endpoint**: `/bpp/receiver/` (receives HTTP requests from BAP adapter)
- **Publishing Routing Keys** (requests to BPP Backend):
  - `bpp.discover` - Phase 1 discover request
  - `bpp.select`, `bpp.init`, `bpp.confirm`, `bpp.status`, `bpp.track`, `bpp.cancel`, `bpp.update`, `bpp.rating`, `bpp.support` - Phase 2+ requests

## Message Flow

### Flow 1: BAP Backend → BPP Backend (Requests)

1. **BAP Backend**: Publishes requests to `beckn_exchange` with routing keys like `bap.discover`, `bap.select`, etc.
2. **bapTxnCaller**: Consumes from `bap_caller_queue`, processes, routes to BPP adapter via HTTP
3. **bppTxnReceiver**: Receives HTTP request at `/bpp/receiver/`, processes, publishes to RabbitMQ
4. **BPP Backend**: Consumes from queues bound to routing keys like `bpp.discover`, `bpp.select`, etc.

### Flow 2: BPP Backend → BAP Backend (Callbacks)

1. **BPP Backend**: Publishes callbacks to `beckn_exchange` with routing keys like `bpp.on_discover`, `bpp.on_select`, etc.
2. **bppTxnCaller**: Consumes from `bpp_caller_queue`, processes, routes to BAP adapter via HTTP
3. **bapTxnReceiver**: Receives HTTP request at `/bap/receiver/`, processes, publishes to RabbitMQ
4. **BAP Backend**: Consumes callbacks from queues bound to routing keys like `bap.on_discover`, `bap.on_select`, etc.

## RabbitMQ Management UI

The RabbitMQ Management Plugin is enabled by default in the Docker Compose setup and provides a web-based UI for monitoring and managing RabbitMQ.

### Accessing the Management UI

1. **Start the services**:
   ```bash
   docker-compose -f docker-compose-onix-bap-rabbit-mq-plugin.yml up -d
   ```

2. **Open the Management UI**:
   - URL: `http://localhost:15672`
   - Username: `admin` (default, as configured in docker-compose)
   - Password: `admin` (default, as configured in docker-compose)

### Key Features for Monitoring

- **Queues Tab**: Monitor queue status, message counts, and consumer connections
- **Exchanges Tab**: Monitor message publishing rates and bindings
- **Connections Tab**: Monitor adapter connections
- **Channels Tab**: Monitor consumer channels and prefetch settings

## Stopping Services

```bash
# Stop BAP services
docker-compose -f docker-compose-onix-bap-rabbit-mq-plugin.yml down

# Stop BPP services
docker-compose -f docker-compose-onix-bpp-rabbit-mq-plugin.yml down

# Stop both and remove volumes
docker-compose -f docker-compose-onix-bap-rabbit-mq-plugin.yml -f docker-compose-onix-bpp-rabbit-mq-plugin.yml down -v
```

## Troubleshooting

### RabbitMQ Connection Issues

- Verify RabbitMQ is running: `docker ps | grep rabbitmq`
- Check RabbitMQ logs: `docker logs rabbitmq`
- Verify network connectivity: Ensure plugin container can reach `rabbitmq:5672`
- Check credentials: Ensure `username` and `password` are configured in `adapter.yaml` for both `publisher` and `rabbitmqConsumer` plugins

### Messages Not Consumed

- Verify queue exists: Access RabbitMQ Management UI at `http://localhost:15672`
- Check queue bindings: Ensure queue is bound to exchange with correct routing key
- Review adapter logs: `docker logs onix-bap-plugin-rabbitmq` or `docker logs onix-bpp-plugin-rabbitmq`
- Verify routing keys match: Check `routingKeys` in adapter config match your Backend producer

### ACK/NACK Issues

- Check adapter logs for processing errors
- Verify message format matches expected schema
- Check signature validation if `validateSign` step is enabled
- Messages with errors will be NACKed and requeued automatically

### HTTP Endpoint Issues

- Verify HTTP port is exposed in Docker Compose (8001 for BAP, 8002 for BPP)
- Check HTTP configuration is enabled in `adapter.yaml`
- Verify network connectivity between adapters
- Check adapter logs for HTTP connection errors

### Consumer Issues

1. **No Consumers Connected**:
   - Check adapter logs: `docker logs onix-bap-plugin-rabbitmq` or `docker logs onix-bpp-plugin-rabbitmq`
   - Verify RabbitMQ connection in logs
   - Check network connectivity
   - Verify credentials in `adapter.yaml` match RabbitMQ server credentials
   - Use RabbitMQ Management UI → Queues → Check "Consumers" column

2. **Messages Not Being Consumed**:
   - Verify consumers are connected (Management UI → Queues → Consumers column)
   - Check if messages are in "Ready" state
   - Verify routing keys match queue bindings
   - Check adapter logs for errors
   - Ensure queue name in config matches the queue name in RabbitMQ

## Customization

### Changing RabbitMQ Connection

Edit the `adapter.yaml` file to update RabbitMQ connection settings:

```yaml
publisher:
  config:
    addr: your-rabbitmq-host:5672
    username: your-username
    password: your-password

rabbitmqConsumer:
  config:
    addr: your-rabbitmq-host:5672
    username: your-username
    password: your-password
```

### Updating Configuration

1. Modify the YAML files in `config/message-baised/rabbit-mq/onix-{bap|bpp}/`
2. Restart the services:
   ```bash
   docker-compose -f docker-compose-onix-{bap|bpp}-rabbit-mq-plugin.yml restart
   ```

## Next Steps

- For API integration: See [Microservice API](./../api/README.md)
- For Kafka integration: See [Microservice Kafka](./../kafka/README.md)
- For monolithic architecture: See [Monolithic RabbitMQ](./../../monolithic/rabbitmq/README.md)

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
