# Docker BAP RabbitMQ Plugin

This directory contains the Docker plugin for ONIX BAP (Buyer App Platform) adapter with RabbitMQ message queue integration.

## Overview

The BAP RabbitMQ plugin is designed to handle BAP-side transactions using RabbitMQ for asynchronous message processing. The adapter consists of two modules:

1. **bapTxnCaller**: Queue consumer that consumes requests from BAP Backend and routes them to BPP via HTTP

2. **bapTxnReceiver**: HTTP handler that receives callbacks from BPP and publishes them to BAP Backend via RabbitMQ

### Architecture

#### bapTxnCaller (Queue Consumer)

- **Queue-Based Consumption**: Consumes messages from `bap_caller_queue` bound to routing keys like `bpp.discover`, `bpp.select`, etc. (requests from BAP Backend)

- **Message Processing**: Messages are processed through configured steps (`validateSchema`, `addRoute`, `sign`)

- **HTTP Routing**: Routes processed messages to BPP adapter via HTTP using `targetType: bpp`

- **ACK/NACK Handling**: 

  - **ACK**: Sent when message processing succeeds

  - **NACK**: Sent with requeue when processing fails (allows retry)

- **Manual Acknowledgment**: Uses `autoAck: false` for reliable message processing

#### bapTxnReceiver (HTTP Handler)

- **HTTP Endpoint**: Receives HTTP requests at `/bap/receiver/` from BPP adapter

- **Message Processing**: Processes through steps (`validateSign`, `validateSchema`, `addRoute`)

- **Publishing**: Publishes processed messages to RabbitMQ with routing keys like `bap.on_discover`, `bap.on_select`, etc. (callbacks to BAP Backend)

## Features

- **Dual-Mode Operation**: 

  - Queue consumer for requests from BAP Backend

  - HTTP handler for callbacks from BPP adapter

- **Queue-Based Message Consumption**: Consumes requests from BAP Backend via RabbitMQ

- **HTTP-Based Callback Reception**: Receives callbacks from BPP adapter via HTTP

- **Message Publishing**: Publishes callbacks to BAP Backend via RabbitMQ

- **Manual ACK/NACK**: Reliable message processing with retry capability

- **Phase 1 Support**: Routes discover requests to CDS for aggregation

- **Phase 2+ Support**: Routes requests directly to BPP and receives callbacks

- **External Configuration**: Config files are mounted from the host

- **Plugin Support**: All required plugins are built and included

## Pre-built Images

Pre-built images are available from Docker Hub:

- `manendrapalsingh/onix-bap-plugin-rabbit-mq:latest`

- `manendrapalsingh/onix-bap-plugin-rabbit-mq:sha-{commit-sha}`

## Building the Image

### Build from Source

```bash
# From repository root
docker build -f docker_plugin/rabbit-mq/docker-bap-rabbitmq-plugin/Dockerfile -t onix-bap-plugin-rabbit-mq:latest .
```

## Running with Docker Compose

### Using Pre-built Image

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"   # AMQP port
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
      RABBITMQ_USERNAME: admin
      RABBITMQ_PASSWORD: admin
    networks:
      - onix-network
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-bap:
    image: redis:alpine
    container_name: redis-bap
    ports:
      - "6379:6379"
    networks:
      - onix-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  onix-bap-plugin-rabbitmq:
    image: manendrapalsingh/onix-bap-plugin-rabbit-mq:latest
    container_name: onix-bap-plugin-rabbitmq
    ports:
      - "8001:8001"  # HTTP port for bapTxnReceiver endpoint
    volumes:
      - ../config/message-baised/rabbit-mq/onix-bap:/app/config/message-baised/rabbit-mq/onix-bap
      - ../schema:/app/schemas:ro
    environment:
      - CONFIG_FILE=/app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml
      - RABBITMQ_USERNAME=admin
      - RABBITMQ_PASSWORD=admin
    depends_on:
      rabbitmq:
        condition: service_healthy
      redis-bap:
        condition: service_healthy
    networks:
      - onix-network
    restart: unless-stopped

