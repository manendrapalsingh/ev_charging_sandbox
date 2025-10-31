# Microservice Architecture - Kafka Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **microservice architecture** with **Apache Kafka** event streaming.

## Architecture Overview

Microservice architecture with Apache Kafka for event streaming between services.

## Components

- **Redis**: Shared caching service
- **Apache Kafka**: Event streaming platform
- **Zookeeper**: (if required) For Kafka coordination
- **Onix-Adapter Caller Service**: Separate service for outgoing events
- **Onix-Adapter Receiver Service**: Separate service for incoming events

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

