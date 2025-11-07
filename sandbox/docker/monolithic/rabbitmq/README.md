# EV Charging Sandbox - RabbitMQ Docker Compose Setup

This directory contains Docker Compose configurations for setting up a complete EV Charging sandbox environment with RabbitMQ message broker integration. The setup includes ONIX adapters (BAP and BPP), mock services (CDS, Registry, BAP-RabbitMQ, BPP-RabbitMQ), and supporting infrastructure.

## Architecture Overview

This setup creates a fully functional sandbox environment for testing and developing EV Charging network integrations using the ONIX protocol with RabbitMQ for asynchronous message processing. The architecture includes:

- **ONIX RabbitMQ Adapters**: Protocol adapters for BAP (Buyer App Provider) and BPP (Buyer Platform Provider) that consume and publish messages via RabbitMQ
- **Mock Services**: Simulated services for testing without real implementations
- **RabbitMQ Message Broker**: Central message broker for asynchronous communication
- **Supporting Services**: Redis for caching and state management

## Services

### Core Services

1. **rabbitmq** (Ports: 5672 AMQP, 15672 Management UI)
   - RabbitMQ message broker for asynchronous communication
   - Used for message routing between adapters and mock services
   - Management UI available at `http://localhost:15672` (admin/admin)

2. **redis-bap** (Port: 6379)
   - Redis cache for the BAP adapter
   - Used for storing transaction state, caching registry lookups, and session management

3. **redis-bpp** (Port: 6380)
   - Redis cache for the BPP adapter
   - Used for storing transaction state, caching registry lookups, and session management

4. **onix-bap-plugin-rabbitmq**
   - ONIX protocol adapter for BAP (Buyer App Provider) with RabbitMQ integration
   - **Queue Consumer**: Consumes messages from `bap_receiver_queue` with routing keys:
     - `bap.on_discover`, `bap.on_select`, `bap.on_init`, `bap.on_confirm`, `bap.on_status`, `bap.on_track`, `bap.on_cancel`, `bap.on_update`, `bap.on_rating`, `bap.on_support`
   - **Message Publisher**: Publishes messages to RabbitMQ exchange for routing to BAP backend
   - Handles protocol compliance, signing, validation, and routing for BAP transactions
   - **Note**: Runs in queue-only mode - no HTTP ports exposed

5. **onix-bpp-plugin-rabbitmq** (Port: 8002)
   - ONIX protocol adapter for BPP (Buyer Platform Provider) with RabbitMQ integration
   - **HTTP Handler** (`bppTxnReceiver`): Receives HTTP requests from BAP adapter at `/bpp/receiver/` and publishes to RabbitMQ
   - **Queue Consumer** (`bppTxnCaller`): Consumes callbacks from `bpp_caller_queue` with routing keys:
     - `bpp.on_discover`, `bpp.on_select`, `bpp.on_init`, `bpp.on_confirm`, `bpp.on_status`, `bpp.on_track`, `bpp.on_cancel`, `bpp.on_update`, `bpp.on_rating`, `bpp.on_support`
   - Routes callbacks to BAP adapter or CDS via HTTP
   - Handles protocol compliance, signing, validation, and routing for BPP transactions

### Mock Services

6. **mock-registry** (Port: 3030)
   - Mock implementation of the network registry service
   - Maintains a registry of all BAPs, BPPs, and CDS services on the network
   - Provides subscriber lookup and key management functionality

7. **mock-cds** (Port: 8082)
   - Mock Catalog Discovery Service (CDS)
   - Aggregates discover requests from BAPs and broadcasts to registered BPPs
   - Collects and aggregates responses from multiple BPPs
   - Handles signature verification and signing

8. **mock-bap-rabbit-mq** (Internal Port: 9003)
   - Mock BAP backend service with RabbitMQ integration
   - Simulates a Buyer App Provider application
   - Consumes messages from RabbitMQ queues (routing keys: `bap.on_discover`, `bap.on_select`, etc.)
   - Publishes requests to RabbitMQ for ONIX adapter processing
   - **Note**: Runs in queue-only mode - no external HTTP ports exposed

9. **mock-bpp-rabbit-mq** (Internal Port: 9004)
   - Mock BPP backend service with RabbitMQ integration
   - Simulates a Buyer Platform Provider application
   - Consumes messages from RabbitMQ queues (routing keys: `bpp.discover`, `bpp.select`, etc.)
   - Publishes responses to RabbitMQ for ONIX adapter processing
   - **Note**: Runs in queue-only mode - no external HTTP ports exposed

