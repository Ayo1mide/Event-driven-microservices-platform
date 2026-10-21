# Microservices API Examples

Complete request/response examples for all microservices endpoints.

## User Service (Port 8001)

Base URL: `http://localhost:8001`
API Docs: `http://localhost:8001/docs`

### 1. Create User

**Request:**
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "name": "John Doe",
    "password": "SecurePassword123!"
  }'
```

**Response (201 Created):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john.doe@example.com",
  "name": "John Doe",
  "is_active": true,
  "created_at": "2024-01-15T10:30:45.123456",
  "updated_at": "2024-01-15T10:30:45.123456"
}
```

**Event Published:**
```json
{
  "event_id": "evt-123456",
  "event_type": "user.created",
  "aggregate_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2024-01-15T10:30:45.123456",
  "version": 1,
  "source_service": "user-service",
  "data": {
    "email": "john.doe@example.com",
    "name": "John Doe",
    "created_at": "2024-01-15T10:30:45.123456"
  }
}
```

### 2. Get User

**Request:**
```bash
curl http://localhost:8001/api/v1/users/550e8400-e29b-41d4-a716-446655440000
```

**Response (200 OK):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john.doe@example.com",
  "name": "John Doe",
  "is_active": true,
  "created_at": "2024-01-15T10:30:45.123456",
  "updated_at": "2024-01-15T10:30:45.123456"
}
```

### 3. List Users

**Request:**
```bash
curl "http://localhost:8001/api/v1/users?skip=0&limit=10"
```

**Response (200 OK):**
```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "name": "John Doe",
    "is_active": true,
    "created_at": "2024-01-15T10:30:45.123456",
    "updated_at": "2024-01-15T10:30:45.123456"
  },
  {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "email": "jane.smith@example.com",
    "name": "Jane Smith",
    "is_active": true,
    "created_at": "2024-01-15T10:35:20.654321",
    "updated_at": "2024-01-15T10:35:20.654321"
  }
]
```

### 4. Update User

**Request:**
```bash
curl -X PUT http://localhost:8001/api/v1/users/550e8400-e29b-41d4-a716-446655440000 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John David Doe",
    "email": "john.david@example.com"
  }'
```

**Response (200 OK):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john.david@example.com",
  "name": "John David Doe",
  "is_active": true,
  "created_at": "2024-01-15T10:30:45.123456",
  "updated_at": "2024-01-15T10:31:10.654321"
}
```

**Event Published:**
```json
{
  "event_id": "evt-789012",
  "event_type": "user.updated",
  "aggregate_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2024-01-15T10:31:10.654321",
  "version": 1,
  "source_service": "user-service",
  "data": {
    "email": "john.david@example.com",
    "name": "John David Doe",
    "updated_at": "2024-01-15T10:31:10.654321"
  }
}
```

### 5. Delete User

**Request:**
```bash
curl -X DELETE http://localhost:8001/api/v1/users/550e8400-e29b-41d4-a716-446655440000
```

**Response (204 No Content):**
```
(empty body)
```

**Event Published:**
```json
{
  "event_id": "evt-345678",
  "event_type": "user.deleted",
  "aggregate_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2024-01-15T10:32:00.123456",
  "version": 1,
  "source_service": "user-service",
  "data": {
    "deleted_at": "2024-01-15T10:32:00.123456"
  }
}
```

### Error Examples

**400 Bad Request (Validation Error):**
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "invalid-email",
    "name": "John",
    "password": "short"
  }'
```

**Response:**
```json
{
  "error_code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": [
    {
      "loc": ["body", "email"],
      "msg": "invalid email format",
      "type": "value_error.email"
    },
    {
      "loc": ["body", "password"],
      "msg": "ensure this value has at least 8 characters",
      "type": "value_error.any_str.min_length"
    }
  ]
}
```

**409 Conflict (Duplicate Email):**
```bash
curl -X POST http://localhost:8001/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "name": "Another User",
    "password": "AnotherPassword123!"
  }'
```

**Response:**
```json
{
  "error_code": "CONFLICT",
  "message": "User with email john.doe@example.com already exists",
  "details": {}
}
```

---

## Billing Service (Port 8002)

Base URL: `http://localhost:8002`
API Docs: `http://localhost:8002/docs`

### 1. Create Invoice

**Request:**
```bash
curl -X POST http://localhost:8002/api/v1/invoices \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 99.99,
    "currency": "USD",
    "due_date": "2024-02-15",
    "description": "Monthly subscription for January 2024"
  }'
```

