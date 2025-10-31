# Microservice Architecture - RabbitMQ Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **microservice architecture** with **RabbitMQ** message broker communication.

## Architecture Overview

Microservice architecture with RabbitMQ for message-based communication between services.

## Components

- **Redis**: Shared caching service
- **RabbitMQ**: Message broker
- **Onix-Adapter Caller Service**: Separate service for outgoing messages
- **Onix-Adapter Receiver Service**: Separate service for incoming messages

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Access to onix-adapter Docker images

## Configuration

*(Configuration files coming soon)*

## Quick Start

```bash
# Start services
docker-compose up -d

# Check logs
docker-compose logs -f
```

## Documentation

Full documentation and example configurations will be added here.

