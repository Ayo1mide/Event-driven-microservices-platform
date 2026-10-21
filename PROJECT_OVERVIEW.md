# Event-Driven Microservices Starter Kit - Project Overview

## 📋 What's Included

A complete, production-ready microservices template with three independent services communicating via Kafka, demonstrating enterprise architecture patterns and best practices.

## 📁 Project Structure

```
backend/
├── services/
│   ├── user-service/                    # User management service
│   │   ├── __init__.py
│   │   ├── main.py                      # FastAPI application
│   │   ├── models.py                    # SQLAlchemy models
│   │   ├── routes.py                    # API endpoints
│   │   ├── events.py                    # Event publishing
│   │   ├── config.py                    # Service configuration
│   │   └── requirements.txt             # Python dependencies
│   │
│   ├── billing-service/                 # Billing & payment service
│   │   ├── __init__.py
│   │   ├── main.py                      # FastAPI application
│   │   ├── models.py                    # Invoice/Payment models
│   │   ├── routes.py                    # API endpoints
│   │   ├── events.py                    # Event consuming & publishing
│   │   ├── config.py                    # Service configuration
│   │   └── requirements.txt             # Python dependencies
│   │
│   └── notification-service/            # Notification & email service
│       ├── __init__.py
│       ├── main.py                      # FastAPI application
│       ├── models.py                    # Notification model
│       ├── routes.py                    # API endpoints
│       ├── tasks.py                     # Celery async tasks
│       ├── celery_app.py                # Celery configuration
│       ├── events.py                    # Event consuming
│       ├── config.py                    # Service configuration
│       └── requirements.txt             # Python dependencies
│
├── shared/                              # Shared utilities & base classes
│   ├── __init__.py
│   ├── config.py                        # Base configuration
│   ├── events/
│   │   ├── __init__.py
│   │   ├── models.py                    # Event schemas & definitions
│   │   ├── publisher.py                 # Kafka event publisher
│   │   └── consumer.py                  # Kafka event consumer base
│   ├── models/
│   │   └── __init__.py                  # SQLAlchemy base classes
│   └── utils/
│       ├── __init__.py
│       ├── exceptions.py                # Custom exception classes
│       ├── logging.py                   # Structured logging setup
│       └── health.py                    # Health check endpoints
│
└── docker-compose.yml                   # Complete infrastructure stack

docker/
├── Dockerfile.service                   # Docker image for services
└── (entrypoint scripts)

Documentation/
├── MICROSERVICES_README.md              # Complete guide & reference
├── GETTING_STARTED.md                   # Step-by-step setup guide
├── ARCHITECTURE.md                      # Architecture deep dive
├── API_EXAMPLES.md                      # Request/response examples
└── PROJECT_OVERVIEW.md                  # This file

Configuration/
├── .env.example                         # Environment template
└── docker-compose.yml                   # Docker orchestration
```

## 🚀 Quick Start

### With Docker (Recommended)
```bash
cd backend
docker-compose up -d

# Check services
curl http://localhost:8001/health  # User Service
curl http://localhost:8002/health  # Billing Service
curl http://localhost:8003/health  # Notification Service
```

### Without Docker
```bash
# Each service in separate terminal
python -m uvicorn main:app --reload --port 8001

# Celery worker
celery -A notification_service.celery_app worker -l info
```

### Services & Ports
- **User Service** (8001): http://localhost:8001/docs
- **Billing Service** (8002): http://localhost:8002/docs
- **Notification Service** (8003): http://localhost:8003/docs
- **Kafka**: localhost:9092
- **Redis**: localhost:6379
- **PostgreSQL**: localhost:5432

## 📚 Documentation Guide

| Document | Purpose | Audience |
|----------|---------|----------|
| [GETTING_STARTED.md](GETTING_STARTED.md) | Setup & quick walkthrough | Everyone starting out |
| [MICROSERVICES_README.md](MICROSERVICES_README.md) | Complete reference manual | Developers building on it |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Design & patterns explained | Architects & senior devs |
| [API_EXAMPLES.md](API_EXAMPLES.md) | Request/response examples | API users & testers |

## 🏗️ Architecture Highlights

### Three Microservices

**1. User Service (Port 8001)**
- User registration & management
- Publishes: `user.created`, `user.updated`, `user.deleted`
- Tech: FastAPI, SQLite
- API: `/api/v1/users`

**2. Billing Service (Port 8002)**
- Invoice generation & payment processing
- Consumes: `user.created`
- Publishes: `billing.invoice.created`, `billing.payment.received/failed`
- Tech: FastAPI, SQLite
- API: `/api/v1/invoices`, `/api/v1/payments`