networks:
  onix-network:
    driver: bridge
```

Run with:

```bash
docker-compose up -d
```

### Building Locally

If you need to build locally, update the service definition:

```yaml
  onix-bap-plugin-rabbitmq:
    build:
      context: ../..
      dockerfile: docker_plugin/rabbit-mq/docker-bap-rabbitmq-plugin/Dockerfile
    # ... rest of the configuration
```

## Running with Docker Run

```bash
docker run -d \
  --name onix-bap-plugin-rabbitmq \
  -v $(pwd)/config/message-baised/rabbit-mq/onix-bap:/app/config/message-baised/rabbit-mq/onix-bap \
  -v $(pwd)/schema:/app/schemas:ro \
  -e CONFIG_FILE=/app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml \
  -e RABBITMQ_USERNAME=admin \
  -e RABBITMQ_PASSWORD=admin \
  --network onix-network \
  manendrapalsingh/onix-bap-plugin-rabbit-mq:latest
```

## Configuration

### Required Services

1. **RabbitMQ**: Message queue server (default: `rabbitmq:5672`)

2. **Redis**: Cache server (default: `redis-bap:6379`)

3. **Registry**: Mock registry service (default: `http://mock-registry:3030`)

### Environment Variables

- `CONFIG_FILE`: Path to the adapter configuration file (default: `/app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml`)

- `RABBITMQ_USERNAME`: RabbitMQ username (required)

- `RABBITMQ_PASSWORD`: RabbitMQ password (required)

### RabbitMQ Configuration

#### bapTxnCaller (Queue Consumer)

The adapter consumes requests from BAP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)

- **Queue**: `bap_caller_queue` (durable)

- **Routing Keys** (requests from BAP Backend): 

  - `bpp.discover` - Discover request

  - `bpp.select`, `bpp.init`, `bpp.confirm`, `bpp.status`, `bpp.track`, `bpp.cancel`, `bpp.update`, `bpp.rating`, `bpp.support` - Other requests

- **Acknowledgment**: Manual (`autoAck: false`)

- **Prefetch Count**: 10

- **Consumer Threads**: 2

#### bapTxnReceiver (HTTP Handler + Publisher)

The adapter publishes callbacks to BAP Backend with the following configuration:

- **Exchange**: `beckn_exchange` (topic exchange, durable)

- **HTTP Endpoint**: `/bap/receiver/` (receives HTTP requests from BPP adapter)

- **Publishing Routing Keys** (callbacks to BAP Backend):

  - `bap.on_discover` - Phase 1 aggregated search results

  - `bap.on_select`, `bap.on_init`, `bap.on_confirm`, `bap.on_status`, `bap.on_track`, `bap.on_cancel`, `bap.on_update`, `bap.on_rating`, `bap.on_support` - Phase 2+ callbacks

#### Queue Configuration Options

All queue parameters are configurable in `adapter.yaml`:

```yaml
rabbitmqConsumer:
  config:
    durable: "true"        # Queue survives broker restarts (default: true)
    autoDelete: "false"    # Don't delete queue when unused (default: false)
    exclusive: "false"     # Queue can be used by multiple connections (default: false)
    noWait: "false"        # Wait for queue creation confirmation (default: false)
    queueArgs: ""          # Optional queue arguments (see below)
```

#### Queue Arguments (queueArgs)

The `queueArgs` parameter allows you to configure advanced RabbitMQ queue options. It accepts either JSON format or key=value pairs:

**JSON Format Example:**

```yaml
queueArgs: '{"x-message-ttl":3600000,"x-max-length":10000,"x-max-priority":10}'
```

**Key-Value Format Example:**

```yaml
queueArgs: "x-message-ttl=3600000,x-max-length=10000,x-dead-letter-exchange=dlx"
```

**Available Queue Arguments:**

