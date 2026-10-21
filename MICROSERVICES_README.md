# Event-Driven Microservices Starter Kit

A production-ready template demonstrating event-driven microservices architecture with Python, FastAPI, Kafka, PostgreSQL, Redis, and Celery. This template showcases enterprise design patterns and scalable architecture skills.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway / Load Balancer                  │
└─────────────────────────────────────────────────────────────────────┘
           │                        │                        │
           ▼                        ▼                        ▼
    ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
    │ User Service │        │Billing Service│     │Notification   │
    │  (Port 8001) │        │  (Port 8002)  │     │Service API    │
    │              │        │              │     │ (Port 8003)    │
    │  FastAPI     │        │  FastAPI     │     │  FastAPI       │
    │  SQLite      │        │  SQLite      │     │  SQLite        │
    └──────────────┘        └──────────────┘     └──────────────┘
           │                        │                   │
           └────────────┬───────────┴───────────┬──────┘
                        │                       │
                        ▼                       ▼
                   ┌──────────────┐      ┌──────────────┐
                   │    Kafka     │      │    Redis     │
                   │   (Broker)   │      │   (Cache &   │
                   │              │      │   Celery)    │
                   │ Topics:      │      └──────────────┘
                   │ • user-events│
                   │ • billing-ev.│
                   │ • notif-ev.  │
                   └──────────────┘
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
      ┌──────────────┐         ┌──────────────┐
      │Celery Worker │         │Celery Worker │
      │(Email Queue) │         │ (SMS Queue)  │
      └──────────────┘         └──────────────┘
```

## Services

### 1. User Service (Port 8001)
- **Responsibility**: User registration, authentication, profile management
- **Events Produced**: `user.created`, `user.updated`, `user.deleted`
- **Technology**: FastAPI, SQLite, Pydantic
- **Endpoints**:
  - `POST /api/v1/users` - Create user
  - `GET /api/v1/users/{user_id}` - Get user
  - `PUT /api/v1/users/{user_id}` - Update user
  - `DELETE /api/v1/users/{user_id}` - Delete user

### 2. Billing Service (Port 8002)
- **Responsibility**: Invoice generation, payment processing, subscription management
- **Events Consumed**: `user.created`
- **Events Produced**: `billing.payment.received`, `billing.payment.failed`, `billing.invoice.created`
- **Technology**: FastAPI, SQLite
- **Endpoints**:
  - `POST /api/v1/invoices` - Create invoice
  - `GET /api/v1/invoices/{invoice_id}` - Get invoice
  - `POST /api/v1/payments` - Process payment
  - `GET /api/v1/payments/{payment_id}` - Get payment

### 3. Notification Service (Port 8003)
- **Responsibility**: Email/SMS notifications based on user and billing events
- **Events Consumed**: `user.created`, `billing.payment.received`, `billing.payment.failed`
- **Events Produced**: `notification.sent`, `notification.failed`
- **Technology**: FastAPI, Celery, Redis, SQLite
- **Async Processing**: Celery workers for email/SMS delivery
- **Endpoints**:
  - `GET /api/v1/notifications/users/{user_id}` - Get user notifications
  - `GET /api/v1/notifications/{notification_id}` - Get notification
  - `GET /api/v1/notifications/stats/summary` - Get statistics

## Infrastructure Components

### Kafka
- **Event Broker** for asynchronous communication between services
- **Topics**:
  - `user-events`: User lifecycle events
  - `billing-events`: Billing and payment events
  - `notification-events`: Notification status events
- **Offset Management**: Auto-commit with idempotency checking

### PostgreSQL
- Primary database for production deployment
- Service-specific databases (can be shared or separate)
- Connection pooling with SQLAlchemy

### Redis
- **Celery Broker** (DB 1) for task queue
- **Celery Result Backend** (DB 2) for task results
- **Cache Layer** (DB 0) for application caching

### Celery
- Async task processing with Redis broker
- Task routing by queue (email, sms, push)
- Automatic retries with exponential backoff
- Task monitoring and dead letter queue handling

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Python 3.11+ (for local development)
- pip, virtualenv

### Running with Docker Compose

```bash
# Clone and navigate to project
cd backend

# Start all services
docker-compose up -d

# Check service health
curl http://localhost:8001/health  # User Service
curl http://localhost:8002/health  # Billing Service
curl http://localhost:8003/health  # Notification Service

