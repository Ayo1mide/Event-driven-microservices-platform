# Event-Driven Microservices Architecture

## Executive Summary

This starter kit demonstrates a **production-ready event-driven microservices architecture** with three independent services communicating asynchronously via Kafka. It showcases enterprise design patterns, scalability skills, and best practices for distributed systems.

## Core Architecture Principles

### 1. **Service Independence**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ User Service │     │Billing Service│    │Notification  │
│  (Port 8001) │     │ (Port 8002)   │    │Service API   │
│              │     │              │    │ (Port 8003)  │
│ FastAPI      │     │ FastAPI      │    │ FastAPI      │
│ SQLite       │     │ SQLite       │    │ SQLite       │
└──────────────┘     └──────────────┘    └──────────────┘
```

Each service:
- Owns its database (no shared data stores)
- Has clear, single responsibility
- Exposes REST API with OpenAPI docs
- Communicates via events only (no direct RPC calls)
- Scales independently

### 2. **Event-Driven Communication**

Instead of services calling each other directly:

```
❌ BEFORE (Monolithic/Tightly Coupled)
User Service → (HTTP) → Billing Service → (HTTP) → Notification Service
                       (cascading failures)

✅ AFTER (Event-Driven)
User Service → publish(user.created) → Kafka Topic
                                        ├→ Billing Service (subscribes)
                                        └→ Notification Service (subscribes)
                     (decoupled, resilient)
```

**Benefits:**
- **Decoupling**: Services don't know about each other
- **Resilience**: If one service is down, others continue working
- **Scalability**: Services scale independently based on load
- **Auditability**: Complete event history for compliance
- **Replay**: Can replay events to rebuild state

### 3. **Kafka Topics Structure**

```
Topic: user-events
├── user.created      → Emitted by User Service
├── user.updated      → Emitted by User Service
└── user.deleted      → Emitted by User Service
    (Consumed by: Billing Service, Notification Service)

Topic: billing-events
├── billing.invoice.created      → Emitted by Billing Service
├── billing.payment.received     → Emitted by Billing Service
└── billing.payment.failed       → Emitted by Billing Service
    (Consumed by: Notification Service)

Topic: notification-events
├── notification.sent   → Emitted by Notification Service
└── notification.failed → Emitted by Notification Service
    (For auditing/analytics)
```

## Service Details

### User Service
**Port:** 8001  
**Purpose:** User account management  
**Database:** SQLite (users table)

```python
# Publishes to Kafka
user.created    → {"email": "...", "name": "...", "created_at": "..."}
user.updated    → {"email": "...", "name": "...", "updated_at": "..."}
user.deleted    → {"deleted_at": "..."}

# API
POST   /api/v1/users              # Create user
GET    /api/v1/users/{id}         # Get user
PUT    /api/v1/users/{id}         # Update user
DELETE /api/v1/users/{id}         # Delete user
GET    /api/v1/users              # List users
```

**Pattern Used:**
```
Request → Database Write → Publish Event → Response
```

### Billing Service
**Port:** 8002  
**Purpose:** Invoice and payment management  
**Database:** SQLite (invoices, payments tables)

```python
# Consumes from Kafka
user.created → Automatically create initial invoice

# Publishes to Kafka
billing.invoice.created    → When invoice created
billing.payment.received   → When payment succeeds
billing.payment.failed     → When payment fails

# API
POST   /api/v1/invoices              # Create invoice
GET    /api/v1/invoices/{id}         # Get invoice
POST   /api/v1/payments              # Process payment
GET    /api/v1/payments/{id}         # Get payment
```

**Pattern Used:**
```
Consume Event → Database Write → Publish Event
Process Payment → Database Write → Publish Event
```

### Notification Service
**Port:** 8003  
**Purpose:** Email/SMS delivery and tracking  
**Database:** SQLite (notifications table)  
**Async Layer:** Celery + Redis

```python
# Consumes from Kafka
user.created              → Queue welcome email
billing.payment.received  → Queue payment confirmation
billing.payment.failed    → Queue payment retry reminder

# Publishes to Kafka
notification.sent   → When email/SMS sent successfully
notification.failed → When email/SMS delivery fails

# API
GET /api/v1/notifications/users/{user_id}    # Get user notifications
GET /api/v1/notifications/{id}               # Get notification
GET /api/v1/notifications/stats/summary      # Get statistics
```

**Pattern Used:**
```
Consume Event → Create Notification Record → Queue Celery Task
                                            ↓
                                    Celery Worker (async)
                                    ├→ Send Email
                                    ├→ Retry on failure
                                    └→ Update Status
```

## Data Flow Example: New User Registration

```
1. CREATE USER
   │
   ├→ Request: POST /api/v1/users
   │   {"email": "john@example.com", "name": "John", "password": "..."}
   │
   └─→ User Service
       ├─→ Hash password
       ├─→ Insert into database
       ├─→ Publish Event: user.created
       └─→ Response: 201 Created

