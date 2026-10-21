# Getting Started with Event-Driven Microservices

A complete step-by-step guide to understand, run, and extend the microservices architecture.

## Prerequisites

- **Docker & Docker Compose** (recommended - includes all services)
  - Install from: https://www.docker.com/products/docker-desktop
- **Python 3.11+** (for local development without Docker)
- **Git** (for cloning and version control)
- **curl** or Postman (for testing APIs)

## Quick Start (Docker - 5 minutes)

### 1. Start All Services
```bash
cd backend
docker-compose up -d
```

### 2. Verify Services Are Running
```bash
# User Service
curl http://localhost:8001/health

# Billing Service
curl http://localhost:8002/health

# Notification Service
curl http://localhost:8003/health
```

Expected response:
```json
{
  "status": "UP",
  "probe": "liveness"
}
```

### 3. Access API Documentation
- **User Service**: http://localhost:8001/docs
- **Billing Service**: http://localhost:8002/docs
- **Notification Service**: http://localhost:8003/docs

### 4. Test the System

**Create a user:**
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "name": "Test User",
    "password": "TestPassword123!"
  }'
```

**Save the user ID from response**, then:

**Check generated invoice (Billing Service consumed user.created):**
```bash
curl "http://localhost:8002/api/v1/invoices?user_id=<user_id>"
```

**Check notification (Notification Service consumed user.created):**
```bash
curl "http://localhost:8003/api/v1/notifications/users/<user_id>"
```

## Architecture Understanding

### The Event Flow

```
┌─────────────────────────────────────┐
│ User Creates Account (User Service) │
└──────────────┬──────────────────────┘
               │
               ▼
        ┌──────────────┐
        │  Kafka Topic │
        │ user-events  │
        └──────────────┘
               │
        ┌──────┴──────┐
        ▼             ▼
   ┌─────────────┐  ┌─────────────────┐
   │   Billing   │  │  Notification   │
   │  Service    │  │   Service       │
   └─────────────┘  └─────────────────┘
        │                   │
        ▼                   ▼
   Creates Invoice   Queues Email Task
        │                   │
        │                   ▼
        │            ┌──────────────┐
        │            │Celery Worker │
        │            │  (email)     │
        │            └──────────────┘
        │                   │
        ▼                   ▼
   Publishes Event    Send Email
   (payment events)   (via SMTP)
```

### Services Responsibilities

| Service | Port | Listens To | Publishes | Action |
|---------|------|-----------|-----------|---------|
| **User** | 8001 | None | user.* events | User CRUD operations |
| **Billing** | 8002 | user.created | billing.* events | Create invoices, process payments |
| **Notification** | 8003 | user.*, billing.* | notification.* events | Queue notification tasks |
| **Celery Workers** | N/A | Redis queue | None | Send emails/SMS (async) |

## Local Development (Without Docker)

### 1. Setup Python Environment

```bash
# Create virtual environment
python3.11 -m venv venv

# Activate it
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r backend/services/user-service/requirements.txt
```

### 2. Start Infrastructure Services (Docker Only)

You still need Kafka and Redis - start only those:
```bash
cd backend
docker-compose up -d kafka zookeeper redis postgres
```

### 3. Run Services Locally

**Terminal 1 - User Service:**
```bash
cd backend/services/user-service
python -m uvicorn main:app --reload --port 8001
```

**Terminal 2 - Billing Service:**
```bash
cd backend/services/billing-service
python -m uvicorn main:app --reload --port 8002
```

**Terminal 3 - Notification Service API:**
```bash
cd backend/services/notification-service
python -m uvicorn main:app --reload --port 8003
```

**Terminal 4 - Celery Worker (Email):**
```bash
cd backend/services/notification-service
celery -A celery_app worker -l info -Q email
```

**Terminal 5 - Celery Worker (SMS):**
```bash
cd backend/services/notification-service
celery -A celery_app worker -l info -Q sms
```

## Key Concepts Explained

### 1. Event-Driven Architecture

Services communicate through **events** (asynchronous messages) rather than direct API calls:

```python
# Service A publishes an event
event = UserCreatedEvent(
    event_id="evt-123",
    aggregate_id=user_id,
    data={"email": "user@example.com"}
)
await publisher.publish_event(event)

