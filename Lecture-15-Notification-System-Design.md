# Lecture 15: Notification System Design

> **Lecture:** 15  
> **Topic:** System Design  
> **Application:** Multi-Channel Notification Service  
> **Scale:** 1 Million Notifications/Minute  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-14-Leaderboard-System-Design|Lecture 14: Leaderboard System]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Architecture Diagram](#7-complete-architecture-diagram)
8. [Interview Q&A](#8-interview-qa)
9. [Key Takeaways](#9-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Notification System?

A **notification system** is a centralized service that sends notifications across multiple channels (Email, SMS, Push) to end users on behalf of client applications.

**Use Cases:**
- **Email:** Order confirmations, newsletters, password resets
- **SMS:** OTP, bank alerts, delivery updates
- **Push Notifications:** In-app alerts, promotional messages

**Examples:** AWS SNS, Twilio, Firebase Cloud Messaging

### Problem It Solves

1. **Multi-Channel Support:** Email, SMS, Push in one system
2. **Template Management:** Reusable notification templates
3. **User Preferences:** Users control notification channels
4. **Delivery Tracking:** Monitor sent/delivered/failed status
5. **Scalability:** Handle millions of notifications/minute

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Multi-Channel Support**
   - Email notifications
   - SMS notifications
   - In-app push notifications (Android, iOS)

2. **Notification Types**
   - **Real-Time:** OTP, bank alerts (immediate)
   - **Scheduled:** Promotional campaigns (future time)

3. **Template Management**
   - Create reusable templates with variables
   - Example: "Hi `{name}`, your order `{orderId}` is ready!"

4. **Delivery Status Tracking**
   - Pending → Sent → Delivered → Failed
   - Dashboard for clients to monitor status

5. **User Preferences**
   - Users opt-in/out of channels
   - Example: Email ✅, SMS ❌, Push ✅

### 2.2 Non-Functional Requirements

1. **Scale**
   - **1 million notifications/minute**
   - Support multiple clients (Amazon, Flipkart, Uber)

2. **CAP Theorem**
   - **Highly Available (AP):** Eventual consistency acceptable
   
   **Why?**
   - ✅ Template changes can take 1-2 mins to propagate
   - ✅ Downtime = Clients can't send notifications ❌

3. **Latency**
   - **OTP/Critical:** Near real-time (< 1 second)
   - **Promotional:** 5-10 seconds acceptable

4. **Durability**
   - Once accepted, notification MUST be sent (no loss)

---

## 3. Core Entities

1. **Client** - Organizations (Amazon, Flipkart, Uber)
2. **User** - End users of client applications
3. **Notification Preference** - User's channel preferences
4. **Content** - Notification message body
5. **Template** - Reusable message format
6. **Delivery Status** - Sent/Delivered/Failed

**Key Distinction:**
- **Client:** Organization using the notification service
- **User:** End user receiving notifications

---

## 4. API Design

### 4.1 Template APIs

#### **Create Template**

```
POST /v1/templates
```

**Request Body:**
```json
{
  "name": "order_confirmation",
  "type": "TRANSACTIONAL",  // or PROMOTIONAL
  "channel": "EMAIL",  // or SMS, PUSH
  "content": "Hi {{name}}, your order {{orderId}} is confirmed!",
  "variables": ["name", "orderId"],
  "version": "v1"
}
```

---

#### **Get Template**

```
GET /v1/templates/{templateId}?version=v1
```

---

#### **Update/Delete Template**

```
PUT /v1/templates/{templateId}
DELETE /v1/templates/{templateId}
```

---

### 4.2 Send Notification

```
POST /v1/notifications/send
```

**Request Body:**
```json
{
  "templateId": "template_123",
  "recipientId": "user_456",  // External user ID
  "variables": {
    "name": "John Doe",
    "orderId": "ORD-789"
  },
  "channel": "EMAIL",  // or SMS, PUSH
  "priority": "HIGH",  // LOW, MEDIUM, HIGH
  "scheduledAt": "2026-01-22T10:00:00Z"  // Optional (for scheduled)
}
```

**Response:**
```json
{
  "notificationId": "notif_999",
  "status": "PENDING"
}
```

---

### 4.3 Get Notification Status

```
GET /v1/notifications/{notificationId}/status
```

**Response:**
```json
{
  "notificationId": "notif_999",
  "status": "DELIVERED",  // PENDING, SENT, DELIVERED, FAILED
  "channel": "EMAIL",
  "sentAt": "2026-01-21T10:05:32Z",
  "deliveredAt": "2026-01-21T10:05:35Z"
}
```

---

### 4.4 User Preference APIs

```
PUT /v1/users/preferences
```

**Request Body:**
```json
{
  "clientId": "amazon",
  "externalUserId": "user_456",
  "preferences": {
    "email": true,
    "sms": false,
    "push": true
  }
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│         CLIENTS (Amazon, Flipkart, Uber)                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                │  Load Balancer  │
                │ (Rate Limiting) │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│Template  │      │  User    │          │Notification
│ Service  │      │Preference│          │ Service  │
└────┬─────┘      │ Service  │          └────┬─────┘
     │            └────┬─────┘               │
     ▼                 ▼                     ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│Template  │      │Preference│          │ Kafka    │
│   DB     │      │   DB     │          │  Queue   │
└──────────┘      └──────────┘          └────┬─────┘
                                             │
                        ┌────────────────────┼──────────────┐
                        │                    │              │
                        ▼                    ▼              ▼
                   ┌──────────┐        ┌──────────┐  ┌──────────┐
                   │  Email   │        │   SMS    │  │  Push    │
                   │ Provider │        │ Provider │  │ Provider │
                   └────┬─────┘        └────┬─────┘  └────┬─────┘
                        │                   │             │
                        ▼                   ▼             ▼
                   ┌──────────┐        ┌──────────┐  ┌──────────┐
                   │SendGrid  │        │ Twilio   │  │FCM/APNS  │
                   │(Email)   │        │  (SMS)   │  │  (Push)  │
                   └──────────┘        └──────────┘  └──────────┘
```

---

## 6. Low-Level Design (LLD)

### 6.1 Template Service

**Responsibility:** Create, store, and manage notification templates

**Template DB (PostgreSQL):**

```sql
CREATE TABLE templates (
    template_id UUID PRIMARY KEY,
    name VARCHAR(100),
    type VARCHAR(20),  -- TRANSACTIONAL, PROMOTIONAL
    channel VARCHAR(20),  -- EMAIL, SMS, PUSH
    content TEXT,
    variables JSONB,  -- ["name", "orderId"]
    version VARCHAR(10),
    is_active BOOLEAN,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_name_version ON templates(name, version);
```

**Why PostgreSQL?**
- ✅ Relational structure
- ✅ ACID compliance
- ✅ Complex queries for versioning

---

### 6.2 User Preference Service

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│              USER UPDATES PREFERENCE                     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  User Pref      │
                │   Service       │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │(user_preferences)
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Pref Consumer   │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Preference│    │Preference│    │          │
  │   DB     │    │  Cache   │    │          │
  │(Postgres)│    │ (Redis)  │    │          │
  └──────────┘    └──────────┘    └──────────┘
```

---

**Preference DB (PostgreSQL):**

```sql
CREATE TABLE user_preferences (
    id UUID PRIMARY KEY,
    client_id VARCHAR(50),  -- amazon, flipkart
    external_user_id VARCHAR(100),  -- User ID from client system
    preferences JSONB,  -- {"email": true, "sms": false, "push": true}
    updated_at TIMESTAMP
);

CREATE UNIQUE INDEX idx_client_user ON user_preferences(client_id, external_user_id);
```

---

**Preference Cache (Redis):**

```
Key: pref:{clientId}:{userId}
Value: {"email": true, "sms": false, "push": true}
TTL: 1 hour
```

**Why Cache?**
- ✅ Provider services check preferences for EVERY notification
- ✅ Avoid DB hit for millions of requests/minute

---

### 6.3 Notification Service (Core!)

#### **Problem: High Volume + Durability**

```
1 million notifications/minute
    ↓
Need to:
1. Accept requests quickly
2. Guarantee NO message loss
3. Route to correct provider
```

---

#### **Solution: Kafka + Outbox Pattern**

---

### 6.4 Complete Notification Flow

```
┌─────────────────────────────────────────────────────────┐
│         CLIENT SENDS NOTIFICATION REQUEST                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Notification   │
                │    Service      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Notification   │  Outbox  │    │          │
  │    DB    │    │   DB     │    │          │
  │(Status)  │    │(Staging) │    │          │
  └──────────┘    └────┬─────┘    └──────────┘
                       │
                       ▼
                  [CDC Pipeline]
                       │
                       ▼
                ┌─────────────────┐
                │     Kafka       │
                │ (Notification   │
                │     Queue)      │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│  Email   │      │   SMS    │          │  Push    │
│ Provider │      │ Provider │          │ Provider │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     ▼                 ▼                     ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│SendGrid/ │      │ Twilio/  │          │FCM/APNS  │
│  SES     │      │  MSG91   │          │          │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     └────────┬────────┴─────────────────────┘
              │
              ▼
        ┌──────────┐
        │  Kafka   │
        │(delivery)│
        └────┬─────┘
             │
             ▼
        ┌──────────┐
        │Delivery  │
        │Consumer  │
        └────┬─────┘
             │
     ┌───────┼───────┐
     │       │       │
     ▼       ▼       ▼
┌──────────┐┌──────────┐┌──────────┐
│Notification││Notification││         │
│    DB    ││ Event DB ││         │
│(Update)  ││(BigQuery)││         │
└──────────┘└──────────┘└──────────┘
```

---

### 6.5 Outbox Pattern (Durability!)

#### **Problem: Message Loss**

**Naive Approach:**
```
Client → Notification Service → Kafka
    ↓
Service ACKs client immediately
    ↓
Problem: If Kafka crashes before consuming, message lost! ❌
```

---

#### **Solution: Outbox Pattern**

**Outbox DB Schema:**

```sql
CREATE TABLE outbox (
    outbox_id UUID PRIMARY KEY,
    notification_id UUID,
    event_type VARCHAR(50),  -- NOTIFICATION_CREATED
    payload JSONB,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP
);

CREATE INDEX idx_published ON outbox(published);
```

---

**Notification DB Schema:**

```sql
CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY,
    client_id VARCHAR(50),
    external_user_id VARCHAR(100),
    template_id UUID,
    channel VARCHAR(20),  -- EMAIL, SMS, PUSH
    payload JSONB,
    status VARCHAR(20),  -- PENDING, SENT, DELIVERED, FAILED
    priority VARCHAR(20),  -- LOW, MEDIUM, HIGH
    scheduled_at TIMESTAMP,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_status ON notifications(status);
CREATE INDEX idx_client ON notifications(client_id);
```

---

**Flow:**

```
Step 1: Notification Service receives request
    ↓
Step 2: Write to TWO tables (Transaction):
    - notifications (status = PENDING)
    - outbox (published = false)
    ↓
Step 3: ACK to client ✅ (Data is durable in DB)
    ↓
Step 4: CDC (Change Data Capture) reads outbox
    ↓
Step 5: Publish to Kafka
    ↓
Step 6: Mark outbox.published = true
```

**Key Insight:** Data persisted BEFORE ACK = No message loss!

---

### 6.6 Kafka Topics (9 Topics!)

**Structure:** `{priority}_{channel}`

**Topics:**
1. `critical_email`
2. `critical_sms`
3. `critical_push`
4. `standard_email`
5. `standard_sms`
6. `standard_push`
7. `promotional_email`
8. `promotional_sms`
9. `promotional_push`

**Additional Topics:**
- `bulk_email`
- `bulk_sms`
- `retry_queue`
- `dead_letter_queue` (DLQ)

---

**Why 9 Topics?**

```
Priority levels: 3 (critical, standard, promotional)
Channels: 3 (email, sms, push)
    ↓
3 × 3 = 9 topics
```

**Benefit:** Each provider consumes only relevant messages

---

### 6.7 Provider Services

#### **Email Provider:**

**Consumes From:**
- `critical_email`
- `standard_email`
- `promotional_email`

**Flow:**
```java
void processEmail(EmailNotification notification) {
    // 1. Check user preference
    UserPreference pref = cache.get("pref:" + notification.getClientId() + ":" + notification.getUserId());
    
    if (!pref.isEmailEnabled()) {
        log.info("User opted out of email");
        return;
    }
    
    // 2. Render template
    String content = templateEngine.render(notification.getTemplateId(), notification.getVariables());
    
    // 3. Send via external provider
    sendGridClient.send(
        notification.getRecipient(),
        notification.getSubject(),
        content
    );
    
    // 4. Publish delivery status
    kafka.produce("delivery_status", new DeliveryStatus(
        notification.getId(),
        "SENT",
        System.currentTimeMillis()
    ));
}
```

---

#### **SMS Provider:**

**External Services:**
- **Twilio:** SMS, WhatsApp
- **MSG91:** SMS in India

**Flow:** Same as email, but check SMS preference + character limits

---

#### **Push Provider:**

**External Services:**
- **FCM (Firebase Cloud Messaging):** Android
- **APNS (Apple Push Notification Service):** iOS

**Flow:**
```java
void processPush(PushNotification notification) {
    // Check preference
    UserPreference pref = cache.get(...);
    
    if (!pref.isPushEnabled()) {
        return;
    }
    
    // Determine platform (Android or iOS)
    String deviceToken = notification.getDeviceToken();
    String platform = notification.getPlatform();  // "ANDROID" or "IOS"
    
    if (platform.equals("ANDROID")) {
        fcmClient.send(deviceToken, notification.getPayload());
    } else {
        apnsClient.send(deviceToken, notification.getPayload());
    }
    
    // Publish status
    kafka.produce("delivery_status", ...);
}
```

---

### 6.8 Two-Flow Architecture (Critical vs Normal)

#### **Flow 1: Promotional/Standard (Durable)**

```
Client request
    ↓
Notification Service
    ↓
Write to Outbox + Notifications DB (Transaction)
    ↓
ACK to client
    ↓
CDC → Kafka
    ↓
Provider consumes
    ↓
Send to external service
    ↓
Latency: 5-10 seconds ✅
```

---

#### **Flow 2: Critical/OTP (Fast)**

```
Client request (OTP)
    ↓
Notification Service
    ↓
Directly publish to Kafka (critical_sms)
    ↓
ACK to client
    ↓
OTP Provider consumes (horizontally scaled!)
    ↓
Send to Twilio
    ↓
Latency: < 1 second ✅
```

**Why Different Flows?**
- **Promotional:** Durability > Speed
- **OTP:** Speed > Durability (user can retry)

---

### 6.9 Delivery Status Tracking

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│       EXTERNAL PROVIDER (Twilio, SendGrid)               │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼ (Webhook)
                ┌─────────────────┐
                │  Email/SMS      │
                │   Provider      │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │ (delivery_status)
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Delivery      │
                │   Consumer      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Notification   │Notification   │          │
  │    DB    │    │  Event DB │    │          │
  │(Update   │    │ (BigQuery)│    │          │
  │ status)  │    │(Analytics)│    │          │
  └──────────┘    └──────────┘    └──────────┘
```

---

**Notification Event DB (BigQuery - Analytics):**

```sql
CREATE TABLE notification_events (
    event_id STRING,
    notification_id STRING,
    event_type STRING,  -- CREATED, SENT, DELIVERED, FAILED, CLICKED
    timestamp TIMESTAMP,
    metadata JSON
);
```

**Why BigQuery?**
- ✅ Time-series analytics
- ✅ Fast aggregation queries
- ✅ Petabyte scale

---

**Delivery Consumer Logic:**

```java
void processDeliveryStatus(DeliveryStatus status) {
    // 1. Update Notifications DB (latest status)
    db.execute(
        "UPDATE notifications SET status = ?, updated_at = ? WHERE notification_id = ?",
        status.getStatus(),
        status.getTimestamp(),
        status.getNotificationId()
    );
    
    // 2. Insert into Event DB (audit trail)
    bigQuery.insert("notification_events", new Event(
        UUID.randomUUID(),
        status.getNotificationId(),
        status.getStatus(),
        status.getTimestamp(),
        status.getMetadata()
    ));
}
```

---

### 6.10 Reporting Service

**Purpose:** Dashboard for clients to view notification analytics

**Queries:**
- Total sent/delivered/failed (today, week, month)
- Delivery rate by channel
- Latency metrics

**Data Sources:**
1. **Notifications DB:** Current status
2. **Notification Event DB:** Historical events, micro-level changes

---

### 6.11 Webhooks (Delivery Confirmation)

**Flow:**

```
1. Provider sends notification via Twilio
    ↓
2. Twilio delivers to user
    ↓
3. Twilio calls webhook: POST /webhooks/twilio/delivery
    ↓
4. Provider receives webhook
    ↓
5. Publish to Kafka (delivery_status)
    ↓
6. Delivery Consumer updates DB
```

**Webhook Endpoint:**

```java
@PostMapping("/webhooks/twilio/delivery")
void handleTwilioWebhook(@RequestBody TwilioEvent event) {
    kafka.produce("delivery_status", new DeliveryStatus(
        event.getNotificationId(),
        event.getStatus(),  // "delivered", "failed"
        System.currentTimeMillis()
    ));
}
```

---

### 6.12 Rate Limiting (Two Levels!)

#### **Level 1: API Gateway**

**Purpose:** Prevent spam/abuse from clients

```
Rate Limit: 1,000 requests/minute per client
```

**Why?**
- ✅ Protect system from malicious clients
- ✅ Fair usage across all clients

---

#### **Level 2: Provider Service**

**Purpose:** Respect external provider limits

```
Twilio: 100 SMS/second
SendGrid: 500 emails/second
```

**Implementation:**
- Use Token Bucket or Leaky Bucket algorithm
- Queue overflow → backpressure to Kafka

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│              CLIENTS (Amazon, Uber)                      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                │  (Rate Limiter) │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│Template  │      │User Pref │          │Notification
│ Service  │      │ Service  │          │  Service │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     ▼                 ▼                     │
┌──────────┐      ┌──────────┐              │
│Template  │      │  Kafka   │              │
│   DB     │      │(user_pref)              │
└──────────┘      └────┬─────┘              │
                       │                    │
                       ▼                    ▼
                  ┌──────────┐      ┌──────────┐┌──────────┐
                  │Pref      │      │Notification││ Outbox  │
                  │Consumer  │      │    DB    ││   DB    │
                  └────┬─────┘      └──────────┘└────┬─────┘
                       │                            │
                       ▼                            ▼
                  ┌──────────┐              [CDC Pipeline]
                  │Pref DB + │                     │
                  │  Cache   │                     ▼
                  └──────────┘              ┌──────────┐
                                           │  Kafka   │
                                           │  Queue   │
                                           └────┬─────┘
                                                │
                   ┌────────────────────────────┼──────────┐
                   │                            │          │
                   ▼                            ▼          ▼
              ┌──────────┐              ┌──────────┐┌──────────┐
              │  Email   │              │   SMS    ││  Push    │
              │ Provider │              │ Provider ││ Provider │
              └────┬─────┘              └────┬─────┘└────┬─────┘
                   │                         │           │
                   ▼                         ▼           ▼
              ┌──────────┐              ┌──────────┐┌──────────┐
              │SendGrid  │              │ Twilio   ││FCM/APNS  │
              └────┬─────┘              └────┬─────┘└────┬─────┘
                   │                         │           │
                   └────────────┬────────────┴───────────┘
                                │
                                ▼
                         ┌──────────┐
                         │  Kafka   │
                         │(delivery)│
                         └────┬─────┘
                              │
                              ▼
                         ┌──────────┐
                         │Delivery  │
                         │Consumer  │
                         └────┬─────┘
                              │
                    ┌─────────┼─────────┐
                    │         │         │
                    ▼         ▼         ▼
               ┌──────────┐┌──────────┐┌──────────┐
               │Notification││Event DB ││Reporting │
               │    DB    ││(BigQuery)││ Service  │
               └──────────┘└──────────┘└──────────┘
```

---

## 8. Interview Q&A

### Q1: Why Outbox Pattern instead of directly publishing to Kafka?

**A:**

**Problem:**
```
Notification Service → Kafka (ACK to client)
    ↓
Kafka crashes before consumer reads
    ↓
Message lost ❌
Client thinks notification sent ❌
```

**Solution (Outbox):**
```
Notification Service → DB (Transaction)
    ↓
ACK to client ✅ (Data persisted)
    ↓
CDC → Kafka (asynchronously)
    ↓
No message loss ✅
```

### Q2: Why 9 Kafka topics? Why not 1?

**A:**

**One Topic Problem:**
```
All notifications → single topic
    ↓
Email provider consumes ALL (including SMS) ❌
    ↓
Need filtering logic in consumer
```

**9 Topics Solution:**
```
critical_email → Email Provider (only)
critical_sms → SMS Provider (only)
    ↓
No filtering needed ✅
Parallel consumption ✅
```

### Q3: Why different flows for OTP vs Promotional?

**A:**

| Aspect | OTP (Critical) | Promotional |
|--------|----------------|-------------|
| Latency | < 1 second | 5-10 seconds OK |
| Durability | Can retry | Must guarantee |
| Flow | Direct to Kafka | Outbox + CDC |

**OTP:** User can retry if failed
**Promotional:** Cannot retry (scheduled campaigns)

### Q4: How to handle external provider failures?

**A:**

**Strategy:**

1. **Retry Queue:** Exponential backoff (1s, 2s, 4s, 8s)
2. **DLQ (Dead Letter Queue):** After 3 retries
3. **Fallback Provider:** Use alternate (Twilio fails → MSG91)
4. **Circuit Breaker:** Stop sending if provider down

### Q5: How to scale for 1M notifications/minute?

**A:**

**Scaling Strategies:**

1. **Horizontal Scaling:**
   - Multiple instances of each provider service
   - Kafka consumer groups for parallel consumption

2. **Partitioning:**
   - Kafka topics partitioned by region/client
   - Each consumer handles specific partition

3. **Caching:**
   - Preference cache (avoid DB hits)
   - Template cache

4. **Rate Limiting:**
   - Throttle at API Gateway
   - Throttle at provider level

### Q6: How to ensure exactly-once delivery?

**A:**

**Problem:** Kafka retries → duplicate notifications

**Solution: Idempotency**

```java
void sendEmail(EmailNotification notification) {
    String idempotencyKey = notification.getId();
    
    // Check if already sent
    if (redis.exists("sent:" + idempotencyKey)) {
        log.info("Already sent, skipping");
        return;
    }
    
    // Send email
    sendGridClient.send(...);
    
    // Mark as sent (TTL 24h)
    redis.setex("sent:" + idempotencyKey, 86400, "true");
}
```

### Q7: How to handle user preference changes in real-time?

**A:**

**Flow:**
```
User updates preference
    ↓
User Pref Service → Kafka
    ↓
Pref Consumer updates DB + Cache
    ↓
Provider checks cache (TTL: 1 hour)
    ↓
Eventual consistency (1-2 mins) ✅
```

**Cache Invalidation:**
- On preference update → invalidate cache key
- Next provider check → cache miss → fetch from DB

### Q8: What if template changes after notification is queued?

**A:**

**Problem:**
```
Template v1: "Order {{orderId}} confirmed"
    ↓
Notification queued (references template_id)
    ↓
Template updated to v2: "Hi {{name}}, order confirmed"
    ↓
Consumer renders → uses v2 (wrong!) ❌
```

**Solution: Include template snapshot**

```json
{
  "notificationId": "notif_123",
  "templateSnapshot": {
    "id": "template_456",
    "version": "v1",
    "content": "Order {{orderId}} confirmed"
  }
}
```

### Q9: How to monitor system health?

**A:**

**Metrics:**
1. **Latency:** p50, p95, p99 for each channel
2. **Throughput:** Notifications/second
3. **Delivery Rate:** Sent/Delivered/Failed %
4. **Kafka Lag:** Consumer lag per topic
5. **External Provider Status:** Availability

**Alerts:**
- Kafka lag > 10,000 messages
- Delivery rate < 95%
- External provider downtime

### Q10: Security considerations?

**A:**

1. **API Authentication:** JWT tokens for clients
2. **Rate Limiting:** Prevent abuse
3. **PII Protection:** Encrypt phone/email in DB
4. **Webhook Validation:** Verify Twilio signatures
5. **Secret Management:** Store API keys in vault (AWS Secrets Manager)

---

## 9. Key Takeaways

✅ **Outbox Pattern:** Durability (no message loss)  
✅ **CDC Pipeline:** Async publish to Kafka  
✅ **9 Kafka Topics:** Priority × Channel separation  
✅ **Two Flows:** Critical (fast) vs Normal (durable)  
✅ **User Preferences:** Cached (avoid DB hits)  
✅ **Webhooks:** Delivery confirmation from providers  
✅ **Rate Limiting:** API Gateway + Provider level  
✅ **Idempotency:** Prevent duplicate sends  
✅ **Multi-Channel:** Email (SendGrid), SMS (Twilio), Push (FCM/APNS)  
✅ **Analytics:** BigQuery for event tracking  

---

## Summary

**Architecture Highlights:**
- 5 microservices (Template, Preference, Notification, Providers, Reporting)
- 3 databases (Template DB, Preference DB, Notification DB)
- 1 analytics DB (BigQuery)
- Kafka for async messaging (9+ topics)
- Redis for preference caching
- Outbox Pattern for durability

**Notification Flow:**
```
Client → Notification Service → Outbox DB
    ↓
CDC → Kafka (9 topics)
    ↓
Provider (Email/SMS/Push) → External Service
    ↓
Webhook → Delivery Status → Update DB
```

**Two Priority Flows:**
```
Critical (OTP): Direct to Kafka (< 1s)
Normal: Outbox → CDC → Kafka (5-10s)
```

**External Providers:**
- **Email:** SendGrid, AWS SES
- **SMS:** Twilio, MSG91
- **Push:** FCM (Android), APNS (iOS)

**Performance:**
- 1M notifications/minute
- < 1s for critical
- 5-10s for promotional

**End of Lecture 15**