2. EVENT PUBLISHED TO KAFKA
   ├─→ Topic: user-events
   │   Event ID: evt-abc123
   │   Type: user.created
   │   Aggregate ID: user-uuid
   │   Data: {email, name, created_at}
   │   Timestamp: 2024-01-15T10:30:45Z

3. SERVICES CONSUME EVENT
   │
   ├─→ Billing Service
   │   ├─→ Consume: user.created
   │   ├─→ Check idempotency (already processed?)
   │   ├─→ Create invoice for new user
   │   ├─→ Insert into invoices table
   │   ├─→ Publish: billing.invoice.created
   │   └─→ Commit offset to Kafka
   │
   └─→ Notification Service
       ├─→ Consume: user.created
       ├─→ Check idempotency (already processed?)
       ├─→ Create notification record
       ├─→ Queue Celery task: send_email
       └─→ Commit offset to Kafka

4. ASYNC EMAIL DELIVERY
   │
   ├─→ Celery Worker picks up task from Redis queue
   │
   ├─→ send_email Task
   │   ├─→ Connect to SMTP
   │   ├─→ Send email with retries
   │   │   (Retry on: timeout, connection error, etc.)
   │   │   (Exponential backoff: 60s, 120s, 240s)
   │   │
   │   ├─→ Update notification status
   │   │   Status: sent
   │   │   Timestamp: 2024-01-15T10:30:47Z
   │   │
   │   └─→ Success logged
   │
   └─→ If failure after 3 retries:
       ├─→ Update notification status: failed
       ├─→ Error reason: "SMTP connection timeout"
       └─→ Alert monitoring system

5. FINAL STATE
   ├─→ User created in user_service.db
   ├─→ Invoice created in billing_service.db
   ├─→ Notification sent by notification_service
   ├─→ Email delivered to user
   └─→ Complete audit trail in event log
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **API Framework** | FastAPI | Modern async Python framework |
| **ORM** | SQLAlchemy | Database abstraction layer |
| **Database** | SQLite/PostgreSQL | Persistent data storage |
| **Event Broker** | Apache Kafka | Asynchronous messaging |
| **Cache/Broker** | Redis | Celery broker & caching |
| **Async Tasks** | Celery | Background job processing |
| **Validation** | Pydantic | Data validation |
| **Logging** | JSON Logging | Structured logging |
| **Containerization** | Docker | Deployment containers |
| **Orchestration** | Docker Compose | Local development |

## Resilience Patterns Implemented

### 1. **Event Sourcing**
All state changes are captured as immutable events:
```
user.created → {"id": "...", "email": "..."}
user.updated → {"id": "...", "name": "..."}
user.deleted → {"id": "..."}
```
Can replay events to rebuild state at any point in time.

### 2. **Idempotent Processing**
Events processed multiple times produce same result:
```python
# Check if already processed
if await is_duplicate(event.event_id):
    return  # Already handled, skip

# Mark as processed
await record_processed_event(event.event_id)
```

### 3. **Offset Management**
Kafka consumers track progress and resume from last position:
```python
# Manual commit ensures at-least-once delivery
consumer.commit()
```

### 4. **Retries with Backoff**
Failed tasks retry with exponential backoff:
```python
@celery_app.task(bind=True, max_retries=3)
def send_email(self, recipient):
    try:
        # send email
    except Exception as exc:
        # Retry after 60s, 120s, 240s
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

### 5. **Graceful Degradation**
Services continue working even if dependencies are down:
```
If Notification Service is down:
  ├─→ User Service: ✓ Still creates users
  ├─→ Billing Service: ✓ Still processes payments
  └─→ Emails: Queued (sent when service recovers)
```

### 6. **Health Checks**
Services expose liveness and readiness probes:
```bash
# Is service running?
GET /health/live → {"status": "UP"}

# Ready to serve traffic?
GET /health/ready → {"status": "UP"}

# Detailed status?
GET /health → {"checks": {"database": "UP", ...}}
```

## Scalability Design

### Horizontal Scaling

**User Service:**
```
Load Balancer
├→ User Service Instance 1 (Port 8001)
├→ User Service Instance 2 (Port 8001)
└→ User Service Instance 3 (Port 8001)
  └→ All read/write to same database
```

**Notification Service:**
```
Kafka Topic: user-events
  ├→ Celery Worker 1 (Email queue)
  ├→ Celery Worker 2 (Email queue)
  ├→ Celery Worker 3 (SMS queue)
  └→ Celery Worker 4 (SMS queue)
    └→ All read from Redis queues