# View logs
docker-compose logs -f user-service
docker-compose logs -f billing-service
docker-compose logs -f notification-service
docker-compose logs -f celery-worker-email
```

### Services URLs
- **User Service API**: http://localhost:8001/docs
- **Billing Service API**: http://localhost:8002/docs
- **Notification Service API**: http://localhost:8003/docs
- **Kafka**: localhost:9092
- **Redis**: localhost:6379
- **PostgreSQL**: localhost:5432

### Running Locally (Without Docker)

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r backend/services/user-service/requirements.txt

# Start services
cd backend/services/user-service
python -m uvicorn main:app --reload --port 8001

# In another terminal, start Celery worker
cd backend/services/notification-service
celery -A celery_app worker -l info -Q email
```

## API Examples

### Create a User
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "name": "John Doe",
    "password": "securepass123"
  }'
```

### Get User
```bash
curl http://localhost:8001/api/v1/users/{user_id}
```

### Create Invoice
```bash
curl -X POST http://localhost:8002/api/v1/invoices \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-123",
    "amount": 99.99,
    "currency": "USD",
    "due_date": "2024-02-28",
    "description": "Monthly subscription"
  }'
```

### Process Payment
```bash
curl -X POST http://localhost:8002/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "invoice_id": "invoice-123",
    "user_id": "user-123",
    "amount": 99.99
  }'
```

### Get User Notifications
```bash
curl http://localhost:8003/api/v1/notifications/users/{user_id}
```

## Event Flow Example

### Scenario: New User Registration

1. **User Creation**
   - Client: `POST /api/v1/users`
   - User Service creates user in database
   - Publishes `user.created` event to Kafka

2. **Event Processing**
   - **Billing Service** consumes `user.created`
     - Creates initial invoice for user
     - Publishes `billing.invoice.created`
   - **Notification Service** consumes `user.created`
     - Creates email notification record
     - Queues `send_email` task to Celery

3. **Async Processing**
   - Celery Worker picks up `send_email` task
   - Sends welcome email (with retries on failure)
   - Updates notification status in database

4. **Event Chain Result**
   - User registered
   - Initial invoice generated
   - Welcome email sent
   - All logged and tracked

## Key Features & Patterns

### 1. Event-Driven Architecture
- Services communicate asynchronously via Kafka
- Event sourcing for audit trail
- Eventual consistency model

### 2. Async Task Processing
- Celery for background jobs with reliability
- Task retries with exponential backoff
- Dead letter queue for failed tasks
- Task routing by queue type

### 3. Database per Service
- Each service has its own database
- Enforces loose coupling
- Enables independent scaling
- Example: Use SQLite for dev, PostgreSQL for production

### 4. Error Handling & Resilience
- Custom exception hierarchy
- Graceful degradation
- Circuit breaker pattern ready
- Idempotent event processing
- Offset management in Kafka

### 5. Configuration Management
- Environment-based configuration
- Settings inheritance
- Service-specific overrides

### 6. Health Checks
- Liveness probe (`/health/live`)
- Readiness probe (`/health/ready`)
- Dependency checks (database, Redis)

### 7. Structured Logging
- JSON formatted logs
- Contextual information
- Service tracing support

### 8. API Standards
- FastAPI auto-generated OpenAPI docs
- Consistent error responses
- API versioning (/api/v1/)

## Project Structure

```
backend/
├── services/
│   ├── user-service/
│   │   ├── main.py                 # FastAPI app
│   │   ├── models.py               # SQLAlchemy models
│   │   ├── routes.py               # API routes
│   │   ├── events.py               # Event publishing
│   │   ├── config.py               # Service config
│   │   └── requirements.txt
│   ├── billing-service/
│   │   ├── main.py
│   │   ├── models.py               # Invoice, Payment models
│   │   ├── routes.py
│   │   ├── events.py               # Event consuming & publishing
│   │   ├── config.py
│   │   └── requirements.txt
│   └── notification-service/
│       ├── main.py
│       ├── models.py               # Notification model
│       ├── celery_app.py           # Celery configuration
│       ├── tasks.py                # Celery tasks
│       ├── routes.py               # API routes
│       ├── events.py               # Event consuming
│       ├── config.py
│       └── requirements.txt
├── shared/
│   ├── __init__.py
│   ├── config.py                   # Base settings
│   ├── events/
│   │   ├── models.py               # Event schemas
│   │   ├── publisher.py            # Kafka producer
│   │   └── consumer.py             # Kafka consumer base
│   ├── models/
│   │   └── __init__.py             # SQLAlchemy base
│   └── utils/
│       ├── exceptions.py           # Custom exceptions
│       ├── logging.py              # Structured logging
│       ├── health.py               # Health checks
│       └── __init__.py
├── docker-compose.yml              # Full infrastructure stack
└── migrations/                      # Alembic migrations (future)
```

## Development Workflow

### 1. Adding a New Service

```python
# 1. Create service directory
mkdir -p backend/services/new-service