| Argument | Type | Description | Example |
|----------|------|-------------|---------|
| `x-message-ttl` | int64 | Message time-to-live in milliseconds | `3600000` (1 hour) |
| `x-expires` | int64 | Queue expiration in milliseconds | `7200000` (2 hours) |
| `x-max-length` | int | Maximum number of messages in queue | `10000` |
| `x-max-priority` | int | Maximum priority level (0-255) | `10` |
| `x-dead-letter-exchange` | string | Dead letter exchange name | `"dlx"` |
| `x-dead-letter-routing-key` | string | Dead letter routing key | `"failed"` |
| `x-queue-type` | string | Queue type: `"classic"` or `"quorum"` | `"quorum"` |
| `x-overflow` | string | Behavior when full: `"drop-head"` or `"reject-publish"` | `"reject-publish"` |
| `x-single-active-consumer` | bool | Only one consumer processes messages | `true` |
| `x-queue-mode` | string | Queue mode: `"default"` or `"lazy"` | `"lazy"` |

**Example Configuration with Queue Arguments:**

```yaml
rabbitmqConsumer:
  config:
    queueName: "bap_receiver_queue"
    durable: "true"
    autoDelete: "false"
    exclusive: "false"
    noWait: "false"
    # Set message TTL to 1 hour, max queue length to 10k, enable priority
    queueArgs: '{"x-message-ttl":3600000,"x-max-length":10000,"x-max-priority":10,"x-dead-letter-exchange":"dlx"}'
```

### Processing Steps

#### bapTxnCaller (Queue Consumer)

Processes requests from BAP Backend through:

- `validateSchema` - Validates message schema

- `addRoute` - Determines routing destination (BPP or CDS)

- `sign` - Signs the message before forwarding

After processing:

- **Success**: Routes to BPP/CDS via HTTP, sends ACK → message removed from queue

- **Error**: Sends NACK with requeue → message retried

#### bapTxnReceiver (HTTP Handler)

Processes callbacks from BPP adapter through:

- `validateSign` - Validates message signatures

- `validateSchema` - Validates message schema

- `addRoute` - Determines publishing routing key

After processing:

- **Success**: Publishes to RabbitMQ with appropriate routing key

- **Error**: Returns HTTP error response

## Producer and Consumer Examples

### Message Flow

#### Flow 1: BAP Backend → BPP Backend (Requests)

1. **BAP Backend**: Publishes requests to `beckn_exchange` with routing keys like `bpp.discover`, `bpp.select`, etc.

2. **bapTxnCaller**: Consumes from `bap_caller_queue`, processes, routes to BPP adapter via HTTP

3. **BPP Adapter**: Receives HTTP request, processes, publishes to BPP Backend

4. **BPP Backend**: Consumes from `bpp_receiver_queue`

#### Flow 2: BPP Backend → BAP Backend (Callbacks)

1. **BPP Backend**: Publishes callbacks to `beckn_exchange` with routing keys like `bpp.on_discover`, `bpp.on_select`, etc.

2. **BPP Adapter**: Consumes from `bpp_caller_queue`, routes to BAP adapter via HTTP

3. **bapTxnReceiver**: Receives HTTP request at `/bap/receiver/`, processes, publishes to RabbitMQ

4. **BAP Backend**: Consumes callbacks from queues bound to routing keys like `bap.on_discover`, `bap.on_select`, etc.

### Producer Examples (BAP Backend Publishing Requests)

#### Node.js Producer

```javascript
const amqp = require('amqplib');

async function publishMessage() {
  const connection = await amqp.connect('amqp://admin:admin@localhost:5672');
  const channel = await connection.createChannel();
  
  const exchange = 'beckn_exchange';
  await channel.assertExchange(exchange, 'topic', { durable: true });
  
  const message = {
    context: {
      domain: "ev_charging_network",
      action: "discover",
      bap_id: "example-bap.com",
      bpp_id: "example-bpp.com",
      transaction_id: "123e4567-e89b-12d3-a456-426614174000",
      message_id: "msg-123",
      timestamp: new Date().toISOString()
    },
    message: {
      // Your message payload
    }
  };
  
  const routingKey = 'bpp.discover';  // Request routing key
  const messageBuffer = Buffer.from(JSON.stringify(message));
  
  channel.publish(exchange, routingKey, messageBuffer, {
    persistent: true,
    contentType: 'application/json'
  });
  
  console.log(`Published message to ${routingKey}`);
  
  setTimeout(() => {
    connection.close();
  }, 500);
}

publishMessage().catch(console.error);
```