**Response (201 Created):**
```json
{
  "id": "inv-123456-789012",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 99.99,
  "currency": "USD",
  "status": "draft",
  "due_date": "2024-02-15",
  "description": "Monthly subscription for January 2024",
  "created_at": "2024-01-15T10:40:00.123456",
  "updated_at": "2024-01-15T10:40:00.123456"
}
```

### 2. Get Invoice

**Request:**
```bash
curl http://localhost:8002/api/v1/invoices/inv-123456-789012
```

**Response (200 OK):**
```json
{
  "id": "inv-123456-789012",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 99.99,
  "currency": "USD",
  "status": "draft",
  "due_date": "2024-02-15",
  "description": "Monthly subscription for January 2024",
  "created_at": "2024-01-15T10:40:00.123456",
  "updated_at": "2024-01-15T10:40:00.123456"
}
```

### 3. List Invoices (by User)

**Request:**
```bash
curl "http://localhost:8002/api/v1/invoices?user_id=550e8400-e29b-41d4-a716-446655440000&skip=0&limit=10"
```

**Response (200 OK):**
```json
[
  {
    "id": "inv-123456-789012",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 99.99,
    "currency": "USD",
    "status": "draft",
    "due_date": "2024-02-15",
    "description": "Monthly subscription for January 2024",
    "created_at": "2024-01-15T10:40:00.123456",
    "updated_at": "2024-01-15T10:40:00.123456"
  }
]
```

### 4. Create Payment (Process Payment)

**Request:**
```bash
curl -X POST http://localhost:8002/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "invoice_id": "inv-123456-789012",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 99.99,
    "transaction_id": "txn-2024-01-15-001"
  }'
```

**Response (201 Created - Success):**
```json
{
  "id": "pmt-987654-321098",
  "invoice_id": "inv-123456-789012",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 99.99,
  "currency": "USD",
  "status": "completed",
  "transaction_id": "txn-2024-01-15-001",
  "failure_reason": null,
  "created_at": "2024-01-15T10:45:00.123456",
  "updated_at": "2024-01-15T10:45:02.654321"
}
```

**Events Published (on successful payment):**
```json
{
  "event_id": "evt-pay-success",
  "event_type": "billing.payment.received",
  "aggregate_id": "pmt-987654-321098",
  "timestamp": "2024-01-15T10:45:02.654321",
  "version": 1,
  "source_service": "billing-service",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "invoice_id": "inv-123456-789012",
    "amount": 99.99,
    "currency": "USD",
    "processed_at": "2024-01-15T10:45:02.654321"
  }
}
```

**Response (201 Created - Failure):**
```json
{
  "id": "pmt-654321-098765",
  "invoice_id": "inv-123456-789012",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 99.99,
  "currency": "USD",
  "status": "failed",
  "transaction_id": "txn-2024-01-15-002",
  "failure_reason": "Insufficient funds",
  "created_at": "2024-01-15T10:50:00.123456",
  "updated_at": "2024-01-15T10:50:01.123456"
}
```

**Event Published (on payment failure):**
```json
{
  "event_id": "evt-pay-fail",
  "event_type": "billing.payment.failed",
  "aggregate_id": "pmt-654321-098765",
  "timestamp": "2024-01-15T10:50:01.123456",
  "version": 1,
  "source_service": "billing-service",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "invoice_id": "inv-123456-789012",
    "amount": 99.99,
    "failure_reason": "Insufficient funds",
    "failed_at": "2024-01-15T10:50:01.123456"
  }
}
```

### 5. Get Payment

**Request:**
```bash
curl http://localhost:8002/api/v1/payments/pmt-987654-321098
```

**Response (200 OK):**
```json
{
  "id": "pmt-987654-321098",
  "invoice_id": "inv-123456-789012",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 99.99,
  "currency": "USD",
  "status": "completed",
  "transaction_id": "txn-2024-01-15-001",
  "failure_reason": null,
  "created_at": "2024-01-15T10:45:00.123456",
  "updated_at": "2024-01-15T10:45:02.654321"
}
```

---

## Notification Service (Port 8003)

Base URL: `http://localhost:8003`
API Docs: `http://localhost:8003/docs`

### 1. Get User Notifications

**Request:**
```bash
curl "http://localhost:8003/api/v1/notifications/users/550e8400-e29b-41d4-a716-446655440000"
```

