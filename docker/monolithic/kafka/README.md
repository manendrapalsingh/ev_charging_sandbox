# Monolithic Architecture - Kafka Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **monolithic architecture** with **Apache Kafka** event streaming.

## Architecture Overview

In this setup, the onix-adapter uses Apache Kafka for high-throughput event streaming. Services communicate through Kafka topics.

## Components

- **Redis**: Used for caching and state management
- **Apache Kafka**: Event streaming platform
- **Zookeeper**: (if required) For Kafka coordination
- **Onix-Adapter**: Single container handling all BAP/BPP operations via Kafka

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