## Configuration Files

Each service has its own configuration file with a service name prefix. This section explains what each config file is used for:

### 1. ONIX BAP RabbitMQ Configuration (`docker/monolithic/rabbitmq/config/onix-bap/`)

**Purpose**: Complete adapter configuration for the ONIX BAP RabbitMQ plugin.

**Usage**: The actual configuration files are mounted from `../../../../docker/monolithic/rabbitmq/config/onix-bap/`. This directory contains:

- **`adapter.yaml`**: Main adapter configuration file
- **`bapTxnCaller-routing.yaml`**: Routing rules for outgoing requests from BAP application
- **`bapTxnReciever-routing.yaml`**: Routing rules for incoming callbacks to BAP
- **`plugin.yaml`**: Plugin-specific configuration

**Key Sections in `adapter.yaml`**:
- **`appName`**: Application identifier ("onix-ev-charging")
- **`log`**: Logging configuration (level, destinations, context keys)
- **`modules`**: Two main modules:
  - **`bapTxnReceiver`**: Handles incoming callbacks from CDS/BPPs via RabbitMQ
    - Consumes from `bap_receiver_queue` with routing keys for all callback actions
    - Validates signatures
    - Routes to backend via RabbitMQ publisher
    - Validates schemas
  - **`bapTxnCaller`**: Handles outgoing requests from BAP application
    - Routes requests using `bapTxnCaller-routing.yaml`
    - Signs requests
    - Publishes to RabbitMQ exchange
- **`plugins`**: Plugin configuration including:
  - Registry connection to mock-registry
  - Key manager for signing/encryption
  - Redis cache configuration
  - Schema validator
  - RabbitMQ consumer (for receiving messages)
  - RabbitMQ publisher (for sending messages)
  - Router for request routing

**Routing Configuration Files**:
- **`bapTxnCaller-routing.yaml`**: Defines routing rules for outgoing requests:
  - Phase 1: `discover` → Routes to CDS via HTTP (`http://mock-cds:8082/csd`)
  - Phase 2+: Other actions (`select`, `init`, `confirm`, etc.) → Routes directly to BPP using context endpoint
- **`bapTxnReciever-routing.yaml`**: Defines routing rules for incoming callbacks:
  - All callbacks (`on_discover`, `on_select`, `on_init`, etc.) → Routes to RabbitMQ publisher with routing keys like `bap.on_discover`, `bap.on_select`, etc.

**Note**: The adapter runs in queue-only mode - no HTTP ports are exposed. All communication happens via RabbitMQ queues.

### 2. ONIX BPP RabbitMQ Configuration (`docker/monolithic/rabbitmq/config/onix-bpp/`)

**Purpose**: Complete adapter configuration for the ONIX BPP RabbitMQ plugin.

**Usage**: The actual configuration files are mounted from `../../../../docker/monolithic/rabbitmq/config/onix-bpp/`. This directory contains:

- **`adapter.yaml`**: Main adapter configuration file
- **`bppTxnCaller-routing.yaml`**: Routing rules for outgoing responses from BPP
- **`bppTxnReciever-routing.yaml`**: Routing rules for incoming requests to BPP
- **`plugin.yaml`**: Plugin-specific configuration

**Key Sections in `adapter.yaml`**:
- **`appName`**: Application identifier ("bpp-ev-charging")
- **`log`**: Logging configuration
- **`http`**: HTTP server configuration (port: 8002)
- **`modules`**: Two main modules:
  - **`bppTxnReceiver`**: HTTP handler that receives requests from BAP adapter
    - HTTP endpoint: `/bpp/receiver/` (receives HTTP requests from BAP adapter)
    - Validates signatures
    - Validates schemas
    - Publishes to RabbitMQ with routing keys (`bpp.discover`, `bpp.select`, etc.)
  - **`bppTxnCaller`**: Queue consumer that consumes callbacks from BPP Backend
    - Consumes from `bpp_caller_queue` with routing keys (`bpp.on_discover`, `bpp.on_select`, etc.)
    - Validates schemas
    - Routes to BAP adapter or CDS via HTTP using `bppTxnCaller-routing.yaml`
    - Signs responses before forwarding
- **`plugins`**: Similar plugin configuration as BAP, but for BPP role

**Routing Configuration Files**:
- **`bppTxnCaller-routing.yaml`**: Defines routing rules for callbacks consumed from BPP Backend:
  - Phase 1: `on_discover` → Routes to CDS via HTTP (`http://mock-cds:8082/csd`) for aggregation
  - Phase 2+: Other responses (`on_select`, `on_init`, `on_confirm`, etc.) → Routes directly to BAP adapter via HTTP