# Service B consumes the event
consumer.register_handler(
    EventType.USER_CREATED,
    handle_user_created  # async function
)
```

**Benefits:**
- Services are loosely coupled
- Can scale independently
- Handle failures gracefully
- Audit trail of all changes

### 2. Event Topics (Kafka)

All events for a domain go to one topic:
- `user-events`: USER_CREATED, USER_UPDATED, USER_DELETED
- `billing-events`: PAYMENT_RECEIVED, INVOICE_CREATED
- `notification-events`: NOTIFICATION_SENT, NOTIFICATION_FAILED

### 3. Async Task Processing (Celery)

For work that takes time (sending emails), queue it as a task:

```python
# Queue task (returns immediately)
send_email.delay(
    notification_id=notif_id,
    recipient="user@example.com",
    subject="Welcome"
)

# Celery worker picks it up and executes
@celery_app.task(bind=True, max_retries=3)
def send_email(self, notification_id, recipient, subject):
    try:
        # Send email
        pass
    except Exception as exc:
        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=60)
```

**Benefits:**
- Requests return immediately
- Tasks retry automatically
- Failures don't crash the API
- Can scale workers independently

### 4. Database per Service

Each service has its own database:
- **User Service**: SQLite (users)
- **Billing Service**: SQLite (invoices, payments)
- **Notification Service**: SQLite (notifications)

**Benefits:**
- Services own their data
- Can use different databases
- Scale databases separately
- Enforces clear boundaries

## Common Tasks

### Add a New Event Type

1. **Define the event** in `backend/shared/events/models.py`:
```python
class MyNewEvent(BaseEvent):
    """Event for new business event."""
    event_type: EventType = Field(default=EventType.MY_NEW_EVENT)
    
    class MyData(BaseModel):
        field1: str
        field2: int
    
    data: MyData = Field(...)
```

2. **Add to enum** in same file:
```python
class EventType(str, Enum):
    MY_NEW_EVENT = "my.new.event"
```

3. **Map to topic** in same file:
```python
def get_topic_for_event(event_type: EventType) -> EventTopic:
    topic_mapping = {
        # ... existing mappings ...
        EventType.MY_NEW_EVENT: EventTopic.MY_EVENTS,
    }
```

4. **Publish from service** where event originates:
```python
from shared.events import MyNewEvent

event = MyNewEvent(
    event_id=str(uuid.uuid4()),
    aggregate_id=resource_id,
    source_service="my-service",
    data=MyNewEvent.MyData(field1="value", field2=42)
)
await publisher.publish_event(event)
```

5. **Consume in other services**:
```python
from shared.events import EventType, MyNewEvent

async def handle_my_new_event(event: MyNewEvent, db: AsyncSession) -> None:
    """Handle event."""
    # Do work based on event
    pass

consumer.register_handler(EventType.MY_NEW_EVENT, handle_my_new_event)
```

### Add a New API Endpoint

1. **Add route** in service's `routes.py`:
```python
@router.post("/resource", response_model=ResponseModel)
async def create_resource(
    data: RequestModel,
    db: AsyncSession,
    publisher: EventPublisher,
) -> ResponseModel:
    """Create resource."""
    # Create in database
    db.add(resource)
    await db.commit()
    
    # Publish event
    await publish_resource_created(publisher, resource)
    
    return ResponseModel(**resource.to_dict())
```

2. **Test in API docs**: http://localhost:PORT/docs

### Add a New Celery Task

1. **Create task** in `notification-service/tasks.py`:
```python
@celery_app.task(bind=True, max_retries=3)
def my_async_task(self, param1, param2):
    try:
        # Do async work
        result = do_something(param1, param2)
        return {"status": "success", "result": result}
    except Exception as exc:
        # Will retry automatically
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

2. **Queue from somewhere**:
```python
from .tasks import my_async_task

my_async_task.delay(param1_value, param2_value)
```

3. **Monitor task** in Celery worker logs:
```bash
celery -A notification_service.celery_app inspect active
```

## Debugging

