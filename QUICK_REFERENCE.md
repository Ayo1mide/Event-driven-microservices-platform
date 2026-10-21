# Microservices Quick Reference

## 🚀 Start Everything (One Command)
```bash
cd backend
docker-compose up -d
```

## 📍 Service URLs
| Service | URL | Docs |
|---------|-----|------|
| User | http://localhost:8001 | /docs |
| Billing | http://localhost:8002 | /docs |
| Notification | http://localhost:8003 | /docs |
| Kafka | localhost:9092 | - |
| Redis | localhost:6379 | - |
| PostgreSQL | localhost:5432 | - |

## 🧪 Test Everything

### 1. Create User (triggers event chain)
```bash
RESPONSE=$(curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "name": "Test User",
    "password": "TestPassword123!"
  }')

USER_ID=$(echo $RESPONSE | jq -r '.id')
echo "Created user: $USER_ID"
```

### 2. Check Invoice Created (Billing Service consumed event)
```bash
curl "http://localhost:8002/api/v1/invoices?user_id=$USER_ID"
```

### 3. Check Notification Queued (Notification Service consumed event)
```bash
curl "http://localhost:8003/api/v1/notifications/users/$USER_ID"
```

### 4. Process Payment
```bash
# Get invoice ID first
INVOICE_ID=$(curl -s "http://localhost:8002/api/v1/invoices?user_id=$USER_ID" | jq -r '.[0].id')

# Process payment
curl -X POST http://localhost:8002/api/v1/payments \
  -H "Content-Type: application/json" \
  -d "{
    \"invoice_id\": \"$INVOICE_ID\",
    \"user_id\": \"$USER_ID\",
    \"amount\": 99.99
  }"
```

## 📊 Watch Events in Real-Time

### Kafka Events
```bash
docker exec microservices-kafka kafka-console-consumer \
  --topic user-events \
  --from-beginning \
  --bootstrap-server localhost:9092
```

### Celery Tasks
```bash
docker exec microservices-redis redis-cli
> LLEN celery          # Queue length
> RPOP celery          # View next task
> KEYS *               # All keys
```

### Service Logs
```bash
# All logs
docker-compose logs -f

# Specific service
docker-compose logs -f user-service
docker-compose logs -f celery-worker-email
```

## 🏗️ Project Structure (TL;DR)
```
backend/
├── services/
│   ├── user-service/          (Port 8001)
│   ├── billing-service/       (Port 8002)
│   └── notification-service/  (Port 8003)
├── shared/                     (Events, exceptions, logging)
└── docker-compose.yml         (All infrastructure)
```

## 📚 Key Files

| File | Purpose |
|------|---------|
| `backend/shared/events/models.py` | Event schemas |
| `backend/shared/events/publisher.py` | Kafka producer |
| `backend/shared/events/consumer.py` | Kafka consumer |
| `services/user-service/routes.py` | User API |
| `services/billing-service/routes.py` | Billing API |
| `services/notification-service/tasks.py` | Celery tasks |

## 🔧 Common Commands

### Start/Stop Services
```bash
# Start all
docker-compose up -d

# Stop all
docker-compose down

# Restart specific service
docker-compose restart user-service
```

### View Logs
```bash
# Follow logs
docker-compose logs -f

# Last 100 lines
docker-compose logs --tail=100

# Specific service
docker-compose logs -f notification-service
```

### Execute Commands
```bash
# Run command in service container
docker exec user-service ls -la

# Bash shell in service
docker exec -it user-service bash

# Redis CLI
docker exec -it microservices-redis redis-cli

# PostgreSQL
docker exec -it microservices-postgres psql -U postgres
```

### Database
```bash
# Connect to PostgreSQL
docker exec -it microservices-postgres psql -U postgres

# List tables
\dt

# Query
SELECT * FROM users;

# Drop database
DROP DATABASE user_service;
```

## 🧩 Event Types

| Event | Topic | Published By | Consumed By |
|-------|-------|--------------|-------------|
| `user.created` | user-events | User Service | Billing, Notification |
| `user.updated` | user-events | User Service | Billing, Notification |
| `user.deleted` | user-events | User Service | Billing, Notification |
| `billing.invoice.created` | billing-events | Billing Service | Notification |
| `billing.payment.received` | billing-events | Billing Service | Notification |
| `billing.payment.failed` | billing-events | Billing Service | Notification |
| `notification.sent` | notification-events | Notification | (audit) |
| `notification.failed` | notification-events | Notification | (audit) |

## 🚨 Troubleshooting

### Service won't start
```bash
# Check logs
docker-compose logs service-name

# Restart
docker-compose restart service-name

# Rebuild
docker-compose up -d --build
```