- **`bppTxnReciever-routing.yaml`**: Defines routing rules for requests received from BAP adapter:
  - Phase 1: `discover` → Routes to RabbitMQ publisher with routing key `bpp.discover`
  - Phase 2+: Other requests (`select`, `init`, `confirm`, etc.) → Routes to RabbitMQ publisher with routing keys like `bpp.select`, `bpp.init`, etc.

**Note**: The adapter exposes HTTP port 8002 for `bppTxnReceiver` endpoint. `bppTxnCaller` consumes callbacks from RabbitMQ queues.

### 3. `mock-registry_config.yml`

**Purpose**: Configuration for the mock registry service that maintains the network participant registry.

**Usage**: Mounted into the mock-registry container at `/app/config/config.yaml`.

**Key Sections**:
- **`server.port`**: Server port (3030)
- **`subscribers`**: List of all network participants:
  - **BAPs**: mock-bap-rabbit-mq, onix-bap, example-bap.com
  - **BPPs**: mock-bpp-rabbit-mq, onix-bpp, example-bpp.com, chargezone-energy-bpp.com
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

### 5. `mock-bap-rabbitMq_config.yml`

**Purpose**: Configuration for the mock BAP backend service with RabbitMQ integration.

**Usage**: Mounted into the mock-bap-rabbit-mq container at `/app/config/config.yaml`.

**Key Sections**:
- **`rabbitmq`**: RabbitMQ connection configuration:
  - `url`: RabbitMQ connection URL (`amqp://admin:admin@rabbitmq:5672/`)
  - `exchange`: Exchange name for message routing (`beckn_exchange`)
- **`server.port`**: Internal server port (9003) - used for health checks only
- **`defaults`**: Default values:
  - `version`: Protocol version ("1.0.0")
  - `country_code`: "IND"
  - `city_code`: "std:080"

**When to Modify**:
- Change RabbitMQ connection details
- Update exchange name
- Change server port (internal only)
- Update default version or location codes

### 6. `mock-bpp-rabbitMq_config.yml`

**Purpose**: Configuration for the mock BPP backend service with RabbitMQ integration.

**Usage**: Mounted into the mock-bpp-rabbit-mq container at `/app/config/config.yaml`.

**Key Sections**:
- **`rabbitmq`**: RabbitMQ connection configuration:
  - `url`: RabbitMQ connection URL (`amqp://admin:admin@rabbitmq:5672/`)
  - `exchange`: Exchange name for message routing (`beckn_exchange`)
- **`server.port`**: Internal server port (9004) - used for health checks only
- **`defaults`**: Default values:
  - `version`: Protocol version ("1.0.0")
  - `country_code`: "IND"
  - `city_code`: "std:080"

**When to Modify**:
- Change RabbitMQ connection details
- Update exchange name
- Change server port (internal only)
- Update default version or location codes

## Docker Compose Files

This sandbox setup includes a unified Docker Compose file that runs all services together:

### 1. `docker-compose.yml` (Unified Sandbox)

**Purpose**: Unified Docker Compose configuration for running the complete RabbitMQ sandbox environment with all services.

**Services Included**:
- `rabbitmq`: RabbitMQ message broker
- `redis-bap`: Redis cache for BAP adapter
- `redis-bpp`: Redis cache for BPP adapter
- `onix-bap-plugin-rabbitmq`: ONIX BAP adapter with RabbitMQ integration
- `onix-bpp-plugin-rabbitmq`: ONIX BPP adapter with RabbitMQ integration
- `mock-registry`: Mock registry service
- `mock-cds`: Mock Catalog Discovery Service
- `mock-bap-rabbit-mq`: Mock BAP backend with RabbitMQ integration
- `mock-bpp-rabbit-mq`: Mock BPP backend with RabbitMQ integration

**Usage**:
```bash
cd sandbox/docker/monolithic/rabbitmq
docker-compose up -d
```

### 2. Separate Docker Compose Files (Alternative)

If you prefer to run services separately, you can use the individual compose files:

**`docker-compose-onix-bap-rabbit-mq-plugin.yml`** (in `docker/monolithic/rabbitmq/`):
- Docker Compose configuration for running the ONIX BAP RabbitMQ plugin with supporting services
- Services: `rabbitmq`, `redis-bap`, `onix-bap-plugin-rabbitmq`

