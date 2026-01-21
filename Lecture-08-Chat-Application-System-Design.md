# Lecture 8: Chat Application System Design (WhatsApp/Messenger)

> **Lecture:** 08  
> **Topic:** System Design  
> **Application:** Chat Application (WhatsApp, Facebook Messenger, Telegram)  
> **Scale:** 1B Users, 100B Messages/Day  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-07-Proximity-Search-Algorithms|Lecture 7: Proximity Search]]

---

## 1. Brief Introduction

### What is a Chat Application?

A **chat application** enables users to send and receive text messages and media files in real-time through one-to-one or group conversations.

**Examples:** WhatsApp, Facebook Messenger, Telegram, Signal, WeChat

### Problem It Solves

1. **Real-Time Communication:** Instant message delivery
2. **One-to-One Chat:** Private conversations
3. **Group Chat:** Multiple participants
4. **Media Sharing:** Images, videos, documents
5. **Message History:** Persistent chat records
6. **Delivery Receipts:** Sent, delivered, read status
7. **Online Status:** See who's active

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **User Registration**
   - Sign up with phone number (WhatsApp) or email (Messenger)
   - Login/logout

2. **One-to-One Messaging**
   - Send text messages
   - Receive messages in real-time

3. **Group Messaging**
   - Create groups
   - Add/remove members
   - Send messages to group

4. **Message History**
   - View past conversations
   - Search messages

5. **Media Sharing**
   - Send images, videos, documents
   - Receive media files

6. **Delivery Receipts**
   - ✓ Sent (message reached server)
   - ✓✓ Delivered (message reached recipient)
   - ✓✓ (Blue) Read (recipient opened message)

7. **Online Status**
   - Show "online" indicator
   - Show "last seen" timestamp

### 2.2 Non-Functional Requirements

1. **Scale**
   - **1 billion users**
   - **100 messages/user/day**
   - **100 billion messages/day**
   - **Storage:** 100B × 1 KB = **100 TB/day**

2. **CAP Theorem**
   - **Highly Available (AP):** Zero downtime
   - **Eventual Consistency:** Messages can have slight delay
   - **Trade-off:** Availability > Consistency

3. **Low Latency**
   - **Message delivery:** < 300ms
   - **Real-time updates**

4. **High Reliability**
   - **Zero message loss**
   - **Guaranteed delivery**

---

## 3. Core Entities

1. **User** - Person using the app
2. **Message** - Text or media content
3. **Chat** - One-to-one conversation
4. **Group** - Multi-user conversation
5. **WebSocket Connection** - Real-time channel

---

## 4. API Design

### 4.1 User Registration

```
POST /v1/users/register
```

**Request Body:**
```json
{
  "phone": "+919876543210",  // WhatsApp
  "email": "user@example.com",  // Messenger
  "name": "John Doe",
  "password": "hashed_password"
}
```

**Additional APIs** (mention verbally):
- `POST /v1/users/login`
- `POST /v1/users/logout`
- `PUT /v1/users/profile`

---

### 4.2 One-to-One Messaging (WebSocket!)

```
WS /v1/chat/send
```

⚠️ **Important:** This is a **WebSocket** connection, not HTTP!

**Message Format:**
```json
{
  "action": "send",
  "senderId": "user_123",
  "receiverId": "user_456",
  "message": "Hello!",
  "type": "text",  // text, image, video
  "timestamp": "2026-01-21T12:00:00Z"
}
```

**Response (Acknowledgment):**
```json
{
  "status": "sent",
  "messageId": "msg_789",
  "timestamp": "2026-01-21T12:00:01Z"
}
```

---

### 4.3 Get Chat History

```
GET /v1/chats?userId={userId}
```

**Response:**
```json
{
  "chats": [
    {
      "chatId": "chat_101",
      "userId": "user_456",
      "userName": "Alice",
      "lastMessage": "See you tomorrow!",
      "timestamp": "2026-01-21T11:30:00Z",
      "unreadCount": 3
    }
  ]
}
```

---

### 4.4 Get Messages in Chat

```
GET /v1/chats/{chatId}/messages?offset={offset}&limit={limit}
```

**Query Parameters:**
- `offset`: Starting position (for lazy loading)
- `limit`: Number of messages (default: 50)