### Check Kafka Topics
```bash
# List topics
docker exec microservices-kafka kafka-topics --list --bootstrap-server localhost:9092

# Consume messages from a topic
docker exec microservices-kafka kafka-console-consumer --topic user-events --from-beginning --bootstrap-server localhost:9092
```

### Check Redis
```bash
# Connect to Redis CLI
docker exec microservices-redis redis-cli

# View keys
KEYS *

# Get a key
GET <key>

# Check Celery queue
LLEN celery  # View queue length
```

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f user-service

# Celery worker
docker-compose logs -f celery-worker-email
```

### Test Database
```bash
# Connect to PostgreSQL
docker exec -it microservices-postgres psql -U postgres

# List databases
\l

# Connect to database
\c user_service

# List tables
\dt

# Query
SELECT * FROM users;
```

## Troubleshooting

### Services won't start
```bash
# Check Docker status
docker ps

# Check logs
docker-compose logs

# Restart all
docker-compose down
docker-compose up -d
```

### Kafka connectivity issues
```bash
# Test Kafka
docker exec microservices-kafka kafka-broker-api-versions --bootstrap-server localhost:9092

# Reset consumer groups
docker exec microservices-kafka kafka-consumer-groups --reset-offsets --to-earliest --execute --group group-name --topic topic-name
```

### Celery tasks not executing
```bash
# Check if Redis is running
redis-cli ping

# View Celery status
celery -A notification_service.celery_app inspect active

# Check for exceptions
docker-compose logs celery-worker-email
```

### Database migration issues
```bash
# Connect to database
docker exec -it microservices-postgres psql -U postgres

# Drop database (WARNING: deletes data)
DROP DATABASE user_service;

# Restart service to recreate
docker-compose restart user-service
```

## Next Steps

1. **Understand the architecture**: Read [MICROSERVICES_README.md](MICROSERVICES_README.md)
2. **Explore API examples**: Check [API_EXAMPLES.md](API_EXAMPLES.md)
3. **Add a new service**: Follow the patterns in existing services
4. **Add new event types**: Extend the event system for your domain
5. **Deploy to production**: Use Kubernetes or similar orchestration
6. **Monitor in production**: Set up Prometheus, Grafana, and ELK

## Architecture Decision Records (ADRs)

### Why Kafka?
- Distributed event streaming
- Replay capability
- Fault tolerance
- Supports multiple consumers

### Why Celery?
- Async task processing
- Automatic retries
- Task routing
- Distributed workers

### Why Database Per Service?
- Independent scaling
- Clear domain boundaries
- Technology flexibility
- Failure isolation

### Why FastAPI?
- Async by default
- Auto-generated API docs
- Type safety with Pydantic
- High performance

## Performance Tuning

### Database Connection Pooling
```python
DATABASE_POOL_SIZE = 20
DATABASE_MAX_OVERFLOW = 10
DATABASE_POOL_RECYCLE = 3600
```

### Kafka Partitioning
Events are partitioned by `aggregate_id` for ordering guarantees per entity.

### Celery Concurrency
```bash
# 4 concurrent workers
celery -A notification_service.celery_app worker --concurrency=4

# CPU-bound tasks
celery -A notification_service.celery_app worker --pool=prefork --concurrency=4
```

### Redis Configuration
- DB 0: Cache
- DB 1: Celery Broker
- DB 2: Celery Results

## Security Considerations

1. **Environment Variables**: All secrets in `.env` files (never commit)
2. **API Authentication**: Add JWT/OAuth in production
3. **Database Credentials**: Use managed secrets in production
4. **SSL/TLS**: Enable for Kafka and service communication
5. **Rate Limiting**: Add rate limiters per endpoint
6. **Input Validation**: Pydantic validates all inputs

## Resources

- **Kafka Documentation**: https://kafka.apache.org/documentation/
- **FastAPI Docs**: https://fastapi.tiangolo.com/
- **Celery Guide**: https://docs.celeryproject.org/
- **SQLAlchemy ORM**: https://docs.sqlalchemy.org/
- **Pydantic**: https://docs.pydantic.dev/

## Support

For questions or issues:
1. Check service logs: `docker-compose logs <service-name>`
2. Review API docs: http://localhost:PORT/docs
3. Check event schema: `backend/shared/events/models.py`
4. Review example code in services