**Usage**:
```bash
cd sandbox/docker/monolithic/rabbitmq
docker-compose -f ../../../../docker/monolithic/rabbitmq/docker-compose-onix-bap-rabbit-mq-plugin.yml up -d
```

**`docker-compose-onix-bpp-rabbit-mq-plugin.yml`** (in `docker/monolithic/rabbitmq/`):
- Docker Compose configuration for running the ONIX BPP RabbitMQ plugin with supporting services
- Services: `rabbitmq`, `redis-bpp`, `onix-bpp-plugin-rabbitmq`

**Usage**:
```bash
cd sandbox/docker/monolithic/rabbitmq
docker-compose -f ../../../../docker/monolithic/rabbitmq/docker-compose-onix-bpp-rabbit-mq-plugin.yml up -d
```

**`mock-bap-rabbitMq/docker-compose.mock-bap-rabbitMq.yml`**:
- Sets up mock BAP service with RabbitMQ integration
- Includes RabbitMQ broker and Redis for the mock service

**`mock-bpp-rabbitMq/docker-compose.mock-bpp-rabbitMq.yml`**:
- Sets up mock BPP service with RabbitMQ integration
- Includes RabbitMQ broker and Redis for the mock service

## Quick Start

### Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to Docker images (pulled automatically)

### Starting All Services

#### Option 1: Unified Docker Compose (Recommended)

The easiest way to start the complete sandbox environment:

```bash
# Navigate to this directory
cd sandbox/docker/monolithic/rabbitmq

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Check service status
docker-compose ps
```

#### Option 2: Start Services Separately

If you prefer to start services in stages:

**Step 1: Start Supporting Services (Registry and CDS)**

```bash
# Navigate to rabbitmq sandbox directory
cd sandbox/docker/monolithic/rabbitmq

# Start registry and CDS services
docker-compose up -d mock-registry mock-cds
```

**Step 2: Start Infrastructure Services**

```bash
# Start RabbitMQ and Redis
docker-compose up -d rabbitmq redis-bap redis-bpp
```

**Step 3: Start ONIX Adapters**

```bash
# Start ONIX BAP and BPP plugins
docker-compose up -d onix-bap-plugin-rabbitmq onix-bpp-plugin-rabbitmq
```

**Step 4: Start Mock Services**

```bash
# Start mock BAP and BPP RabbitMQ services
docker-compose up -d mock-bap-rabbit-mq mock-bpp-rabbit-mq
```

### Viewing Logs

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f onix-bap-plugin-rabbitmq
docker-compose logs -f mock-bap-rabbit-mq
```

### Checking Service Status

```bash
# Check all services
docker-compose ps

# Check RabbitMQ management UI
# Open http://localhost:15672 (admin/admin)
```

### Stopping Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Stop specific services
docker-compose stop onix-bap-plugin-rabbitmq
```

## Service Endpoints

Once all services are running, you can access:

| Service | Endpoint | Description |
|---------|----------|-------------|
| **RabbitMQ Management** | `http://localhost:15672` | RabbitMQ Management UI (admin/admin) |
| **Mock Registry** | `http://localhost:3030` | Registry service |
| | `http://localhost:3030/lookup` | Lookup subscribers |
| **Mock CDS** | `http://localhost:8082` | Catalog Discovery Service |
| | `http://localhost:8082/csd` | CDS aggregation endpoint |
| **ONIX BAP Plugin** | Queue-based | Consumes from `bap_receiver_queue` |
| **ONIX BPP Plugin** | Queue-based | Consumes from `bpp_receiver_queue` |
| **Mock BAP RabbitMQ** | Queue-based | Consumes from RabbitMQ queues |
| **Mock BPP RabbitMQ** | Queue-based | Consumes from RabbitMQ queues |

## Message Flow

### Service Discovery Flow (Phase 1)

1. **BAP Application** → Publishes `discover` request to RabbitMQ with routing key
2. **ONIX BAP Plugin** → Consumes message, routes to **Mock CDS** via HTTP
3. **Mock CDS** → Broadcasts discover to all registered BPPs
4. **ONIX BPP Plugin** → Receives discover from CDS, publishes to RabbitMQ with routing key `bpp.discover`
5. **Mock BPP RabbitMQ** → Consumes `bpp.discover`, processes, publishes `on_discover` response
6. **ONIX BPP Plugin** → Routes `on_discover` response to **Mock CDS** via HTTP
7. **Mock CDS** → Aggregates responses, sends to **ONIX BAP Plugin**
8. **ONIX BAP Plugin** → Publishes aggregated response to RabbitMQ with routing key `bap.on_discover`
9. **Mock BAP RabbitMQ** → Consumes `bap.on_discover` callback