```

### Vertical Scaling

- **Increase workers**: `--concurrency=8`
- **Increase connection pool**: `DATABASE_POOL_SIZE=50`
- **Increase Celery prefetch**: `worker_prefetch_multiplier=10`

## Performance Characteristics

| Metric | Value | Notes |
|--------|-------|-------|
| **API Response Time** | <100ms | Sync database operations |
| **Event Processing Latency** | <1s | Kafka consumer poll interval |
| **Email Delivery** | <5s | Async via Celery |
| **Database Throughput** | 1000+ ops/sec | With connection pooling |
| **Kafka Throughput** | 1M+ msgs/sec | Cluster dependent |

## Security Architecture

### Authentication/Authorization
```
FastAPI Dependency Injection
├→ Validate request
├→ Check JWT/OAuth token (add in production)
├→ Extract user context
└→ Pass to route handler
```

### Data Isolation
```
Each Service
├→ Own database
├→ Own database user/password
└→ Own API credentials
```

### Secrets Management
```
Environment Variables (.env file)
├→ DATABASE_PASSWORD
├→ KAFKA_CREDENTIALS
├→ REDIS_PASSWORD
├→ SMTP_PASSWORD
└→ (Never committed to git)
```

## Deployment Architecture

### Local Development
```
Docker Compose
├─ PostgreSQL Container
├─ Kafka Container
├─ Zookeeper Container
├─ Redis Container
├─ User Service Container
├─ Billing Service Container
├─ Notification Service Container
└─ Celery Workers Container
```

### Production (Kubernetes)
```
Kubernetes Cluster
├─ Namespace: microservices
│
├─ Services (Deployments)
│  ├─ user-service (3 replicas)
│  ├─ billing-service (3 replicas)
│  └─ notification-service (3 replicas)
│
├─ StatefulSets
│  ├─ Kafka (3 brokers)
│  └─ PostgreSQL (1 primary, 2 replicas)
│
├─ ConfigMaps
│  └─ Service configurations
│
├─ Secrets
│  └─ Database passwords, API keys
│
└─ Ingress
   └─ API Gateway (Kong, Nginx Ingress)
```

## Monitoring & Observability

### Metrics
- **Prometheus** scrapes service metrics
- **Prometheus Operator** in Kubernetes
- Key metrics:
  - Request latency (p50, p95, p99)
  - Error rates
  - Database connection pool usage
  - Kafka consumer lag
  - Celery task duration

### Logging
```
Service → Structured JSON Logs → ELK Stack
├─ Elasticsearch (storage)
├─ Logstash (processing)
└─ Kibana (visualization)

Key log fields:
├─ timestamp
├─ level (ERROR, WARNING, INFO)
├─ logger (service name)
├─ message
├─ event_id (for tracing)
└─ context (user_id, request_id)
```

### Tracing
```
Distributed Tracing
├─ Jaeger
├─ Trace events through all services
├─ Identify bottlenecks
└─ Visualize request flow
```

### Alerting
```
Alert Manager
├─ Service is down
├─ High error rate
├─ High latency
├─ Database issues
├─ Kafka consumer lag
└─ Celery queue overflow
```

## Cost Optimization

### Database
- Connection pooling reduces resource usage
- Periodic cleanup of old records
- Proper indexing on frequently queried columns

### Kafka
- Partition topics by entity for balanced load
- Retention policies for old events
- Compression for storage savings

### Celery
- Batching tasks when possible
- Auto-scaling workers based on queue depth
- Dead letter queue for failed tasks

## Testing Strategy

### Unit Tests
```python
# Test individual functions
def test_user_creation():
    user = create_user("john@example.com", "John")
    assert user.email == "john@example.com"
```

### Integration Tests
```python
# Test service interactions
async def test_user_creation_creates_invoice():
    user = await create_user()
    # User Service publishes event
    # Billing Service consumes event
    # Verify invoice was created
```

### End-to-End Tests
```python
# Test complete user journey
async def test_complete_registration_flow():
    # 1. Create user
    # 2. Verify invoice created
    # 3. Verify email queued
    # 4. Verify email sent
```

## Summary of Enterprise Patterns

✅ **Event Sourcing** - Complete audit trail  
✅ **CQRS** - Separate read/write models (extensible)  
✅ **Saga Pattern** - Distributed transactions  
✅ **Idempotency** - Safe retries  
✅ **Circuit Breaker** - Prevent cascading failures (extensible)  
✅ **Health Checks** - Service readiness  
✅ **Structured Logging** - Observability  
✅ **Configuration Management** - Environment-based settings  
✅ **Database per Service** - Loose coupling  
✅ **Async Processing** - Scalability  
✅ **Offset Management** - Exactly-once delivery (at-least-once impl.)  
✅ **API Versioning** - Future compatibility  

## Getting Started

1. **Read**: [GETTING_STARTED.md](GETTING_STARTED.md) - Step-by-step guide
2. **Run**: `cd backend && docker-compose up -d`
3. **Explore**: http://localhost:8001/docs (API docs)
4. **Test**: See [API_EXAMPLES.md](API_EXAMPLES.md)
5. **Learn**: Read code comments and documentation
6. **Extend**: Add new services following existing patterns

## Key Takeaways

This starter kit demonstrates:
- **Microservices architecture** with clear boundaries
- **Event-driven communication** for loose coupling
- **Async processing** for scalability
- **Enterprise patterns** for resilience
- **Production-ready setup** with Docker
- **Complete documentation** for learning

Perfect for:
- Learning microservices design
- Building distributed systems
- Understanding event-driven architecture
- Implementing scalable applications
- Teaching distributed system concepts