### Can't connect to Kafka
```bash
# Check Kafka is running
docker-compose logs kafka

# Test connection
docker exec microservices-kafka kafka-broker-api-versions \
  --bootstrap-server localhost:9092
```

### Redis not responding
```bash
# Check Redis
docker-compose logs redis

# Test Redis
docker exec microservices-redis redis-cli ping
# Should return: PONG
```

### Database errors
```bash
# Check PostgreSQL
docker-compose logs postgres

# Connect and check
docker exec -it microservices-postgres psql -U postgres -l
```

## 📈 Performance Testing

### Load Test User Service
```bash
# Install Apache Bench
# brew install httpd  (macOS)

# Run test
ab -n 1000 -c 10 http://localhost:8001/api/v1/users
```

### Monitor Kafka Throughput
```bash
docker exec microservices-kafka kafka-consumer-perf-test \
  --broker-list localhost:9092 \
  --topic user-events \
  --messages 10000
```

## 🔍 Debugging

### Check Service Health
```bash
# Liveness
curl http://localhost:8001/health/live

# Readiness
curl http://localhost:8001/health/ready

# Full health
curl http://localhost:8001/health
```

### Check Celery Status
```bash
docker exec celery-worker-email celery -A services.notification_service.celery_app inspect active
```

### Check Kafka Consumer Groups
```bash
docker exec microservices-kafka kafka-consumer-groups \
  --list \
  --bootstrap-server localhost:9092
```

## 💾 Data Cleanup

### Reset Database
```bash
# Stop services
docker-compose down

# Remove volumes (deletes all data)
docker-compose down -v

# Start fresh
docker-compose up -d
```

### Reset Kafka Offsets
```bash
docker exec microservices-kafka kafka-consumer-groups \
  --reset-offsets \
  --to-earliest \
  --execute \
  --group notification-service-group \
  --topic user-events
```

## 📝 API Quick Tests

### Create User (via curl)
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","name":"User","password":"Pass123!"}'
```

### Get User
```bash
curl http://localhost:8001/api/v1/users/{user_id}
```

### Create Invoice
```bash
curl -X POST http://localhost:8002/api/v1/invoices \
  -H "Content-Type: application/json" \
  -d '{"user_id":"...","amount":99.99,"due_date":"2024-02-15"}'
```

### Get Notifications
```bash
curl http://localhost:8003/api/v1/notifications/users/{user_id}
```

## 🎯 Key Concepts

| Concept | Explanation |
|---------|-------------|
| **Event** | Immutable message about something that happened |
| **Topic** | Named feed of events (like a message channel) |
| **Producer** | Service that publishes events |
| **Consumer** | Service that subscribes to events |
| **Consumer Group** | Named group of consumers (each gets message once) |
| **Offset** | Position in the topic (like a bookmark) |
| **Task** | Unit of work queued in Celery |
| **Worker** | Process that executes Celery tasks |
| **Broker** | Redis/message queue that holds tasks |
| **Aggregate** | Entity identified by aggregate_id |

## 🎓 Learning Path

1. **Start**: `docker-compose up -d`
2. **Explore**: Open http://localhost:8001/docs
3. **Test**: Run user creation curl command above
4. **Watch**: `docker-compose logs -f`
5. **Read**: [GETTING_STARTED.md](GETTING_STARTED.md)
6. **Understand**: [ARCHITECTURE.md](ARCHITECTURE.md)
7. **Extend**: Add new events or services

## 📞 Quick Help

| Question | Answer |
|----------|--------|
| Where is User Service code? | `backend/services/user-service/` |
| How do I add a new event? | See Event Schemas in `shared/events/models.py` |
| How do I test an API? | Use http://localhost:PORT/docs (Swagger UI) |
| How do services communicate? | Through Kafka events |
| How are emails sent? | Celery workers process async tasks |
| Where is my data? | Each service has its own database |
| How do I scale? | Run multiple service instances + Celery workers |
| What if a service crashes? | Events are in Kafka, can replay when back up |

## 🆘 Emergency Commands

```bash
# Stop everything (safe)
docker-compose down

# Restart everything
docker-compose restart

# Full reset (dangerous - loses data)
docker-compose down -v && docker-compose up -d

# View all running containers
docker ps

# Remove all stopped containers
docker container prune

# Clean up Docker system
docker system prune
```

---

**For detailed information, see:**
- 📖 [MICROSERVICES_README.md](MICROSERVICES_README.md) - Complete reference
- 🚀 [GETTING_STARTED.md](GETTING_STARTED.md) - Setup guide
- 🏗️ [ARCHITECTURE.md](ARCHITECTURE.md) - Architecture details
- 📊 [API_EXAMPLES.md](API_EXAMPLES.md) - API usage examples