**3. Notification Service (Port 8003)**
- Email/SMS delivery via Celery
- Consumes: `user.created`, `billing.payment.*`
- Tech: FastAPI, Celery, Redis, SQLite
- API: `/api/v1/notifications`
- Workers: 2 email workers, optional SMS workers

### Infrastructure (Docker Compose)

```
Docker Compose provides:
├─ PostgreSQL 15         (Data storage)
├─ Apache Kafka          (Event broker)
├─ Zookeeper            (Kafka coordinator)
├─ Redis 7              (Cache & Celery broker)
├─ User Service         (FastAPI, port 8001)
├─ Billing Service      (FastAPI, port 8002)
├─ Notification Service (FastAPI, port 8003)
├─ Celery Worker Email  (async tasks)
├─ Celery Worker SMS    (async tasks)
└─ Celery Beat          (scheduled tasks)
```

## 🔑 Key Features

### Event-Driven Architecture
- ✅ Kafka for asynchronous messaging
- ✅ Event sourcing for audit trail
- ✅ Topic-based communication
- ✅ Consumer groups for scaling

### Resilience Patterns
- ✅ Idempotent event processing
- ✅ Graceful failure handling
- ✅ Automatic retries with backoff
- ✅ Health checks (liveness & readiness)
- ✅ Offset management

### Database Design
- ✅ Database per service (loose coupling)
- ✅ SQLAlchemy ORM
- ✅ Connection pooling
- ✅ Timestamp tracking

### Async Processing
- ✅ Celery for background jobs
- ✅ Redis as task broker
- ✅ Task routing by queue
- ✅ Automatic retries

### API Standards
- ✅ FastAPI with auto-generated docs
- ✅ API versioning (/api/v1/)
- ✅ Consistent error responses
- ✅ Request validation with Pydantic

### Observability
- ✅ Structured JSON logging
- ✅ Service health checks
- ✅ Error tracking
- ✅ Event tracing

## 🎓 Learning Outcomes

By studying this codebase, you'll understand:

1. **Microservices Design**
   - Service boundaries and responsibilities
   - Communication patterns
   - Data management strategies

2. **Event-Driven Architecture**
   - Event sourcing and event streams
   - Consumer patterns
   - Eventual consistency

3. **Distributed Systems**
   - Handling failures gracefully
   - Scaling independently
   - Maintaining consistency

4. **Python Web Development**
   - FastAPI for modern APIs
   - SQLAlchemy ORM patterns
   - Async/await programming

5. **Enterprise Patterns**
   - Saga pattern for transactions
   - Idempotent processing
   - Health checks and monitoring

6. **DevOps & Containerization**
   - Docker images and containers
   - Docker Compose orchestration
   - Service networking

## 🛠️ Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Language** | Python | 3.11+ |
| **Web Framework** | FastAPI | 0.104+ |
| **Server** | Uvicorn | 0.24+ |
| **ORM** | SQLAlchemy | 2.0+ |
| **Validation** | Pydantic | 2.5+ |
| **Async Jobs** | Celery | 5.3+ |
| **Cache/Broker** | Redis | 7+ |
| **Event Streaming** | Kafka | 7.5+ |
| **Database** | SQLite/PostgreSQL | Latest |
| **Containerization** | Docker | Latest |
| **Orchestration** | Docker Compose | Latest |
| **Logging** | python-json-logger | 2.0+ |

## 📊 Service Dependencies

```
User Service
├─ FastAPI
├─ SQLite
├─ Kafka (publisher)
└─ Pydantic

Billing Service
├─ FastAPI
├─ SQLite
├─ Kafka (publisher + consumer)
└─ Pydantic

Notification Service
├─ FastAPI
├─ SQLite
├─ Kafka (consumer)
├─ Celery (task queue)
└─ Redis (broker)

Infrastructure
├─ PostgreSQL (shared DB option)
├─ Kafka + Zookeeper
├─ Redis
└─ Docker
```

## 🔄 Event Flow Example

```
1. User created
   POST /api/v1/users
   ↓
2. User Service publishes event
   Topic: user-events
   Event: user.created
   ↓
3. Services consume event
   Billing Service → Creates invoice → Publishes billing.invoice.created
   Notification Service → Creates notification → Queues email task
   ↓
4. Celery worker processes
   Picks up email from Redis queue
   Sends via SMTP
   Updates notification status
   ↓
5. Complete flow logged
   Event audit trail maintained
   All state changes recorded
```

## 🧪 Testing This System