#### Go Producer

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "time"
    
    "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp091.Dial("amqp://admin:admin@localhost:5672/")
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()
    
    ch, err := conn.Channel()
    if err != nil {
        log.Fatalf("Failed to open channel: %v", err)
    }
    defer ch.Close()
    
    exchange := "beckn_exchange"
    err = ch.ExchangeDeclare(
        exchange,
        "topic",
        true,  // durable
        false, // auto-delete
        false, // internal
        false, // no-wait
        nil,
    )
    if err != nil {
        log.Fatalf("Failed to declare exchange: %v", err)
    }
    
    message := map[string]interface{}{
        "context": map[string]interface{}{
            "domain": "ev_charging_network",
            "action": "discover",
            "bap_id": "example-bap.com",
            "bpp_id": "example-bpp.com",
            "transaction_id": "123e4567-e89b-12d3-a456-426614174000",
            "message_id": "msg-123",
            "timestamp": time.Now().Format(time.RFC3339),
        },
        "message": map[string]interface{}{
            // Your message payload
        },
    }
    
    body, _ := json.Marshal(message)
    routingKey := "bpp.discover"  // Request routing key
    
    err = ch.PublishWithContext(
        context.Background(),
        exchange,
        routingKey,
        false, // mandatory
        false, // immediate
        amqp091.Publishing{
            ContentType:  "application/json",
            DeliveryMode: amqp091.Persistent,
            Body:         body,
        },
    )
    if err != nil {
        log.Fatalf("Failed to publish: %v", err)
    }
    
    log.Printf("Published message to %s", routingKey)
}
```

#### Python Producer

```python
import pika
import json
from datetime import datetime

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        port=5672,
        credentials=pika.PlainCredentials('admin', 'admin')
    )
)
channel = connection.channel()

exchange = 'beckn_exchange'
channel.exchange_declare(exchange=exchange, exchange_type='topic', durable=True)

message = {
    "context": {
        "domain": "ev_charging_network",
        "action": "discover",
        "bap_id": "example-bap.com",
        "bpp_id": "example-bpp.com",
        "transaction_id": "123e4567-e89b-12d3-a456-426614174000",
        "message_id": "msg-123",
        "timestamp": datetime.now().isoformat()
    },
    "message": {
        # Your message payload
    }
}

routing_key = 'bpp.discover'  # Request routing key
channel.basic_publish(
    exchange=exchange,
    routing_key=routing_key,
    body=json.dumps(message),
    properties=pika.BasicProperties(
        delivery_mode=2,  # Make message persistent
        content_type='application/json'
    )
)

print(f"Published message to {routing_key}")
connection.close()
```

#### Java Producer

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

public class Producer {
    private static final String EXCHANGE = "beckn_exchange";
    private static final String ROUTING_KEY = "bpp.discover";  // Request routing key
    
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            
            channel.exchangeDeclare(EXCHANGE, "topic", true);
            
            Map<String, Object> message = new HashMap<>();
            Map<String, Object> context = new HashMap<>();
            context.put("domain", "ev_charging_network");
            context.put("action", "discover");
            context.put("bap_id", "example-bap.com");
            context.put("bpp_id", "example-bpp.com");
            context.put("transaction_id", "123e4567-e89b-12d3-a456-426614174000");
            context.put("message_id", "msg-123");
            context.put("timestamp", Instant.now().toString());
            
            message.put("context", context);
            message.put("message", new HashMap<>()); // Your payload
            
            ObjectMapper mapper = new ObjectMapper();
            String messageJson = mapper.writeValueAsString(message);
            
            channel.basicPublish(
                EXCHANGE,
                ROUTING_KEY,
                new AMQP.BasicProperties.Builder()
                    .contentType("application/json")
                    .deliveryMode(2) // Persistent
                    .build(),
                messageJson.getBytes()
            );
            
            System.out.println("Published message to " + ROUTING_KEY);
        }
    }
}
```