**Response:**
```json
{
  "messages": [
    {
      "messageId": "msg_789",
      "senderId": "user_123",
      "message": "Hello!",
      "type": "text",
      "timestamp": "2026-01-21T12:00:00Z",
      "deliveryStatus": "read"
    }
  ],
  "hasMore": true
}
```

⚠️ **Note:** This uses **lazy loading** (not pagination with page numbers)

---

### 4.5 Group Management

**Create Group:**
```
POST /v1/groups
```

**Request Body:**
```json
{
  "name": "Family Group",
  "members": ["user_123", "user_456", "user_789"]
}
```

**Add Member:**
```
POST /v1/groups/{groupId}/members
```

**Request Body:**
```json
{
  "userId": "user_101"
}
```

**Remove Member:**
```
DELETE /v1/groups/{groupId}/members/{userId}
```

---

### 4.6 Group Messaging (WebSocket!)

```
WS /v1/groups/{groupId}/send
```

**Message Format:**
```json
{
  "action": "send",
  "senderId": "user_123",
  "groupId": "group_456",
  "message": "Hello everyone!",
  "type": "text",
  "timestamp": "2026-01-21T12:00:00Z"
}
```

---

### 4.7 Get Group Messages

```
GET /v1/groups/{groupId}/messages?offset={offset}&limit={limit}
```

**Response:**
```json
{
  "messages": [
    {
      "messageId": "msg_789",
      "senderId": "user_123",
      "senderName": "John",
      "message": "Hello everyone!",
      "timestamp": "2026-01-21T12:00:00Z"
    }
  ]
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    1B Users                              │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   API    │    │WebSocket │    │   CDN    │
  │ Gateway  │    │ Gateway  │    │          │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       │               │                │
  ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐
  │  User    │    │  Chat    │    │  Media   │
  │ Service  │    │ Service  │    │ Upload   │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ User DB  │    │ Chat DB  │    │ S3       │
  │(Postgres)│    │(Cassandra)│   │ Bucket   │
  └──────────┘    └──────────┘    └──────────┘


  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Group   │    │ Message  │    │  Search  │
  │ Service  │    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Group DB │    │  Redis   │    │Elastic   │
  │(Postgres)│    │  Stream  │    │ Search   │
  └──────────┘    └──────────┘    └──────────┘
```

### 5.2 Service Responsibilities

**User Service:**
- User registration, login
- Return JWT token

**Chat Service:**
- Handle WebSocket connections
- Send/receive messages
- Route messages to recipients

**Group Service:**
- Create/manage groups
- Get group members

**Message Service:**
- Persist messages to database
- Retrieve undelivered messages

**Media Upload Service:**
- Upload images/videos to S3
- Return S3 URL

**Search Service:**
- Search messages by keyword
- Retrieve message history

---

## 6. Low-Level Design (LLD)

### 6.1 User Service

**Database:** PostgreSQL

**Schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    password_hash VARCHAR(255),
    status VARCHAR(20),  -- online, offline
    last_seen TIMESTAMP,
    created_at TIMESTAMP
);
```

**Flow:**
1. User registers → Store in PostgreSQL
2. User logs in → Validate → Return JWT token
3. JWT used in all API calls

---

### 6.2 WebSocket Connection (The Heart of the System!)

#### **Why WebSocket?**

**Four Connection Types:**

| Type | Direction | Use Case | Pros | Cons |
|------|-----------|----------|------|------|
| **HTTP** | Client → Server | REST APIs | Simple | ❌ No real-time |
| **Long Polling** | Client ↔ Server | Notifications | Better than HTTP | ❌ Many connections |
| **SSE** | Server → Client | Push notifications | Server push | ❌ One-way only |
| **WebSocket** | Client ↔ Server | Chat | ✅ Real-time, bidirectional | Complex |

---

#### **Why NOT HTTP?**

**Problem:**
```
User sends message → POST /messages
User receives message → GET /messages (poll every 1 second)
    ↓
100 requests/minute just to check for new messages! ❌
```

**Why it fails:**
- ❌ Too many requests (polling)
- ❌ High latency (1-second delay)
- ❌ Wasted bandwidth (empty responses)

---

#### **Why NOT Long Polling?**

**Problem:**
```
User sends message → POST /messages
User waits for reply → Long poll (30 seconds timeout)
    ↓
If no reply → Timeout → Open new long poll
    ↓