### Transaction Flow (Phase 2+)

1. **BAP Application** → Publishes `select/init/confirm` request to RabbitMQ
2. **ONIX BAP Plugin** → Consumes message, routes directly to **ONIX BPP Plugin** (bypasses CDS)
3. **ONIX BPP Plugin** → Publishes to RabbitMQ with routing key `bpp.select/bpp.init/bpp.confirm`
4. **Mock BPP RabbitMQ** → Consumes request, processes, publishes response
5. **ONIX BPP Plugin** → Routes callback to **ONIX BAP Plugin**
6. **ONIX BAP Plugin** → Publishes callback to RabbitMQ with routing key `bap.on_select/bap.on_init/bap.on_confirm`
7. **Mock BAP RabbitMQ** → Consumes callback

## Network Architecture

All services run on shared Docker networks allowing:
- Service-to-service communication using container names as hostnames
- Isolated networking from other Docker containers
- Easy service discovery without IP address management
- RabbitMQ message routing across services

**Networks Used**:
- `onix-network`: Used by ONIX adapters and mock services
- `mock-network`: Used by standalone mock services

## RabbitMQ Queue Structure

### Exchange
- **Name**: `beckn_exchange`
- **Type**: Topic exchange (allows routing based on routing keys)

### BAP Queues and Routing Keys
- **Queue**: `bap_receiver_queue`
- **Routing Keys**:
  - `bap.on_discover`
  - `bap.on_select`
  - `bap.on_init`
  - `bap.on_confirm`
  - `bap.on_status`
  - `bap.on_track`
  - `bap.on_cancel`
  - `bap.on_update`
  - `bap.on_rating`
  - `bap.on_support`

### BPP Queues and Routing Keys
- **Queue**: `bpp_receiver_queue`
- **Routing Keys**:
  - `bpp.discover`
  - `bpp.select`
  - `bpp.init`
  - `bpp.confirm`
  - `bpp.status`
  - `bpp.track`
  - `bpp.cancel`
  - `bpp.update`
  - `bpp.rating`
  - `bpp.support`

## RabbitMQ Management UI

The RabbitMQ Management Plugin is enabled by default and provides a web-based UI for monitoring and managing RabbitMQ. This is essential for testing consumer behavior, monitoring queues, and debugging message flow.

### Accessing the Management UI

1. **Start the services**:
   ```bash
   cd sandbox/docker/monolithic/rabbitmq
   docker-compose up -d
   ```

2. **Open the Management UI**:
   - URL: `http://localhost:15672`
   - Username: `admin`
   - Password: `admin`

### Key Features for Testing Consumer Behavior

#### 1. **Overview Dashboard**
   - **Location**: Home page after login
   - **What to Monitor**:
     - Total messages published/consumed
     - Message rates (messages per second)
     - Connection count
     - Queue count
     - Consumer count

#### 2. **Queues Tab** - Monitor Queue Status
   - **Location**: Click "Queues" in the top navigation
   - **Key Queues to Monitor**:
     - `bap_receiver_queue` - Messages consumed by ONIX BAP plugin
     - `bpp_receiver_queue` - Messages consumed by ONIX BPP plugin
   
   **What to Check**:
   - **Ready**: Number of messages waiting to be consumed
   - **Unacked**: Messages being processed (not yet acknowledged)
   - **Total**: Total messages that have passed through the queue
   - **Consumers**: Number of active consumers connected to the queue
   - **Consumer utilization**: How efficiently consumers are processing messages
   - **Message rates**: Messages per second being published/consumed

   **Testing Consumer Behavior**:
   - Watch the "Ready" count decrease as consumers process messages
   - Monitor "Unacked" to see messages currently being processed
   - If "Ready" grows, consumers may be slower than message producers
   - If "Unacked" stays high, consumers may be stuck or processing slowly

#### 3. **Exchanges Tab** - Monitor Message Publishing
   - **Location**: Click "Exchanges" in the top navigation
   - **Key Exchange**: `beckn_exchange`
   
   **What to Check**:
   - **Publish rate**: Messages per second being published
   - **Bindings**: See which queues are bound to which routing keys
   - Click on `beckn_exchange` to see all bound queues and routing keys

#### 4. **Connections Tab** - Monitor Consumer Connections
   - **Location**: Click "Connections" in the top navigation
   - **What to Check**:
     - Active connections from ONIX adapters
     - Connection state (running, idle)
     - Channels per connection
     - Message rates per connection