### Consumer Examples (BAP Backend Consuming Callbacks)

The adapter publishes callbacks to `beckn_exchange` with routing keys. Your BAP Backend should consume these.

#### Node.js Consumer

```javascript
const amqp = require('amqplib');

async function consumeMessages() {
  const connection = await amqp.connect('amqp://admin:admin@localhost:5672');
  const channel = await connection.createChannel();
  
  const exchange = 'beckn_exchange';
  await channel.assertExchange(exchange, 'topic', { durable: true });
  
  // Consume callbacks from the adapter
  // The adapter publishes to routing keys like:
  // bap.on_discover, bap.on_select, etc.
  const routingKey = 'bap.on_discover';
  const queueName = `bap_${routingKey.replace(/\./g, '_')}_queue`;
  
  const queue = await channel.assertQueue(queueName, { durable: true });
  await channel.bindQueue(queue.queue, exchange, routingKey);
  
  // Set prefetch to process one message at a time
  channel.prefetch(1);
  
  console.log(`Waiting for messages on ${routingKey}. To exit press CTRL+C`);
  
  channel.consume(queue.queue, (msg) => {
    if (msg) {
      try {
        const content = JSON.parse(msg.content.toString());
        console.log('Received callback:', content);
        
        // Process your callback here
        // ...
        
        // Acknowledge after successful processing
        channel.ack(msg);
      } catch (error) {
        console.error('Error processing message:', error);
        // Reject and requeue on error
        channel.nack(msg, false, true);
      }
    }
  }, {
    noAck: false // Manual acknowledgment
  });
}

consumeMessages().catch(console.error);
```

#### Go Consumer

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "time"
    
    "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp091.Dial("amqp://admin:admin@localhost:5672/")
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()
    
    ch, err := conn.Channel()
    if err != nil {
        log.Fatalf("Failed to open channel: %v", err)
    }
    defer ch.Close()
    
    exchange := "beckn_exchange"
    routingKey := "bap.on_discover"  // Callback routing key
    
    err = ch.ExchangeDeclare(
        exchange,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        log.Fatalf("Failed to declare exchange: %v", err)
    }
    
    queue, err := ch.QueueDeclare(
        "bap_on_discover_queue", // queue name for callbacks
        true,  // durable
        false, // delete when unused
        false, // exclusive
        false, // no-wait
        nil,
    )
    if err != nil {
        log.Fatalf("Failed to declare queue: %v", err)
    }
    
    err = ch.QueueBind(
        queue.Name,
        routingKey,
        exchange,
        false,
        nil,
    )
    if err != nil {
        log.Fatalf("Failed to bind queue: %v", err)
    }
    
    err = ch.Qos(1, 0, false) // Process one message at a time
    if err != nil {
        log.Fatalf("Failed to set QoS: %v", err)
    }
    
    msgs, err := ch.Consume(
        queue.Name,
        "",
        false, // auto-ack
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        log.Fatalf("Failed to register consumer: %v", err)
    }
    
    log.Printf("Waiting for messages on %s. Press CTRL+C to exit", routingKey)
    
    for msg := range msgs {
        var message map[string]interface{}
        if err := json.Unmarshal(msg.Body, &message); err != nil {
            log.Printf("Error parsing message: %v", err)
            msg.Nack(false, true) // Requeue on error
            continue
        }
        
        log.Printf("Received message: %+v", message)
        
        // Process your message here
        // ...
        
        // Acknowledge after successful processing
        msg.Ack(false)
    }
}
```

#### Python Consumer

```python
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        port=5672,
        credentials=pika.PlainCredentials('admin', 'admin')
    )
)
channel = connection.channel()

exchange = 'beckn_exchange'
routing_key = 'bap.on_discover'  # Callback routing key

channel.exchange_declare(exchange=exchange, exchange_type='topic', durable=True)

queue_name = 'bap_on_discover_queue'  # Queue for callbacks
channel.queue_declare(queue=queue_name, durable=True)
channel.queue_bind(exchange=exchange, queue=queue_name, routing_key=routing_key)