Repeat forever! ❌
```

**Why it fails:**
- ❌ Multiple connections (one per 30 seconds)
- ❌ Unpredictable reply time
- ❌ Connection overhead

---

#### **Why NOT SSE (Server-Sent Events)?**

**Problem:**
```
User sends message → POST /messages
Server pushes reply → SSE notification
User fetches message → GET /messages
    ↓
2-3 requests per message! ❌
```

**Why it fails:**
- ❌ One-way only (server → client)
- ❌ Still need HTTP for sending
- ❌ Multiple connections

---

#### **Why WebSocket? ✅**

**Solution:**
```
User opens WebSocket → Single persistent connection
User sends message → WebSocket.send()
User receives message → WebSocket.onMessage()
    ↓
ONE connection for both send and receive! ✅
```

**Why it works:**
- ✅ **Bidirectional:** Send and receive simultaneously
- ✅ **Persistent:** Connection stays open
- ✅ **Low latency:** < 300ms
- ✅ **Efficient:** No polling overhead

---

### 6.3 WebSocket Connection Flow

#### **Step 1: Establish WebSocket Connection**

**Process:**

```
1. Client sends HTTP request:
   GET /v1/chat/connect HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade

2. Server validates JWT token

3. Server performs handshake:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade

4. Connection upgraded to WebSocket ✅
```

⚠️ **Important:** **All WebSocket connections start as HTTP!**

---

#### **Step 2: WebSocket Registry (Redis Cache)**

**Problem:** How does Chat Service know which user is connected to which WebSocket?

**Solution:** **WebSocket Registry**

**Redis Schema:**

**Key:** `ws:user:{userId}`

**Value:**
```json
{
  "userId": "user_123",
  "connectionId": "ws_conn_456",
  "chatServiceId": "chat_service_1",
  "connectedAt": "2026-01-21T12:00:00Z",
  "lastActivity": "2026-01-21T12:05:00Z"
}
```

**TTL:** 60 seconds (auto-expire if inactive)

---

**Why Redis?**
- ✅ **Fast lookup:** O(1) to find user's connection
- ✅ **TTL support:** Auto-remove inactive users
- ✅ **Shared state:** All Chat Services can access

---

#### **Step 3: Sticky Connections**

**Problem:** User must always connect to the **same Chat Service**

**Why?**
```
User 1 connects to Chat Service 1
User 1 disconnects (network issue)
User 1 reconnects → Must go to Chat Service 1 (not 2 or 3!)
```

**Solution:** **WebSocket Gateway** routes based on userId

**Algorithm:**
```python
def route_websocket(user_id):
    # Check if user already has a connection
    connection = redis.get(f"ws:user:{user_id}")
    
    if connection:
        # Route to existing Chat Service
        return connection['chatServiceId']
    else:
        # Assign to least-loaded Chat Service
        chat_service = get_least_loaded_service()
        return chat_service
```

---

### 6.4 Sending Messages (One-to-One)

#### **Flow:**

```
User 1 (connected to Chat Service 1)
    ↓
Sends message to User 4
    ↓
Chat Service 1 checks: Is User 4 online?
    ↓
Redis Lookup: ws:user:user_4
    ↓
    ├─ FOUND → User 4 is online (Chat Service 2)
    │   ↓
    │   Send message via Redis Stream
    │   ↓
    │   Chat Service 2 receives message
    │   ↓
    │   Chat Service 2 sends to User 4 via WebSocket
    │
    └─ NOT FOUND → User 4 is offline
        ↓
        Send message to Redis Stream (offline channel)
        ↓
        Message Service persists to database
        ↓
        Notification Service sends push notification
```

---

#### **Detailed Steps:**

**1. User 1 sends message:**
```json
{
  "action": "send",
  "senderId": "user_1",
  "receiverId": "user_4",
  "message": "Hello!",
  "type": "text"
}
```

**2. Chat Service 1 checks Redis:**
```python
receiver_connection = redis.get("ws:user:user_4")

if receiver_connection:
    # User 4 is online
    target_service = receiver_connection['chatServiceId']
    channel = f"chat:{target_service}"
    
    # Publish to Redis Stream
    redis.xadd(channel, {
        "senderId": "user_1",
        "receiverId": "user_4",
        "message": "Hello!",
        "type": "text"
    })