#### 5. **Channels Tab** - Monitor Consumer Channels
   - **Location**: Click "Channels" in the top navigation
   - **What to Check**:
     - Active channels (each consumer uses a channel)
     - Prefetch count (how many unacked messages per consumer)
     - Consumer count per channel
     - Message acknowledgment rates

#### 6. **Publish/Get Messages** - Test Message Publishing
   - **Location**: Go to "Exchanges" → Click on `beckn_exchange` → Scroll to "Publish message"
   - **How to Test**:
     1. Select routing key (e.g., `bpp.discover`, `bpp.select`, etc.)
     2. Enter message payload (JSON format)
     3. Click "Publish message"
     4. Monitor the target queue to see if the message appears
     5. Watch consumer process the message

   **Pre-formatted Test Messages**:
   - Ready-to-use test messages are available in `message/bap/` directory
   - See [Test Messages Documentation](message/bap/README.md) for detailed usage instructions
   - Messages include all BAP APIs: discover (8 variants), select, init, confirm, update, track, cancel, rating, support

   **Example Test Message** (for `bpp.discover`):
   ```json
   {
     "context": {
       "version": "2.0.0",
       "action": "discover",
       "domain": "beckn.one:deg:ev-charging",
       "location": {
         "country": { "code": "IND" },
         "city": { "code": "std:080" }
       },
       "bap_id": "example-bap.com",
       "bap_uri": "http://mock-bap-rabbit-mq:9003",
       "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
       "message_id": "110e8400-e29b-41d4-a716-446655440009",
       "timestamp": "2025-01-27T10:00:00Z",
       "ttl": "PT30S"
     },
     "message": {
       "spatial": [{
         "op": "s_dwithin",
         "targets": "$['beckn:availableAt'][*]['geo']",
         "geometry": {
           "type": "Point",
           "coordinates": [77.59, 12.94]
         },
         "distanceMeters": 10000
       }]
     }
   }
   ```

#### 7. **Get Messages** - Manually Consume Messages
   - **Location**: Go to "Queues" → Click on a queue → Scroll to "Get messages"
   - **How to Use**:
     1. Select acknowledgment mode (Ack mode: `Nack message requeue true/false`)
     2. Set number of messages to retrieve
     3. Click "Get messages"
     4. View message payload and headers
   
   **Note**: This is useful for debugging but doesn't test actual consumer behavior. Use this to inspect messages without consuming them permanently.

### Testing Consumer Behavior - Step by Step

#### Test 1: Verify Consumers are Connected
1. Go to **Queues** → Click `bap_receiver_queue`
2. Check **"Consumers"** column - should show 2 (from ONIX BAP plugin configuration)
3. If 0, check adapter logs: `docker-compose logs onix-bap-plugin-rabbitmq`

#### Test 2: Monitor Message Consumption
1. Publish a test message to `beckn_exchange` with routing key `bap.on_discover`
2. Go to **Queues** → `bap_receiver_queue`
3. Watch **"Ready"** count - should increase briefly
4. Watch **"Unacked"** count - should increase as consumer processes
5. Watch **"Ready"** decrease to 0 as message is consumed
6. Check **"Message rates"** - should show consumption rate

#### Test 3: Test Consumer Prefetch
1. Publish multiple messages (e.g., 10 messages)
2. Monitor **"Unacked"** - should not exceed prefetch count (configured as 10 in adapter)
3. If messages are processed faster than prefetch, **"Unacked"** should stay at prefetch limit
4. As messages are acknowledged, new ones should be delivered

#### Test 4: Test Consumer Failure/Recovery
1. Stop a consumer: `docker-compose stop onix-bap-plugin-rabbitmq`
2. Publish messages - they should accumulate in **"Ready"**
3. Restart consumer: `docker-compose start onix-bap-plugin-rabbitmq`
4. Watch **"Ready"** decrease as consumer resumes processing

#### Test 5: Monitor Message Flow End-to-End
1. **Publish** message to `bpp.discover` routing key
2. Monitor `bpp_receiver_queue` - message should appear
3. Mock BPP should consume and process
4. Mock BPP publishes response to `bap.on_discover`
5. Monitor `bap_receiver_queue` - response should appear
6. ONIX BAP plugin should consume response

### Using RabbitMQ CLI Tools (Alternative)

If you prefer command-line tools:

