# Monolithic Architecture - Kafka Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **monolithic architecture** with **Apache Kafka** event streaming.

## Architecture Overview

In this setup, the onix-adapter uses Apache Kafka for high-throughput event streaming. Services communicate through Kafka topics, enabling scalable, distributed message processing.

### Components

- **Redis**: Used for caching and state management
- **Apache Kafka**: Event streaming platform for high-throughput messaging
- **Zookeeper**: (if required) For Kafka coordination and metadata management
- **Onix-Adapter**: Single container handling all BAP/BPP operations via Kafka

## Directory Structure

```
docker/monolithic/kafka/
├── docker-compose-onix-bap-kafka-plugin.yml    # BAP service configuration
├── docker-compose-onix-bpp-kafka-plugin.yml    # BPP service configuration
├── config/
│   └── message-baised/
│       └── kafka/
│           ├── onix-bap/
│           │   └── adapter.yaml                # BAP adapter configuration
│           └── onix-bpp/
│               └── adapter.yaml                # BPP adapter configuration
└── README.md                                    # This file
```

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to onix-adapter Docker images:
  - `manendrapalsingh/onix-bap-plugin-kafka:latest`
  - `manendrapalsingh/onix-bpp-plugin-kafka:latest`
- Kafka server (included in docker-compose)
- Redis server (included in docker-compose)

## Architecture

### BAP Adapter Modules

1. **bapTxnCaller**: Kafka consumer that consumes requests from BAP Backend and routes them to BPP via HTTP
2. **bapTxnReceiver**: HTTP handler that receives callbacks from BPP and publishes them to BAP Backend via Kafka

### BPP Adapter Modules

1. **bppTxnCaller**: Kafka consumer that consumes callbacks from BPP Backend and routes them to BAP/CDS via HTTP
2. **bppTxnReceiver**: HTTP handler that receives requests from BAP adapter and publishes them to BPP Backend via Kafka

## Features

- **Event Streaming**: High-throughput, distributed event processing
- **Scalability**: Kafka's distributed architecture supports horizontal scaling
- **Durability**: Messages are persisted and replicated across Kafka brokers
- **Consumer Groups**: Support for parallel message processing
- **Topic-Based Routing**: Messages routed by topic names
- **Manual Offset Management**: Control over message consumption
- **Phase 1 Support**: Routes discover requests to CDS for aggregation
- **Phase 2+ Support**: Routes requests directly to BPP and receives callbacks

## Quick Start

### For BAP (Buyer App Provider)

1. **Start the BAP services:**
   ```bash
   docker-compose -f docker-compose-onix-bap-kafka-plugin.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bap|onix-bap-plugin-kafka|kafka|zookeeper)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-onix-bap-kafka-plugin.yml logs -f onix-bap-plugin-kafka
   ```

### For BPP (Buyer Platform Provider)

1. **Start the BPP services:**
   ```bash
   docker-compose -f docker-compose-onix-bpp-kafka-plugin.yml up -d
   ```

2. **Verify services are running:**
   ```bash
   docker ps | grep -E "(redis-onix-bpp|onix-bpp-plugin-kafka|kafka|zookeeper)"
   ```

3. **Check logs:**
   ```bash
   docker-compose -f docker-compose-onix-bpp-kafka-plugin.yml logs -f onix-bpp-plugin-kafka
   ```

## Configuration

### Required Services

1. **Kafka**: Event streaming platform (default: `kafka:9092`)
2. **Zookeeper**: (if required) For Kafka coordination (default: `zookeeper:2181`)
3. **Redis**: Cache server 
   - BAP: `redis-onix-bap:6379`
   - BPP: `redis-onix-bpp:6379`
4. **Registry**: Mock registry service (default: `http://mock-registry:3030`)

### Environment Variables

- `CONFIG_FILE`: Path to the adapter configuration file
  - BAP: `/app/config/message-baised/kafka/onix-bap/adapter.yaml`
  - BPP: `/app/config/message-baised/kafka/onix-bpp/adapter.yaml`

### Kafka Configuration

#### Topic Configuration

- **Topic Naming**: Topics follow the pattern `{domain}.{action}` (e.g., `ev_charging_network.discover`)
- **Topic Durability**: Topics are persistent and replicated
- **Partitioning**: Topics can be partitioned for parallel processing
- **Consumer Groups**: Each adapter uses a consumer group for message distribution

#### BAP bapTxnCaller (Kafka Consumer)

The adapter consumes requests from BAP Backend with the following configuration:

- **Topics** (requests from BAP Backend): 
  - `ev_charging_network.discover` - Discover request
  - `ev_charging_network.select`, `ev_charging_network.init`, `ev_charging_network.confirm`, etc. - Other requests
- **Consumer Group**: `bap-caller-group`
- **Offset Management**: Manual or automatic based on configuration
- **Processing**: Messages are processed through configured steps (`validateSchema`, `addRoute`, `sign`)

#### BAP bapTxnReceiver (HTTP Handler + Publisher)

The adapter publishes callbacks to BAP Backend with the following configuration:

- **HTTP Endpoint**: `/bap/receiver/` (receives HTTP requests from BPP adapter)
- **Publishing Topics** (callbacks to BAP Backend):
  - `ev_charging_network.on_discover` - Phase 1 aggregated search results
  - `ev_charging_network.on_select`, `ev_charging_network.on_init`, etc. - Phase 2+ callbacks

#### BPP bppTxnCaller (Kafka Consumer)

The adapter consumes callbacks from BPP Backend with the following configuration:

- **Topics** (callbacks from BPP Backend): 
  - `ev_charging_network.on_discover` - Phase 1 on_discover callback
  - `ev_charging_network.on_select`, `ev_charging_network.on_init`, etc. - Phase 2+ callbacks