else:
    # User 4 is offline
    redis.xadd("offline_messages", {
        "senderId": "user_1",
        "receiverId": "user_4",
        "message": "Hello!",
        "type": "text"
    })
```

**3. Chat Service 2 consumes from Redis Stream:**
```python
# Chat Service 2 subscribes to "chat:chat_service_2"
message = redis.xread("chat:chat_service_2")

# Send to User 4 via WebSocket
websocket.send(user_4_connection, message)
```

**4. User 4 receives message:**
```json
{
  "messageId": "msg_789",
  "senderId": "user_1",
  "message": "Hello!",
  "timestamp": "2026-01-21T12:00:00Z"
}
```

**5. User 4 sends acknowledgment:**
```json
{
  "action": "ack",
  "messageId": "msg_789",
  "status": "delivered"
}
```

---

### 6.5 Redis Stream (Why Not Kafka?)

**Comparison:**

| Feature | Redis Stream | Kafka |
|---------|--------------|-------|
| **Latency** | < 10ms | 50-100ms |
| **Throughput** | 100K msg/sec | 1M msg/sec |
| **Persistence** | In-memory (optional disk) | Disk-based |
| **Use Case** | Real-time chat | Event streaming |

**Decision:** **Redis Stream** (lower latency for chat)

---

**Redis Stream Channels:**

```
chat:chat_service_1 → Messages for users on Chat Service 1
chat:chat_service_2 → Messages for users on Chat Service 2
offline_messages → Messages for offline users
```

**Example:**
```python
# Publish message
redis.xadd("chat:chat_service_2", {
    "senderId": "user_1",
    "receiverId": "user_4",
    "message": "Hello!"
})

# Consume message (Chat Service 2)
messages = redis.xread({"chat:chat_service_2": "0"})
for message in messages:
    send_to_user(message)