```bash
# List queues
docker exec rabbitmq rabbitmqctl list_queues

# List exchanges
docker exec rabbitmq rabbitmqctl list_exchanges

# List bindings
docker exec rabbitmq rabbitmqctl list_bindings

# Get queue info
docker exec rabbitmq rabbitmqctl list_queues name messages consumers

# Monitor queue in real-time
watch -n 1 'docker exec rabbitmq rabbitmqctl list_queues name messages consumers'
```

### Troubleshooting Consumer Issues

1. **No Consumers Connected**:
   - Check adapter logs: `docker-compose logs onix-bap-plugin-rabbitmq`
   - Verify RabbitMQ connection in logs
   - Check network connectivity

2. **Messages Not Being Consumed**:
   - Verify consumers are connected (Management UI → Queues → Consumers column)
   - Check if messages are in "Ready" state
   - Verify routing keys match queue bindings
   - Check adapter logs for errors

3. **High Unacked Messages**:
   - Consumers may be processing slowly
   - Check adapter processing time in logs
   - Consider increasing consumer threads in adapter config
   - Check for errors causing message processing to hang

4. **Messages Accumulating**:
   - Consumers may be stopped or crashed
   - Check consumer count in Management UI
   - Restart consumers if needed
   - Verify message format matches expected schema

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

### Changing RabbitMQ Configuration

Edit the adapter configuration files in `docker/monolithic/rabbitmq/config/`:

1. **Change exchange name**: Update `exchange` in consumer/publisher plugin configs
2. **Change queue names**: Update `queue` in consumer plugin configs
3. **Change routing keys**: Update `routingKeys` in consumer plugin configs and routing YAML files

### Modifying ONIX Adapter Routing

To change how requests/responses are routed through the ONIX adapters, edit the routing configuration files:

1. **For BAP routing changes**, edit `docker/monolithic/rabbitmq/config/onix-bap/`:
   - `bapTxnCaller-routing.yaml` - Modify outgoing request routing
   - `bapTxnReciever-routing.yaml` - Modify incoming callback routing

2. **For BPP routing changes**, edit `docker/monolithic/rabbitmq/config/onix-bpp/`:
   - `bppTxnCaller-routing.yaml` - Modify outgoing response routing
   - `bppTxnReciever-routing.yaml` - Modify incoming request routing

3. **Restart the adapter services**:
   ```bash
   docker-compose restart onix-bap-plugin-rabbitmq onix-bpp-plugin-rabbitmq
   ```

**Note**: The routing files use YAML format with `routingRules` sections. Each rule specifies domain, version, target type (url, publisher, bap, or bpp), and endpoints.

### Updating Signing Keys

1. Generate new key pairs for the service
2. Update the service's config file (e.g., adapter.yaml)
3. Update `mock-registry_config.yml` with the new public key
4. Restart affected services

## Troubleshooting

### Service Won't Start

1. **Check ports are available**:
   ```bash
   lsof -i :5672   # RabbitMQ AMQP
   lsof -i :15672  # RabbitMQ Management
   lsof -i :6379   # Redis BAP
   lsof -i :6380   # Redis BPP
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
   # Check ONIX adapter configs
   docker exec onix-bap-plugin-rabbitmq ls -la /app/config/message-baised/rabbit-mq/onix-bap/
   
   # Check mock service configs
   docker exec mock-bap-rabbit-mq ls -la /app/config/
   ```

2. **Check config file syntax**:
   ```bash
   # View adapter configuration
   docker exec onix-bap-plugin-rabbitmq cat /app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml
   
   # View routing configurations
   docker exec onix-bap-plugin-rabbitmq cat /app/config/message-baised/rabbit-mq/onix-bap/bapTxnCaller-routing.yaml
   ```

### RabbitMQ Connection Issues

1. **Verify RabbitMQ is running**:
   ```bash
   curl http://localhost:15672/api/overview
   # Or check management UI at http://localhost:15672
   ```

2. **Check queue bindings**:
   - Open RabbitMQ Management UI: `http://localhost:15672`
   - Navigate to "Exchanges" → `beckn_exchange`
   - Check "Bindings" to see queue bindings

3. **Check queue status**:
   - Navigate to "Queues" in Management UI
   - Verify `bap_receiver_queue` and `bpp_receiver_queue` exist
   - Check message counts and consumer status

### Registry Lookup Failures

1. **Verify registry is running**:
   ```bash
   curl http://localhost:3030/health
   ```

2. **Check subscriber registration**:
   ```bash
   curl http://localhost:3030/lookup?subscriber_id=mock-bap-rabbit-mq
   ```

