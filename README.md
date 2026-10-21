# Event-Driven Microservices Platform

Author: Ahmed Ayo Eremosele  
Maintainer: Ahmed Ayo Eremosele

A production-style microservices platform demonstrating event-driven backend architecture, asynchronous processing, service-to-service communication, and scalable system design.

## Overview

This project is built to showcase how multiple independent services can work together using event-driven patterns. It includes a user service, billing service, and notification service coordinated by Kafka-based messaging and asynchronous task processing.

The system is designed to demonstrate strong backend engineering practices such as:

- service separation and bounded contexts
- asynchronous workflows
- event-driven integration
- idempotent processing patterns
- Docker-based local orchestration
- API-driven system communication

## Tech Stack

- Python
- FastAPI
- SQLAlchemy
- PostgreSQL
- Redis
- Celery
- Apache Kafka
- Docker
- Docker Compose

## Architecture

```text
User Service        Billing Service        Notification Service
     |                    |                        |
     |---- user.created --> Kafka ---------------------->
     |                                                  |
     |                                                  |-- async email task
     |                                                  |
     |<---- invoice / notification state updates --------|
```

## Key Features

- Independent microservices with separate responsibilities
- Event-driven communication using Kafka
- Asynchronous background tasks with Celery
- Redis for task queueing and caching
- PostgreSQL-backed persistence
- Health checks and service observability
- Dockerized local infrastructure
- API documentation via FastAPI

## Services

### User Service
Handles user registration and management.

### Billing Service
Handles invoice creation and payment processing logic.

### Notification Service
Consumes events and triggers asynchronous notifications.

## Getting Started

```bash
cd backend
docker-compose up -d
```

Verify services:

```bash
curl http://localhost:8001/health
curl http://localhost:8002/health
curl http://localhost:8003/health
```

## Documentation

The project includes detailed local docs and architecture references for setup, system design, commands, and API examples.

## Contact

- GitHub: [@Ayo1mide](https://github.com/Ayo1mide)
- Email: ayojoshua11411@gmail.com
- Portfolio: https://www.userkoolai.com/

## License

MIT