```

---

### 6.6 Message Persistence (Cassandra)

**Why Cassandra?**
- ✅ **Write-heavy:** 100B messages/day
- ✅ **Scalable:** Horizontal scaling
- ✅ **High availability:** No single point of failure

**Schema:**
```sql
CREATE TABLE messages (
    chat_id UUID,
    message_id UUID,
    sender_id UUID,
    receiver_id UUID,
    message TEXT,
    type VARCHAR(20),  -- text, image, video
    timestamp TIMESTAMP,
    delivery_status VARCHAR(20),  -- sent, delivered, read
    PRIMARY KEY (chat_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Why this schema?**
- `chat_id` = Partition key (all messages in one chat on same node)
- `timestamp` = Clustering key (sorted by time)
- `message_id` = Unique identifier

---

**Message Service (Consumer):**

```python
# Subscribe to offline_messages channel
while True:
    messages = redis.xread({"offline_messages": "0"})
    
    for message in messages:
        # Persist to Cassandra
        cassandra.execute("""
            INSERT INTO messages (chat_id, message_id, sender_id, receiver_id, message, type, timestamp, delivery_status)
            VALUES (?, ?, ?, ?, ?, ?, ?, 'sent')
        """, [chat_id, message_id, sender_id, receiver_id, message, type, timestamp])
```

---

### 6.7 Offline Message Handling

**Scenario:** User 4 is offline, User 1 sends message

**Flow:**

```
1. Chat Service 1 checks Redis → User 4 not found

2. Publish to "offline_messages" channel

3. Message Service consumes → Persist to Cassandra

4. Notification Service consumes → Send push notification
   ├─ Android: FCM (Firebase Cloud Messaging)
   └─ iOS: APNS (Apple Push Notification Service)

5. User 4 comes online → Opens WebSocket

6. Chat Service calls Message Service:
   "Get all undelivered messages for user_4"

7. Message Service queries Cassandra:
   SELECT * FROM messages
   WHERE receiver_id = 'user_4'
   AND delivery_status = 'sent'

8. Chat Service sends messages to User 4 via WebSocket

9. User 4 sends acknowledgment → Update delivery_status = 'delivered'
```

---

### 6.8 Delivery Receipts (✓, ✓✓, ✓✓ Blue)

**Three States:**

| Status | Icon | Meaning |
|--------|------|---------|
| **Sent** | ✓ | Message reached server |
| **Delivered** | ✓✓ | Message reached recipient's device |
| **Read** | ✓✓ (Blue) | Recipient opened message |

---

**Flow:**

**1. Sent (✓):**
```
User 1 sends message
    ↓
Chat Service receives message
    ↓
Chat Service sends acknowledgment to User 1
    ↓
User 1 shows ✓
```

**2. Delivered (✓✓):**
```
Chat Service sends message to User 4
    ↓
User 4's device receives message
    ↓
User 4 sends acknowledgment: "delivered"
    ↓
Chat Service forwards to User 1
    ↓
User 1 shows ✓✓
```

**3. Read (✓✓ Blue):**
```
User 4 opens chat
    ↓
User 4 sends acknowledgment: "read"
    ↓
Chat Service forwards to User 1
    ↓
User 1 shows ✓✓ (Blue)
```

**All via WebSocket!** (No separate API calls)

---

### 6.9 Group Messaging

**Challenge:** Send message to multiple users efficiently

**Naive Approach (❌):**
```
Group has 100 members
    ↓
Send message 100 times (one per member)
    ↓
Too slow! ❌
```

**Optimized Approach (✅):**
```
1. User 1 sends message to Group 456

2. Chat Service calls Group Service:
   "Get all members of Group 456"

3. Group Service returns: [user_2, user_3, user_4, ...]

4. Chat Service iterates through members:
   FOR EACH member:
       Check if online (Redis lookup)
       IF online:
           Send via Redis Stream
       ELSE:
           Publish to offline_messages

5. Each Chat Service sends to its connected users
```

---

**Why individual messages?**

**Reason:** **Individual delivery status**

```
Group message sent to 10 users:
    ├─ User 2: ✓✓ (Read)
    ├─ User 3: ✓✓ (Delivered)
    ├─ User 4: ✓ (Sent)
    └─ User 5: ✓ (Sent)
```

WhatsApp shows **who read the message!**

**Implementation:** Each message has separate `delivery_status` per recipient

---

### 6.10 Group Database

**Schema:**

**Table 1: Groups**
```sql
CREATE TABLE groups (
    group_id UUID PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    thumbnail_url TEXT,
    created_at TIMESTAMP
);
```

**Table 2: Group Members**
```sql
CREATE TABLE group_members (
    id SERIAL PRIMARY KEY,  -- Auto-increment
    group_id UUID REFERENCES groups(group_id),
    user_id UUID REFERENCES users(user_id),
    joined_at TIMESTAMP,
    role VARCHAR(20)  -- admin, member
);

-- Index for fast lookup
CREATE INDEX idx_group_members ON group_members(group_id);
```

**Why `id` column?**
- `group_id` is not unique (one group has many members)
- Need unique primary key

---

### 6.11 Media Upload (Images/Videos)

**Challenge:** Cannot send large files via WebSocket!

**Solution:** Separate upload flow

**Flow:**

```
1. User selects image

2. Client calls Media Upload Service (HTTP POST)
   POST /v1/media/upload
   Content-Type: multipart/form-data

3. Media Upload Service uploads to S3

4. S3 returns URL: https://cdn.example.com/images/abc123.jpg

5. Media Upload Service returns URL to client

6. Client sends message via WebSocket:
   {
     "action": "send",
     "message": "https://cdn.example.com/images/abc123.jpg",
     "type": "image"
   }

7. Recipient receives URL → Downloads from CDN
```

---

**Why CDN?**
- ✅ **Low latency:** Geographically distributed
- ✅ **High bandwidth:** Handles millions of downloads
- ✅ **Caching:** Popular images cached

---

### 6.12 Online Status & Last Seen

**Challenge:** Show "online" or "last seen 5 minutes ago"

**Solution:** Use WebSocket Registry

**Logic:**

```python
def get_user_status(user_id):
    connection = redis.get(f"ws:user:{user_id}")
    
    if connection:
        # User has active WebSocket
        return "online"
    else:
        # User is offline, get last seen
        user = db.query("SELECT last_seen FROM users WHERE user_id = ?", user_id)
        return f"last seen {format_time(user.last_seen)}"
```

---

**Updating Last Seen:**

**Flow:**

```
1. User's WebSocket disconnects (network loss, app closed)

2. Redis TTL expires (60 seconds)

3. Redis sends event: "ws:user:user_123 expired"

4. User Management Service listens to Redis events

5. User Management Service updates database:
   UPDATE users
   SET status = 'offline', last_seen = NOW()
   WHERE user_id = 'user_123'
```

**Implementation:** Redis Keyspace Notifications + CDC Pipeline

---

### 6.13 Message Search (Elasticsearch)

**Use Case:** Search "pizza" in chat history

**Flow:**

```
1. User types "pizza" in search bar

2. Client calls Search Service:
   GET /v1/search?query=pizza&chatId=chat_101

3. Search Service queries Elasticsearch

4. Elasticsearch returns matching messages

5. Search Service fetches media URLs from S3 (if needed)

6. Return results to client
```

---

**Elasticsearch Index:**

```json
{
  "mappings": {
    "properties": {
      "message_id": { "type": "keyword" },
      "chat_id": { "type": "keyword" },
      "sender_id": { "type": "keyword" },
      "message": { "type": "text" },
      "type": { "type": "keyword" },
      "timestamp": { "type": "date" }
    }
  }
}
```

**Query:**
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "pizza" } },
        { "term": { "chat_id": "chat_101" } }
      ]
    }
  },
  "sort": [
    { "timestamp": "desc" }
  ]
}
```

---

**CDC Pipeline:**
```
Cassandra (messages table)
    ↓