# Set prefetch to process one message at a time
channel.basic_qos(prefetch_count=1)

print(f'Waiting for messages on {routing_key}. To exit press CTRL+C')

def callback(ch, method, properties, body):
    try:
        message = json.loads(body)
        print(f"Received message: {message}")
        
        # Process your message here
        # ...
        
        # Acknowledge after successful processing
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        print(f"Error processing message: {e}")
        # Reject and requeue on error
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

channel.basic_consume(
    queue=queue_name,
    on_message_callback=callback,
    auto_ack=False  # Manual acknowledgment
)

channel.start_consuming()
```

#### Java Consumer

```java
import com.rabbitmq.client.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer {
    private static final String EXCHANGE = "beckn_exchange";
    private static final String ROUTING_KEY = "bap.on_discover";  // Callback routing key
    private static final String QUEUE_NAME = "bap_on_discover_queue";  // Queue for callbacks
    
    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        
        channel.exchangeDeclare(EXCHANGE, "topic", true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        channel.queueBind(QUEUE_NAME, EXCHANGE, ROUTING_KEY);
        
        // Process one message at a time
        channel.basicQos(1);
        
        System.out.println("Waiting for messages on " + ROUTING_KEY + ". Press CTRL+C to exit");
        
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            try {
                String message = new String(delivery.getBody(), "UTF-8");
                System.out.println("Received message: " + message);
                
                // Process your message here
                // ...
                
                // Acknowledge after successful processing
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            } catch (Exception e) {
                System.err.println("Error processing message: " + e);
                // Reject and requeue on error
                channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
            }
        };
        
        channel.basicConsume(QUEUE_NAME, false, deliverCallback, consumerTag -> {});
    }
}
```

## Integration with Existing System

### Adding to Existing Docker Compose

```yaml
services:
  # ... your existing services ...

  onix-bap-plugin-rabbitmq:
    image: manendrapalsingh/onix-bap-plugin-rabbit-mq:latest
    container_name: onix-bap-plugin-rabbitmq
    volumes:
      - ./config/message-baised/rabbit-mq/onix-bap:/app/config/message-baised/rabbit-mq/onix-bap
      - ./schema:/app/schemas:ro
    environment:
      - CONFIG_FILE=/app/config/message-baised/rabbit-mq/onix-bap/adapter.yaml
      - RABBITMQ_USERNAME=admin
      - RABBITMQ_PASSWORD=admin
    networks:
      - your-existing-network
    depends_on:
      - rabbitmq
      - redis-bap
    restart: unless-stopped
```

## Troubleshooting

### RabbitMQ Connection Issues

- Verify RabbitMQ is running: `docker ps | grep rabbitmq`

- Check RabbitMQ logs: `docker logs rabbitmq`

- Verify network connectivity: Ensure plugin container can reach `rabbitmq:5672`

- Check credentials: Ensure `RABBITMQ_USERNAME` and `RABBITMQ_PASSWORD` environment variables are set

### Messages Not Consumed

- Verify queue exists: Access RabbitMQ Management UI at `http://localhost:15672`

- Check queue bindings: Ensure queue is bound to exchange with correct routing key

- Review adapter logs: `docker logs onix-bap-plugin-rabbitmq`

- Verify routing keys match: 

  - For requests: Check `routingKeys` in `bapTxnCaller` config match your BAP Backend producer (e.g., `bpp.discover`)

  - For callbacks: Check that BAP Backend consumes from queues bound to callback routing keys (e.g., `bap.on_discover`)

### ACK/NACK Issues

- Check adapter logs for processing errors

- Verify message format matches expected schema

- Check signature validation if `validateSign` step is enabled

- Messages with errors will be NACKed and requeued automatically

### Config Not Found

- Verify config directory is mounted correctly

- Check that `adapter.yaml` exists at the mounted path

- Verify file permissions

## Related Documentation

- [Main Docker Plugin README](../../README.md)

- [Configuration Guide](../../../../CONFIG.md)

- [Setup Guide](../../../../SETUP.md)