3. **Verify subscriber URIs are reachable from registry**:
   - Check network connectivity
   - Verify service names match container names

### Message Processing Issues

1. **Check message consumption**:
   - Use RabbitMQ Management UI to monitor queue depths
   - Check if messages are being consumed or stuck

2. **Check adapter logs**:
   ```bash
   docker-compose logs -f onix-bap-plugin-rabbitmq
   docker-compose logs -f onix-bpp-plugin-rabbitmq
   ```

3. **Verify routing keys match**:
   - Ensure routing keys in adapter configs match those used by publishers
   - Check exchange and queue bindings in RabbitMQ Management UI

## File Structure

```
sandbox/docker/monolithic/rabbitmq/
├── docker-compose.yml                 # Unified compose file for all services
├── README.md                          # This file
├── mock-registry_config.yml           # Mock registry configuration
├── mock-cds_config.yml                # Mock CDS configuration
├── mock-bap-rabbitMq_config.yml       # Mock BAP RabbitMQ configuration
├── mock-bpp-rabbitMq_config.yml       # Mock BPP RabbitMQ configuration
└── message/                           # Pre-formatted test messages
    ├── bap/                           # BAP API test messages
    │   ├── README.md                  # Usage instructions for BAP messages
    │   ├── discover-along-a-route.json
    │   ├── discover-by-evse.json
    │   ├── discover-by-cpo.json
    │   ├── discover-by-station.json
    │   ├── discover-within-boundary.json
    │   ├── discover-within-timerange.json
    │   ├── discover-connector-spec.json
    │   ├── discover-vehicle-spec.json
    │   ├── select.json
    │   ├── init.json
    │   ├── confirm.json
    │   ├── update.json
    │   ├── track.json
    │   ├── cancel.json
    │   ├── rating.json
    │   └── support.json
    └── bpp/                           # BPP API test messages (future)

# Docker Compose files (in parent directory - for separate usage)
docker/monolithic/rabbitmq/
├── docker-compose-onix-bap-rabbit-mq-plugin.yml    # BAP plugin compose file
├── docker-compose-onix-bpp-rabbit-mq-plugin.yml     # BPP plugin compose file
└── config/
    ├── onix-bap/
    │   ├── adapter.yaml                 # Main BAP adapter configuration
    │   ├── bapTxnCaller-routing.yaml    # BAP caller routing rules
    │   ├── bapTxnReciever-routing.yaml # BAP receiver routing rules
    │   └── plugin.yaml                  # Plugin-specific configuration
    └── onix-bpp/
        ├── adapter.yaml                 # Main BPP adapter configuration
        ├── bppTxnCaller-routing.yaml    # BPP caller routing rules
        ├── bppTxnReciever-routing.yaml  # BPP receiver routing rules
        └── plugin.yaml                  # Plugin-specific configuration

# Mock services (standalone compose files)
sandbox/mock-bap-rabbitMq/
├── config.yml                         # Mock BAP RabbitMQ configuration
└── docker-compose.mock-bap-rabbitMq.yml

sandbox/mock-bpp-rabbitMq/
├── config.yml                         # Mock BPP RabbitMQ configuration
└── docker-compose.mock-bpp-rabbitMq.yml
```

## Additional Resources

- [ONIX Protocol Documentation](https://github.com/beckn/onix)
- [BAP/BPP Specification](https://github.com/beckn/protocol-specifications)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Main RabbitMQ README](../../../../../docker/monolithic/rabbitmq/README.md) - Detailed ONIX RabbitMQ adapter documentation

## Notes

- The ONIX BAP RabbitMQ adapter exposes HTTP port 8001 for `bapTxnReceiver` endpoint.
- The ONIX BPP RabbitMQ adapter exposes HTTP port 8002 for `bppTxnReceiver` endpoint.
- `bapTxnCaller` and `bppTxnCaller` consume messages from RabbitMQ queues.
- Volume mounts:
  - Entire config directory: `/app/config/message-baised/rabbit-mq/onix-{bap|bpp}` (for routing files and adapter config)
  - Schema directory: `/app/schemas` (read-only, from root `schemas/` directory)
- All mock services use simplified configurations suitable for testing.
- Production deployments should use proper key management and secure configurations.
- Network service discovery uses Docker container names, which must match the service names in configuration files.
- RabbitMQ queues are durable by default, ensuring message persistence across container restarts.
- The adapters use manual acknowledgment (`autoAck: false`) for reliable message processing with retry capability.