Debezium (Change Data Capture)
    ↓
Kafka
    ↓
Consumer Service
    ↓
Elasticsearch
```

---

### 6.14 Message History (Lazy Loading)

**Challenge:** Load old messages as user scrolls

**Flow:**

```
1. User opens chat → Load last 50 messages

2. User scrolls up → Load next 50 messages

3. Repeat until no more messages
```

**API:**
```
GET /v1/chats/{chatId}/messages?offset=0&limit=50
```

**Response:**
```json
{
  "messages": [...],
  "hasMore": true
}
```

**Next request:**
```
GET /v1/chats/{chatId}/messages?offset=50&limit=50
```

---

**Optimization:** **Local Storage**

WhatsApp claims: "We don't store messages on servers"

**How?**
- ✅ Messages stored locally on device (SQLite database)
- ✅ Only undelivered messages stored on server
- ✅ Once delivered → Delete from server

**Implementation:**
```python
# After message delivered
if message.delivery_status == 'delivered':
    cassandra.execute("DELETE FROM messages WHERE message_id = ?", message_id)
```

**Result:** Reduced server storage!

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   API    │    │WebSocket │    │   CDN    │
  │ Gateway  │    │ Gateway  │    │          │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       │               │                │
  ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐
  │  User    │    │  Chat    │    │  Media   │
  │ Service  │    │ Service  │    │ Upload   │
  └────┬─────┘    │ (WS)     │    │ Service  │
       │          └────┬─────┘    └────┬─────┘
       │               │                │
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ User DB  │    │WebSocket │    │   S3     │
  │(Postgres)│    │ Registry │    │  Bucket  │
  └──────────┘    │ (Redis)  │    └──────────┘
                  └────┬─────┘
                       │
                       ▼
                  ┌──────────┐
                  │  Redis   │
                  │  Stream  │
                  └────┬─────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Message  │  │  Group   │  │Notification│
  │ Service  │  │ Service  │  │  Service  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │              │
       ▼             ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Chat DB  │  │ Group DB │  │FCM/APNS  │
  │(Cassandra)│ │(Postgres)│  │          │
  └────┬─────┘  └──────────┘  └──────────┘
       │
       │ CDC
       ▼
  ┌──────────┐
  │Elastic   │
  │ Search   │
  └──────────┘
```

---

## 8. Complete Message Flow

### 8.1 User Comes Online

```
Step 1: User opens app
    ↓
Step 2: Login via User Service → Get JWT token
    ↓
Step 3: Request WebSocket connection
    HTTP GET /v1/chat/connect
    Upgrade: websocket
    ↓
Step 4: WebSocket Gateway checks Redis
    IF connection exists:
        Route to existing Chat Service
    ELSE:
        Assign to least-loaded Chat Service
    ↓
Step 5: WebSocket handshake → Connection established
    ↓
Step 6: Chat Service registers in Redis:
    SET ws:user:user_123 {connectionId, chatServiceId}
    EXPIRE ws:user:user_123 60
    ↓
Step 7: Chat Service calls Message Service:
    "Get undelivered messages for user_123"
    ↓
Step 8: Message Service queries Cassandra:
    SELECT * FROM messages
    WHERE receiver_id = 'user_123'
    AND delivery_status = 'sent'
    ↓
Step 9: Chat Service sends messages via WebSocket
    ↓
Step 10: User receives messages → Sends acknowledgment
    ↓
Step 11: Update delivery_status = 'delivered'
    ↓
Step 12: User opens chat → Sends acknowledgment
    ↓
Step 13: Update delivery_status = 'read'
```