**Response (200 OK):**
```json
[
  {
    "id": "notif-001",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "notification_type": "email",
    "recipient": "john.doe@example.com",
    "subject": "Welcome to Microservices Platform",
    "body": "Welcome John Doe! Your account has been created.",
    "status": "sent",
    "event_id": "evt-123456",
    "retry_count": 0,
    "error_message": null,
    "created_at": "2024-01-15T10:30:45.123456",
    "updated_at": "2024-01-15T10:30:46.654321"
  },
  {
    "id": "notif-002",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "notification_type": "email",
    "recipient": "john.doe@example.com",
    "subject": "Payment Received",
    "body": "Payment of $99.99 USD has been received. Thank you!",
    "status": "sent",
    "event_id": "evt-pay-success",
    "retry_count": 1,
    "error_message": null,
    "created_at": "2024-01-15T10:45:02.654321",
    "updated_at": "2024-01-15T10:45:05.123456"
  }
]
```

### 2. Get Single Notification

**Request:**
```bash
curl http://localhost:8003/api/v1/notifications/notif-001
```

**Response (200 OK):**
```json
{
  "id": "notif-001",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "notification_type": "email",
  "recipient": "john.doe@example.com",
  "subject": "Welcome to Microservices Platform",
  "body": "Welcome John Doe! Your account has been created.",
  "status": "sent",
  "event_id": "evt-123456",
  "retry_count": 0,
  "error_message": null,
  "created_at": "2024-01-15T10:30:45.123456",
  "updated_at": "2024-01-15T10:30:46.654321"
}
```

### 3. List All Notifications (with status filter)

**Request:**
```bash
curl "http://localhost:8003/api/v1/notifications?status=sent&skip=0&limit=50"
```

**Response (200 OK):**
```json
[
  {
    "id": "notif-001",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "notification_type": "email",
    "recipient": "john.doe@example.com",
    "subject": "Welcome to Microservices Platform",
    "body": "Welcome John Doe! Your account has been created.",
    "status": "sent",
    "event_id": "evt-123456",
    "retry_count": 0,
    "error_message": null,
    "created_at": "2024-01-15T10:30:45.123456",
    "updated_at": "2024-01-15T10:30:46.654321"
  }
]
```

### 4. Get Notification Statistics

**Request:**
```bash
curl http://localhost:8003/api/v1/notifications/stats/summary
```

**Response (200 OK):**
```json
{
  "summary": {
    "pending": 5,
    "sent": 42,
    "failed": 2,
    "bounced": 1
  },
  "total": 50
}
```

### 5. Failed Notification with Retries

**Response (200 OK):**
```json
{
  "id": "notif-failed-001",
  "user_id": "660e8400-e29b-41d4-a716-446655440001",
  "notification_type": "email",
  "recipient": "invalid-email@",
  "subject": "Payment Reminder",
  "body": "Your payment is due on 2024-02-15",
  "status": "failed",
  "event_id": "evt-failure-001",
  "retry_count": 3,
  "error_message": "Invalid email address format",
  "created_at": "2024-01-15T11:00:00.123456",
  "updated_at": "2024-01-15T11:05:30.654321"
}
```

---

## Health Check Endpoints

### Liveness Probe
```bash
curl http://localhost:8001/health/live
```

**Response (200 OK):**
```json
{
  "status": "UP",
  "probe": "liveness"
}
```

### Readiness Probe
```bash
curl http://localhost:8001/health/ready
```

**Response (200 OK):**
```json
{
  "status": "UP",
  "probe": "readiness"
}
```

### Full Health Check
```bash
curl http://localhost:8001/health
```

**Response (200 OK):**
```json
{
  "service": "user-service",
  "status": "UP",
  "timestamp": "2024-01-15T11:15:30.123456",
  "checks": {
    "database": {
      "status": "UP",
      "details": {
        "message": "Database connection OK"
      }
    }
  }
}
```

---

## Event Consumption Sequence

### Scenario: Complete User Registration to Payment Flow

**Step 1: Create User (User Service)**
```
POST /api/v1/users → Event: user.created
```

**Step 2: Events Consumed (Async)**
```
Billing Service consumes user.created
  → Creates invoice
  → Publishes billing.invoice.created

Notification Service consumes user.created
  → Creates notification record
  → Queues Celery task: send_email
```

**Step 3: Email Sent (Celery Worker)**
```
Celery Worker picks up send_email task
  → Sends welcome email
  → Updates notification status to "sent"
```

**Step 4: Process Payment (Billing Service)**
```
POST /api/v1/payments → Payment processing
  → Success: billing.payment.received event
  → Notification Service consumes event
  → Celery task: send_email (payment confirmation)
```

This entire flow happens asynchronously with guaranteed delivery and retry logic.
