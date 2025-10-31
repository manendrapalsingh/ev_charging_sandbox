# Monolithic Architecture - RabbitMQ Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **monolithic architecture** with **RabbitMQ** message broker communication.

## Architecture Overview

In this setup, the onix-adapter uses RabbitMQ for asynchronous message-based communication. Services communicate through message queues instead of direct HTTP calls.

## Components

- **Redis**: Used for caching and state management
- **RabbitMQ**: Message broker for asynchronous communication
- **Onix-Adapter**: Single container handling all BAP/BPP operations via RabbitMQ

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

