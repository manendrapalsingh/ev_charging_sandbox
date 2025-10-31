# Microservice Architecture - API Integration

This guide demonstrates how to integrate the **onix-adapter** with BAP and BPP applications using **Docker containers** in a **microservice architecture** with **REST API** communication.

## Architecture Overview

In a microservice architecture, the onix-adapter components are split into separate containers/services. Each component can be scaled independently.

## Components

- **Redis**: Shared caching service
- **Onix-Adapter Caller Service**: Handles outgoing requests
- **Onix-Adapter Receiver Service**: Handles incoming requests
- **API Gateway** (optional): For routing and load balancing

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