---

### 8.2 User Sends Message (Recipient Online)

```
Step 1: User 1 sends message to User 4
    WebSocket.send({
        senderId: "user_1",
        receiverId: "user_4",
        message: "Hello!"
    })
    ↓
Step 2: Chat Service 1 receives message
    ↓
Step 3: Check Redis: Is User 4 online?
    connection = redis.get("ws:user:user_4")
    ↓
Step 4: User 4 is online (Chat Service 2)
    ↓
Step 5: Publish to Redis Stream:
    redis.xadd("chat:chat_service_2", message)
    ↓
Step 6: Chat Service 2 consumes from Redis Stream
    ↓
Step 7: Chat Service 2 sends to User 4 via WebSocket
    ↓
Step 8: User 4 receives message
    ↓
Step 9: User 4 sends acknowledgment: "delivered"
    ↓
Step 10: Chat Service 2 forwards acknowledgment to Chat Service 1
    ↓
Step 11: Chat Service 1 sends acknowledgment to User 1
    ↓
Step 12: User 1 shows ✓✓
```

---

### 8.3 User Sends Message (Recipient Offline)

```
Step 1: User 1 sends message to User 4
    ↓
Step 2: Chat Service 1 checks Redis: Is User 4 online?
    connection = redis.get("ws:user:user_4")
    ↓
Step 3: User 4 is offline (not found in Redis)
    ↓
Step 4: Publish to Redis Stream (offline channel):
    redis.xadd("offline_messages", message)
    ↓
Step 5: Message Service consumes from Redis Stream
    ↓
Step 6: Message Service persists to Cassandra:
    INSERT INTO messages (...)
    VALUES (..., delivery_status='sent')
    ↓
Step 7: Notification Service consumes from Redis Stream
    ↓
Step 8: Notification Service sends push notification:
    FCM (Android) or APNS (iOS)
    ↓
Step 9: User 4 receives notification
    ↓
Step 10: User 4 opens app → Connects WebSocket
    ↓
Step 11: Chat Service retrieves undelivered messages
    ↓
Step 12: Send messages to User 4
    ↓
Step 13: User 4 sends acknowledgment
    ↓
Step 14: Update delivery_status = 'delivered'
```

---

### 8.4 Group Message

```
Step 1: User 1 sends message to Group 456
    ↓
Step 2: Chat Service calls Group Service:
    "Get members of Group 456"
    ↓
Step 3: Group Service returns: [user_2, user_3, user_4, ...]
    ↓
Step 4: FOR EACH member:
    Check Redis: Is member online?
    IF online:
        Publish to Redis Stream (member's channel)
    ELSE:
        Publish to offline_messages
    ↓
Step 5: Each Chat Service sends to its connected users
    ↓
Step 6: Users receive message → Send acknowledgment
    ↓
Step 7: Update delivery_status per user
```

---

## 9. Interview Q&A

### Q1: Why WebSocket over HTTP?

**A:**
- ✅ **Bidirectional:** Send and receive simultaneously
- ✅ **Persistent:** No repeated connections
- ✅ **Low latency:** < 300ms
- ✅ **Efficient:** No polling overhead

**HTTP polling:**
- ❌ 100 requests/minute (wasteful)
- ❌ 1-second delay

### Q2: Why Redis Stream over Kafka?

**A:**

| Feature | Redis Stream | Kafka |
|---------|--------------|-------|
| Latency | < 10ms ✅ | 50-100ms |
| Use Case | Real-time chat | Event streaming |

**Decision:** Redis Stream (lower latency)

### Q3: Why Cassandra for messages?

**A:**
- ✅ **Write-heavy:** 100B messages/day
- ✅ **Scalable:** Horizontal scaling
- ✅ **High availability:** No SPOF

**MySQL:**
- ❌ Write bottleneck
- ❌ Difficult to scale

### Q4: How to handle 1 billion WebSocket connections?