### Manual Testing
```bash
# Create user (triggers entire event chain)
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "name": "Test User",
    "password": "TestPassword123!"
  }'

# Check if invoice was created (in Billing Service)
curl "http://localhost:8002/api/v1/invoices?user_id=<user_id>"

# Check if notification was queued (in Notification Service)
curl "http://localhost:8003/api/v1/notifications/users/<user_id>"
```

### Kafka Testing
```bash
# Watch events being published
docker exec microservices-kafka kafka-console-consumer \
  --topic user-events \
  --from-beginning \
  --bootstrap-server localhost:9092
```

### Redis Testing
```bash
# Check Celery queue
docker exec microservices-redis redis-cli
> LLEN celery  # Queue length
> KEYS *       # All keys
```

## 📈 Scalability Features

### Horizontal Scaling
- Add more service instances behind load balancer
- Kafka partitions distribute events across consumers
- Celery workers process tasks in parallel

### Vertical Scaling
- Increase database connection pool size
- Increase Celery worker concurrency
- Adjust Kafka consumer batch size

### Monitoring Ready
- Health check endpoints for orchestrators
- Structured logging for log analysis
- Prometheus metrics support (extensible)

## 🔒 Security Considerations

1. **Secrets Management**
   - Environment variables in `.env` files
   - Never commit secrets to version control
   - Use managed secrets service in production

2. **API Security**
   - Add JWT/OAuth for authentication (extensible)
   - Validate all inputs with Pydantic
   - Use HTTPS in production

3. **Database Security**
   - Use strong passwords
   - Enable SSL connections
   - Implement RLS in production

## 🚢 Production Deployment

### Kubernetes
All services can be deployed to Kubernetes:
- Separate deployments for each service
- StatefulSets for Kafka/databases
- ConfigMaps for configuration
- Secrets for sensitive data
- Ingress for API gateway

### Cloud Platforms
- AWS ECS/Fargate
- Google Cloud Run
- Azure Container Instances
- DigitalOcean App Platform

## 📖 Recommended Reading Order

1. **First time?** → [GETTING_STARTED.md](GETTING_STARTED.md)
2. **Understand architecture** → [ARCHITECTURE.md](ARCHITECTURE.md)
3. **Reference guide** → [MICROSERVICES_README.md](MICROSERVICES_README.md)
4. **Test APIs** → [API_EXAMPLES.md](API_EXAMPLES.md)
5. **Study code** → Review service implementations

## 🆘 Common Tasks

### Add a new event type
See "Add a New Event Type" in [MICROSERVICES_README.md](MICROSERVICES_README.md)

### Add a new API endpoint
See "Add a New API Endpoint" in [GETTING_STARTED.md](GETTING_STARTED.md)

### Add a new Celery task
See "Add a New Celery Task" in [GETTING_STARTED.md](GETTING_STARTED.md)

### Deploy to production
See "Production Deployment" in [MICROSERVICES_README.md](MICROSERVICES_README.md)

## 📝 Code Conventions

### Naming
- Snake case: `user_service`, `create_invoice`
- Class names: PascalCase: `UserService`, `InvoiceModel`
- Constants: UPPER_CASE: `MAX_RETRIES`, `DEFAULT_TIMEOUT`

### Structure
- Models in `models.py`
- Routes in `routes.py`
- Events in `events.py`
- Configuration in `config.py`
- Tasks in `tasks.py` (Celery)

### Imports
- Grouped by: stdlib, third-party, local
- One import per line
- Alphabetically sorted

### Documentation
- Docstrings on all functions
- Type hints on all parameters
- Comments for non-obvious logic

## 🤝 Contributing

To extend this starter:
1. Follow existing patterns
2. Add tests for new features
3. Update documentation
4. Test with Docker Compose
5. Verify all services start

## 📄 License

MIT - Feel free to use and modify for your projects

## ✨ Key Takeaways

This starter kit demonstrates:
- ✅ **Enterprise-grade architecture** with proven patterns
- ✅ **Production-ready code** that's actually deployable
- ✅ **Learning tool** for understanding distributed systems
- ✅ **Extensible foundation** for building real applications
- ✅ **Best practices** in Python, microservices, and DevOps

Perfect for developers who want to:
- Learn microservices architecture
- Understand event-driven systems
- Build scalable applications
- Implement production patterns
- Teach distributed systems

## 🎯 Next Steps

1. ✅ Run `docker-compose up` and see it work
2. ✅ Read the architecture overview
3. ✅ Try the API examples
4. ✅ Study the service code
5. ✅ Add a new event type
6. ✅ Deploy to your platform

Enjoy building!