# 2. Create main.py with FastAPI app
# 3. Create models.py with database schemas
# 4. Create routes.py with API endpoints
# 5. Create events.py for event handling
# 6. Create config.py for settings
# 7. Create requirements.txt

# 8. Add to docker-compose.yml
# 9. Test service endpoints
```

### 2. Adding Event Communication

```python
# In event-producing service:
from shared.events import EventPublisher, UserCreatedEvent

await publisher.publish_event(event)

# In event-consuming service:
from shared.events import EventConsumer, EventType

consumer = EventConsumer(bootstrap_servers, group_id, topics)
consumer.register_handler(EventType.USER_CREATED, handle_user_created)
await consumer.start()
```

### 3. Adding Async Tasks

```python
# Define Celery task
@celery_app.task(bind=True, max_retries=3)
def my_task(self, param1, param2):
    try:
        # Do work
        pass
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)

# Queue task
my_task.delay(param1_value, param2_value)

# Monitor tasks
celery -A notification_service.celery_app events
```

## Production Deployment

### Kubernetes Configuration
Services can be deployed to Kubernetes with:
- Separate pods per service
- StatefulSet for Kafka/Redis
- ConfigMaps for configuration
- PersistentVolumes for databases
- Horizontal Pod Autoscaling

### Load Balancing
- API Gateway (Kong, Nginx, AWS ALB) routes to services
- Each service scales independently
- Health checks guide traffic

### Monitoring
- Prometheus for metrics
- Grafana for visualization
- ELK Stack for logs
- Sentry for error tracking

### Database Migration
For PostgreSQL in production:
```bash
alembic init alembic
alembic revision --autogenerate -m "Initial migration"
alembic upgrade head
```

## Performance Considerations

1. **Connection Pooling**: SQLAlchemy pool_size=20, max_overflow=10
2. **Kafka Partitioning**: By aggregate_id for ordering guarantees
3. **Celery Prefetch**: worker_prefetch_multiplier=4 for balanced load
4. **Redis TTL**: Cache entries expire automatically (3600s)
5. **Task Time Limit**: Hard (30min) and soft (25min) limits prevent hangs

## Security Best Practices

1. **Password Hashing**: SHA256 (upgrade to bcrypt/argon2 in production)
2. **Environment Variables**: All secrets in .env files
3. **CORS**: Configure per service if needed
4. **API Versioning**: /api/v1/ allows future upgrades
5. **Input Validation**: Pydantic validation on all inputs
6. **Database Connection**: Use connection pooling with timeouts

## Troubleshooting

### Service won't start
```bash
# Check logs
docker-compose logs service-name

# Verify Kafka is ready
docker-compose logs kafka | grep "Cluster ID"

# Test Kafka connectivity
docker exec microservices-kafka kafka-topics --list --bootstrap-server localhost:9092
```

### Messages not being consumed
```bash
# Check consumer groups
docker exec microservices-kafka kafka-consumer-groups --list --bootstrap-server localhost:9092

# Reset offset
docker exec microservices-kafka kafka-consumer-groups --reset-offsets --to-earliest --execute --group group-name --topic topic-name
```

### Celery tasks not running
```bash
# Ensure Redis is running
redis-cli ping

# Check Celery worker status
celery -A notification_service.celery_app inspect active

# View pending tasks
celery -A notification_service.celery_app inspect reserved
```

## Contributing

1. Follow existing patterns in the codebase
2. Add tests for new features
3. Update documentation
4. Ensure all services start with Docker Compose
5. Create migration scripts for database changes

## License

MIT