**A:**
**Problem:** 1B connections = Too much memory!

**Solution:** **TTL + Auto-disconnect**

```
1. WebSocket TTL: 60 seconds
2. If inactive → Disconnect
3. User sends heartbeat every 30 seconds
4. If heartbeat received → Extend TTL
5. If no heartbeat → Disconnect

Result: Only active users have connections!
```

**Estimate:**
- 1B users, 10% active = 100M connections
- 100M connections × 10 KB/connection = 1 TB memory
- Distributed across 1,000 servers = 1 GB/server ✅

### Q5: How to prevent message loss?

**A:**
**Strategies:**

1. **Persist to Redis Stream first:**
   - Message stored before sending
   - If send fails → Retry from Redis

2. **Acknowledgments:**
   - Client sends "received" acknowledgment
   - If no acknowledgment → Retry

3. **Cassandra replication:**
   - Replication factor: 3
   - No single point of failure

### Q6: How to handle network disconnections?

**A:**
**Flow:**

```
1. User's network disconnects
2. WebSocket connection drops
3. Redis TTL expires (60 seconds)
4. User reconnects
5. WebSocket Gateway checks Redis
6. No existing connection → Create new one
7. Chat Service retrieves undelivered messages
8. Send to user
```

### Q7: How does group message delivery status work?

**A:**
**Challenge:** Show who read the message

**Solution:** **Individual delivery status per member**

**Schema:**
```sql
CREATE TABLE group_message_status (
    message_id UUID,
    user_id UUID,
    delivery_status VARCHAR(20),  -- sent, delivered, read
    PRIMARY KEY (message_id, user_id)
);
```

**Query:**
```sql
SELECT user_id, delivery_status
FROM group_message_status
WHERE message_id = 'msg_789'
```

**Result:**
```
user_2: read
user_3: delivered
user_4: sent
```

### Q8: How to optimize message storage?

**A:**
**WhatsApp approach:**

1. **Local storage:** Messages stored on device
2. **Server storage:** Only undelivered messages
3. **Auto-delete:** Once delivered → Delete from server

**Result:** 90% reduction in server storage!

### Q9: How to handle peak traffic (New Year)?

**A:**
**Strategies:**

1. **Horizontal scaling:** Add more Chat Services
2. **Redis Cluster:** Shard WebSocket Registry
3. **Cassandra sharding:** Partition by chat_id
4. **Rate limiting:** Max 100 messages/minute/user

### Q10: Security considerations?

**A:**
1. **End-to-End Encryption:** Encrypt messages (Signal Protocol)
2. **JWT Authentication:** Validate all WebSocket connections
3. **TLS/SSL:** Encrypt WebSocket traffic
4. **Rate Limiting:** Prevent spam
5. **Input Validation:** Prevent XSS attacks

---

## 10. Key Takeaways

✅ **WebSocket:** Bidirectional, persistent, low-latency  
✅ **Redis Stream:** Lower latency than Kafka (< 10ms)  
✅ **Cassandra:** Write-heavy, scalable  
✅ **WebSocket Registry:** Track user connections (Redis)  
✅ **Sticky connections:** User always connects to same Chat Service  
✅ **TTL:** Auto-disconnect inactive users (60 seconds)  
✅ **Offline messages:** Persist to Cassandra + Push notification  
✅ **Group messages:** Individual delivery status per member  
✅ **Media upload:** Separate HTTP upload to S3  
✅ **CDN:** Low-latency media delivery  

---

## Summary

**Architecture Highlights:**
- 8+ microservices
- 5 databases (PostgreSQL x2, Cassandra, Redis, Elasticsearch)
- WebSocket Gateway (sticky connections)
- Redis Stream (message routing)
- S3 + CDN (media storage)

**WebSocket Flow:**
- HTTP → Handshake → WebSocket upgrade
- Register in Redis (with TTL)
- Send/receive via Redis Stream
- Acknowledgments for delivery status

**Offline Handling:**
- Check Redis → Not found → Offline
- Persist to Cassandra
- Send push notification (FCM/APNS)
- Retrieve on reconnect

**Group Messaging:**
- Get members from Group Service
- Send individually to each member
- Track delivery status per member

**Performance:**
- Message delivery: < 300ms
- 100B messages/day
- 100M concurrent connections

**End of Lecture 8**