- **Consumer Group**: `bpp-caller-group`
- **Offset Management**: Manual or automatic based on configuration

#### BPP bppTxnReceiver (HTTP Handler + Publisher)

The adapter publishes requests to BPP Backend with the following configuration:

- **HTTP Endpoint**: `/bpp/receiver/` (receives HTTP requests from BAP adapter)
- **Publishing Topics** (requests to BPP Backend):
  - `ev_charging_network.discover` - Phase 1 discover request
  - `ev_charging_network.select`, `ev_charging_network.init`, etc. - Phase 2+ requests

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

## Message Flow

### Flow 1: BAP Backend → BPP Backend (Requests)

1. **BAP Backend**: Publishes requests to Kafka topics like `ev_charging_network.discover`, `ev_charging_network.select`, etc.
2. **bapTxnCaller**: Consumes from Kafka topics, processes, routes to BPP adapter via HTTP
3. **bppTxnReceiver**: Receives HTTP request at `/bpp/receiver/`, processes, publishes to Kafka
4. **BPP Backend**: Consumes from Kafka topics like `ev_charging_network.discover`, `ev_charging_network.select`, etc.

### Flow 2: BPP Backend → BAP Backend (Callbacks)

1. **BPP Backend**: Publishes callbacks to Kafka topics like `ev_charging_network.on_discover`, `ev_charging_network.on_select`, etc.
2. **bppTxnCaller**: Consumes from Kafka topics, processes, routes to BAP adapter via HTTP
3. **bapTxnReceiver**: Receives HTTP request at `/bap/receiver/`, processes, publishes to Kafka
4. **BAP Backend**: Consumes callbacks from Kafka topics like `ev_charging_network.on_discover`, `ev_charging_network.on_select`, etc.

## Kafka Management

### Using Kafka CLI Tools

```bash
# List topics
docker exec kafka kafka-topics.sh --list --bootstrap-server localhost:9092

# Create a topic
docker exec kafka kafka-topics.sh --create --topic ev_charging_network.discover --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

# Describe a topic
docker exec kafka kafka-topics.sh --describe --topic ev_charging_network.discover --bootstrap-server localhost:9092

# Consume messages from a topic
docker exec kafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ev_charging_network.discover --from-beginning

# Produce messages to a topic
docker exec -it kafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic ev_charging_network.discover
```

### Consumer Group Management

```bash
# List consumer groups
docker exec kafka kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Describe consumer group
docker exec kafka kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group bap-caller-group

# Reset consumer group offsets
docker exec kafka kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group bap-caller-group --reset-offsets --to-earliest --topic ev_charging_network.discover --execute
```

## Stopping Services

```bash
# Stop BAP services
docker-compose -f docker-compose-onix-bap-kafka-plugin.yml down

# Stop BPP services
docker-compose -f docker-compose-onix-bpp-kafka-plugin.yml down

# Stop both and remove volumes
docker-compose -f docker-compose-onix-bap-kafka-plugin.yml -f docker-compose-onix-bpp-kafka-plugin.yml down -v
```

## Troubleshooting

### Kafka Connection Issues

- Verify Kafka is running: `docker ps | grep kafka`
- Check Kafka logs: `docker logs kafka`
- Verify network connectivity: Ensure plugin container can reach `kafka:9092`
- Check broker configuration: Verify `bootstrap.servers` in adapter config

### Messages Not Consumed

- Verify topics exist: Use `kafka-topics.sh --list` to list topics
- Check consumer group status: Use `kafka-consumer-groups.sh --describe`
- Review adapter logs: `docker logs onix-bap-plugin-kafka` or `docker logs onix-bpp-plugin-kafka`
- Verify topic names match: Check topic names in adapter config match your Backend producer

### Offset Issues

- Check consumer group offsets: Use `kafka-consumer-groups.sh --describe`
- Reset offsets if needed: Use `kafka-consumer-groups.sh --reset-offsets`
- Verify offset management mode: Check if using manual or automatic offset commits

### HTTP Endpoint Issues

- Verify HTTP port is exposed in Docker Compose (8001 for BAP, 8002 for BPP)
- Check HTTP configuration is enabled in `adapter.yaml`
- Verify network connectivity between adapters
- Check adapter logs for HTTP connection errors

### Consumer Issues

1. **No Consumers Connected**:
   - Check adapter logs: `docker logs onix-bap-plugin-kafka` or `docker logs onix-bpp-plugin-kafka`
   - Verify Kafka connection in logs
   - Check network connectivity
   - Verify broker addresses in `adapter.yaml`

2. **Messages Not Being Consumed**:
   - Verify consumer group is active (use Kafka CLI tools)
   - Check if messages are in topics (use `kafka-console-consumer.sh`)
   - Verify topic names match between producer and consumer
   - Check adapter logs for errors

## Customization

### Changing Kafka Connection

Edit the `adapter.yaml` file to update Kafka connection settings:

```yaml
kafkaConsumer:
  config:
    brokers: "kafka:9092"
    groupId: "bap-caller-group"
    topics: "ev_charging_network.discover,ev_charging_network.select"

kafkaProducer:
  config:
    brokers: "kafka:9092"
    topic: "ev_charging_network.on_discover"
```

### Updating Configuration

1. Modify the YAML files in `config/message-baised/kafka/onix-{bap|bpp}/`
2. Restart the services:
   ```bash
   docker-compose -f docker-compose-onix-{bap|bpp}-kafka-plugin.yml restart
   ```

## Next Steps

- For RabbitMQ integration: See [Monolithic RabbitMQ](./../rabbitmq/README.md)
- For API integration: See [Monolithic API](./../api/README.md)
- For microservice architecture: See [Microservice Kafka](./../../microservice/kafka/README.md)

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
