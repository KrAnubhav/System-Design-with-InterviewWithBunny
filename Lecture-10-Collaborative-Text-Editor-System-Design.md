# Lecture 10: Collaborative Text Editor System Design (Google Docs)

> **Lecture:** 10  
> **Topic:** System Design  
> **Application:** Collaborative Text Editor (Google Docs, Notion, Confluence)  
> **Scale:** Millions of Users, Billions of Documents  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-09-Distributed-Job-Scheduler-System-Design|Lecture 9: Distributed Job Scheduler]]

---

## 1. Brief Introduction

### What is a Collaborative Text Editor?

A **collaborative text editor** allows multiple users to simultaneously edit the same document in real-time, seeing each other's changes instantly.

**Examples:** Google Docs, Notion, Confluence, Microsoft Office 365, Dropbox Paper

### Problem It Solves

1. **Real-Time Collaboration:** Multiple users edit simultaneously
2. **Conflict Resolution:** Handle concurrent edits gracefully
3. **Version Control:** Track document history
4. **Presence Awareness:** See who's editing
5. **Cursor Tracking:** View other users' cursor positions

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Document Management**
   - Create, read, update, delete documents

2. **Collaborative Editing**
   - Multiple users edit same document simultaneously
   - Single user can edit alone

3. **Real-Time Updates**
   - See other users' changes in near real-time (< 100ms)

4. **Presence & Cursor**
   - Show who's online
   - Display cursor positions

5. **Version Control**
   - Save document versions
   - Retrieve previous versions

### 2.2 Non-Functional Requirements

1. **Scale**
   - **Millions of users**
   - **Billions of documents**

2. **CAP Theorem**
   - **Offline Mode:** Highly Available (AP) - Eventual consistency
   - **Collaborative Mode:** Highly Consistent (CP) - Real-time sync
   
   **Why?**
   - ✅ Offline: Delay acceptable (1-2 seconds)
   - ✅ Collaborative: Consistency critical (< 100ms)

3. **Low Latency**
   - **< 100ms** for edit propagation between users

---

## 3. Core Entities

1. **User** - Person editing the document
2. **Document** - Text file being edited
3. **Edit** - Change operation (insert, delete)
4. **Cursor** - User's cursor position

---

## 4. API Design

### 4.1 Create Document

```
POST /v1/documents
```

**Request Body:**
```json
{
  "title": "Project Proposal",
  "content": ""
}
```

**Response:**
```json
{
  "documentId": "doc_123",
  "url": "https://docs.example.com/doc_123"
}
```

---

### 4.2 Get Document (Read-Only)

```
GET /v1/documents/{documentId}
```

**Response:**
```json
{
  "documentId": "doc_123",
  "title": "Project Proposal",
  "content": "Lorem ipsum...",
  "version": "1.0",
  "createdBy": "user_456",
  "createdAt": "2026-01-21T12:00:00Z"
}
```

---

### 4.3 Edit Document (WebSocket!)

```
WS /v1/documents/{documentId}/edit
```

⚠️ **Important:** This is a **WebSocket** connection, not HTTP!

**Edit Event:**
```json
{
  "action": "insert",
  "position": 29,
  "character": "R",
  "userId": "user_123",
  "timestamp": "2026-01-21T12:00:00.123Z"
}
```

**Delete Event:**
```json
{
  "action": "delete",
  "position": 15,
  "length": 5,
  "userId": "user_123",
  "timestamp": "2026-01-21T12:00:00.456Z"
}
```

---

### 4.4 Get Document Versions

```
GET /v1/documents/{documentId}/versions
```

**Response:**
```json
{
  "versions": [
    {
      "versionId": "v1.0",
      "createdBy": "user_123",
      "createdAt": "2026-01-21T12:00:00Z",
      "url": "s3://docs/doc_123_v1.0"
    },
    {
      "versionId": "v1.1",
      "createdBy": "user_456",
      "createdAt": "2026-01-21T13:00:00Z",
      "url": "s3://docs/doc_123_v1.1"
    }
  ]
}
```

---

### 4.5 Get Specific Version

```
GET /v1/documents/{documentId}/versions/{versionId}
```

**Response:**
```json
{
  "documentId": "doc_123",
  "versionId": "v1.0",
  "content": "Original content...",
  "createdAt": "2026-01-21T12:00:00Z"
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

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
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Document │    │ Document │    │   S3     │
  │ Metadata │    │  Editor  │    │  Bucket  │
  │ Service  │    │ Service  │    │          │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │ Metadata │    │Operation │
  │    DB    │    │    DB    │
  │(Cassandra)│   │(Cassandra)│
  └──────────┘    └──────────┘
```

### 5.2 Service Responsibilities

**Document Metadata Service:**
- Create, update, delete document metadata
- Store title, owner, timestamps

**Document Editor Service:**
- Handle WebSocket connections
- Apply Operational Transformation (OT)
- Manage canonical copy in Redis
- Auto-save every 10-20 seconds

**S3 Bucket:**
- Store actual document files
- Store document versions

**CDN:**
- Cache read-only documents
- Low-latency delivery

---

## 6. Low-Level Design (LLD)

### 6.1 Why WebSocket? (Not HTTP!)

**Three Approaches Compared:**

---

#### **Approach 1: File Replacement (❌ Naive)**

**Flow:**
```
User A types "ABC"
    ↓
POST /documents/{id} (entire file)
    ↓
Server saves file
    ↓
SSE notification to User B
    ↓
User B fetches entire file
```

**Problems:**
- ❌ **Large payload:** Send entire file on every keystroke
- ❌ **Too many requests:** Fetch entire file for every change
- ❌ **High latency:** > 1 second

---

#### **Approach 2: Locking Protocol (❌ Not Collaborative)**

**Flow:**
```
User A acquires lock
    ↓
User A edits document
    ↓
User B waits...
    ↓
User A releases lock
    ↓
User B acquires lock
```

**Types:**
- **Pessimistic Lock:** Banking (ACID)
- **Optimistic Lock:** Git merge

**Problem:**
- ❌ **Not simultaneous:** Only one user edits at a time

---

#### **Approach 3: WebSocket + Delta Sync (✅ Optimal)**

**Flow:**
```
User A types "R" at position 29
    ↓
WebSocket: {action: "insert", position: 29, char: "R"}
    ↓
Server applies OT
    ↓
WebSocket: Broadcast to all users
    ↓
User B sees change in < 100ms
```

**Why it works:**
- ✅ **Small payload:** Only send delta (change)
- ✅ **Bidirectional:** Send and receive simultaneously
- ✅ **Low latency:** < 100ms
- ✅ **Persistent connection:** No repeated handshakes

---

### 6.2 Conflict Resolution Problem

**Scenario:**

```
Initial state: "BC"

User Alice: Insert "A" at position 0
    ↓
Alice's view: "ABC"

User Bob: Insert "D" at position 2
    ↓
Bob's view: "BCD"
```

**Naive Reconciliation:**

```
Alice receives: "Insert D at position 2"
    ↓
Alice's view: "ABDC" ❌

Bob receives: "Insert A at position 0"
    ↓
Bob's view: "ABCD" ✅

INCONSISTENT! ❌
```

**Problem:** Blindly applying operations causes inconsistency

---

### 6.3 Solution 1: Operational Transformation (OT)

**Algorithm:**

**OT transforms concurrent operations during merging. The transformation function adjusts the effect of an operation to account for changes made by other operations.**

---

#### **How OT Works:**

**Initial state:** "BC"

**Alice:** Insert "A" at position 0 → "ABC"  
**Bob:** Insert "D" at position 2 → "BCD"

---

**Step 1: Bob's operation reaches Alice**

**Naive approach:**
```
Insert "D" at position 2
    ↓
"ABC" → "ABDC" ❌
```

**OT approach:**
```
OT detects: Alice inserted at position 0
    ↓
Transform Bob's operation: position 2 → position 3
    ↓
Insert "D" at position 3
    ↓
"ABC" → "ABCD" ✅
```

---

**Step 2: Alice's operation reaches Bob**

**Naive approach:**
```
Insert "A" at position 0
    ↓
"BCD" → "ABCD" ✅
```

**Result:** Both users see "ABCD" ✅

---

#### **OT Transformation Function:**

```python
def transform(op1, op2):
    """
    Transform op1 based on op2
    """
    if op1.action == "insert" and op2.action == "insert":
        if op2.position <= op1.position:
            # op2 inserted before op1, shift op1 right
            op1.position += 1
    
    elif op1.action == "delete" and op2.action == "insert":
        if op2.position <= op1.position:
            # op2 inserted before op1, shift op1 right
            op1.position += 1
    
    elif op1.action == "insert" and op2.action == "delete":
        if op2.position < op1.position:
            # op2 deleted before op1, shift op1 left
            op1.position -= op2.length
    
    return op1
```

---

#### **OT Limitations:**

1. **Single Server Requirement:**
   - All operations must go through **same server**
   - Requires sticky WebSocket connections

2. **Complex Implementation:**
   - Must handle all operation combinations
   - Edge cases are tricky

---

### 6.4 Solution 2: CRDT (Conflict-Free Replicated Data Type)

**Algorithm:**

**CRDT is a data structure designed for distributed systems that automatically resolves conflicts caused by concurrent updates.**

---

#### **How CRDT Works:**

**Key Idea:** Assign unique, sortable positions to each character

**Initial state:**
```
B → position 0.5
C → position 0.75
```

---

**Alice:** Insert "A" before "B"
```
A → position 0.25
B → position 0.5
C → position 0.75
```

**Bob:** Insert "D" after "C"
```
B → position 0.5
C → position 0.75
D → position 0.875
```

---

**Reconciliation (both sides):**

**Sort by position:**
```
0.25 → A
0.5  → B
0.75 → C
0.875 → D

Result: "ABCD" ✅
```

**Both users see "ABCD"** (consistent!)

---

#### **CRDT Advantages:**

✅ **No central server:** Decentralized  
✅ **Automatic conflict resolution:** No transformation needed  
✅ **Simpler implementation:** Just sort by position  

#### **CRDT Disadvantages:**

❌ **Position explosion:** Positions grow (0.25 → 0.125 → 0.0625...)  
❌ **Tombstones:** Deleted characters still tracked  
❌ **Memory overhead:** More metadata per character  

---

### 6.5 Google Docs Approach: Operational Transformation (OT)

**Why Google chose OT:**
- ✅ **Lower memory:** No position metadata
- ✅ **Proven at scale:** Used since 2010
- ✅ **Sticky connections:** Already needed for WebSocket

---

### 6.6 Document Editor Service (Detailed)

#### **Components:**

```
┌─────────────────────────────────────────────────────────┐
│           Document Editor Service                        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  WebSocket   │  │     OT       │  │   Redis      │  │
│  │   Handler    │  │   Engine     │  │  (Canonical  │  │
│  │              │  │              │  │    Copy)     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

#### **Flow:**

**Step 1: User opens document for editing**

```
User requests edit access
    ↓
HTTP GET /documents/{id}/edit
    ↓
Upgrade to WebSocket
    ↓
Server fetches document from S3
    ↓
Server stores canonical copy in Redis
    ↓
Server sends document to client
```

---

**Step 2: User makes edit**

```
User types "R" at position 29
    ↓
Client sends via WebSocket:
    {action: "insert", position: 29, char: "R"}
    ↓
Server receives operation
    ↓
Server does TWO things:
    1. Persist operation to Operation DB (via Kafka)
    2. Apply OT to canonical copy (Redis)
    ↓
Server broadcasts to all connected users
```

---

**Step 3: Other users receive edit**

```
User B receives operation via WebSocket
    ↓
Client applies OT locally
    ↓
User B sees change in < 100ms
```

---

### 6.7 Database Schemas

#### **Metadata DB (Cassandra)**

```sql
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    title TEXT,
    blob_url TEXT,  -- S3 URL
    created_by UUID,
    created_at TIMESTAMP,
    last_modified_by UUID,
    last_modified_at TIMESTAMP
);
```

---

#### **Operation DB (Cassandra)**

```sql
CREATE TABLE operations (
    document_id UUID,
    timestamp TIMESTAMP,
    user_id UUID,
    action TEXT,  -- insert, delete
    position INT,
    character TEXT,
    data JSONB,
    PRIMARY KEY (document_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp ASC);
```

**Why store operations?**
- ✅ **Audit trail:** Who changed what
- ✅ **Replay:** Reconstruct document from operations
- ✅ **Conflict resolution:** OT needs operation history

---

#### **Version DB (Cassandra)**

```sql
CREATE TABLE versions (
    id UUID PRIMARY KEY,
    document_id UUID,
    version_id TEXT,  -- "v1.0", "v1.1"
    created_by UUID,
    created_at TIMESTAMP,
    blob_url TEXT  -- S3 URL for this version
);

CREATE INDEX idx_document_id ON versions(document_id);
```

---

### 6.8 Auto-Save & Versioning

**Challenge:** Save document without disrupting editing

**Solution:** Auto-save every 10-20 seconds

---

#### **Auto-Save Flow:**

```
Every 10-20 seconds:
    ↓
Document Editor Service
    ↓
Fetch canonical copy from Redis
    ↓
Save to S3 (new version)
    ↓
Publish event to Kafka: "version_created"
    ↓
Metadata Consumer updates Metadata DB
```

---

#### **Versioning Strategy:**

**During active editing:**
```
v1.0 (initial)
    ↓
v1.0.1 (auto-save after 20s)
    ↓
v1.0.2 (auto-save after 40s)
    ↓
v1.0.3 (auto-save after 60s)
```

**After session ends:**
```
Reconciliation Service runs:
    ↓
Fetch v1.0 from S3
    ↓
Fetch all operations from Operation DB
    ↓
Apply operations to v1.0
    ↓
Save final version: v1.1
    ↓
Delete intermediate versions (v1.0.1, v1.0.2, v1.0.3)
    ↓
Delete operations from Operation DB
```

**Result:** Only major versions stored (v1.0, v1.1, v1.2...)

---

### 6.9 Reconciliation Service

**Purpose:** Optimize storage by consolidating versions

**Flow:**

```python
def reconcile_document(document_id):
    # Triggered when no active editors
    
    # 1. Fetch initial version
    initial_version = s3.get(f"{document_id}_v1.0")
    
    # 2. Fetch all operations
    operations = db.query("""
        SELECT * FROM operations
        WHERE document_id = ?
        ORDER BY timestamp ASC
    """, document_id)
    
    # 3. Apply operations
    final_content = initial_version
    for op in operations:
        final_content = apply_operation(final_content, op)
    
    # 4. Save final version
    s3.put(f"{document_id}_v1.1", final_content)
    
    # 5. Delete intermediate versions
    s3.delete(f"{document_id}_v1.0.1")
    s3.delete(f"{document_id}_v1.0.2")
    s3.delete(f"{document_id}_v1.0.3")
    
    # 6. Delete operations
    db.execute("""
        DELETE FROM operations
        WHERE document_id = ?
    """, document_id)
    
    # 7. Notify Metadata DB
    kafka.publish("version_created", {
        "document_id": document_id,
        "version_id": "v1.1"
    })
```

---

### 6.10 Cursor Position Tracking

**Challenge:** Show where other users are typing

**Solution:** Redis cache with TTL

---

#### **Flow:**

```
User A moves cursor to position 42
    ↓
Client sends via WebSocket:
    {action: "cursor_move", position: 42}
    ↓
Server stores in Redis:
    SET cursor:doc_123:user_A 42 EX 60
    ↓
Server broadcasts to all users:
    {userId: "user_A", position: 42}
    ↓
User B sees User A's cursor at position 42
```

**Why Redis?**
- ✅ **Fast:** < 1ms lookup
- ✅ **TTL:** Auto-expire after 60 seconds (user left)

---

### 6.11 Presence Awareness

**Challenge:** Show who's editing

**Solution:** WebSocket Registry (Redis)

---

#### **Flow:**

```
User A opens document for editing
    ↓
WebSocket connection established
    ↓
Server stores in Redis:
    SADD editors:doc_123 "user_A"
    EXPIRE editors:doc_123 60
    ↓
Server broadcasts to all users:
    {action: "user_joined", userId: "user_A"}
    ↓
User B sees "User A is editing"
```

**When user leaves:**
```
WebSocket disconnects
    ↓
Server removes from Redis:
    SREM editors:doc_123 "user_A"
    ↓
Server broadcasts:
    {action: "user_left", userId: "user_A"}
```

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
│         (OT on Client Side)                              │
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
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Document │    │ Document │    │   S3     │
  │ Metadata │    │  Editor  │    │  Bucket  │
  │ Service  │    │ Service  │    │          │
  └────┬─────┘    │ (OT)     │    └──────────┘
       │          └────┬─────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │  Kafka   │    │  Redis   │
  │          │    │(Canonical│
  └────┬─────┘    │  Copy)   │
       │          └──────────┘
       ▼
  ┌──────────┐    ┌──────────┐
  │ Metadata │    │Operation │
  │ Consumer │    │ Consumer │
  └────┬─────┘    └────┬─────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │ Metadata │    │Operation │
  │    DB    │    │    DB    │
  │(Cassandra)│   │(Cassandra)│
  └──────────┘    └──────────┘
       │
       │
       ▼
  ┌──────────┐
  │Version DB│
  │(Cassandra)│
  └──────────┘


  ┌──────────────────────────┐
  │  Reconciliation Service  │
  │  (Runs after session)    │
  └──────────────────────────┘
```

---

## 8. Complete Edit Flow

### 8.1 User Opens Document for Editing

```
Step 1: User clicks "Edit"
    ↓
Step 2: HTTP GET /documents/{id}/edit
    ↓
Step 3: Upgrade to WebSocket (handshake)
    ↓
Step 4: Server fetches document from S3
    ↓
Step 5: Server stores canonical copy in Redis
    SET doc:doc_123 "document content" EX 3600
    ↓
Step 6: Server sends document to client
    ↓
Step 7: Server registers user in WebSocket Registry
    SADD editors:doc_123 "user_A"
    ↓
Step 8: Server broadcasts to all users:
    {action: "user_joined", userId: "user_A"}
```

---

### 8.2 User Makes Edit

```
Step 1: User types "R" at position 29
    ↓
Step 2: Client sends via WebSocket:
    {
      action: "insert",
      position: 29,
      character: "R",
      userId: "user_A",
      timestamp: "2026-01-21T12:00:00.123Z"
    }
    ↓
Step 3: Server receives operation
    ↓
Step 4: Server publishes to Kafka (persistence):
    kafka.publish("operations", operation)
    ↓
Step 5: Server applies OT to canonical copy (Redis):
    current_doc = redis.get("doc:doc_123")
    updated_doc = apply_operation(current_doc, operation)
    redis.set("doc:doc_123", updated_doc)
    ↓
Step 6: Server broadcasts to all connected users (except sender):
    websocket.broadcast(operation)
    ↓
Step 7: User B receives operation
    ↓
Step 8: User B applies OT locally:
    transformed_op = transform(operation, local_operations)
    apply_to_ui(transformed_op)
    ↓
Step 9: User B sees change in < 100ms
```

---

### 8.3 Auto-Save (Every 10-20 Seconds)

```
Step 1: Timer triggers (20 seconds elapsed)
    ↓
Step 2: Server fetches canonical copy from Redis:
    doc_content = redis.get("doc:doc_123")
    ↓
Step 3: Server generates version ID:
    version_id = "v1.0.1"
    ↓
Step 4: Server saves to S3:
    s3.put("doc_123_v1.0.1", doc_content)
    ↓
Step 5: Server publishes to Kafka:
    kafka.publish("version_created", {
      document_id: "doc_123",
      version_id: "v1.0.1",
      blob_url: "s3://docs/doc_123_v1.0.1"
    })
    ↓
Step 6: Metadata Consumer updates Version DB
```

---

### 8.4 User Closes Document

```
Step 1: User closes tab
    ↓
Step 2: WebSocket disconnects
    ↓
Step 3: Server removes from WebSocket Registry:
    SREM editors:doc_123 "user_A"
    ↓
Step 4: Server broadcasts:
    {action: "user_left", userId: "user_A"}
    ↓
Step 5: IF no more editors:
    Trigger Reconciliation Service
```

---

### 8.5 Reconciliation (After Session Ends)

```
Step 1: Reconciliation Service detects no active editors
    ↓
Step 2: Fetch initial version from S3:
    initial = s3.get("doc_123_v1.0")
    ↓
Step 3: Fetch all operations from Operation DB:
    operations = db.query("SELECT * FROM operations WHERE document_id = 'doc_123'")
    ↓
Step 4: Apply operations sequentially:
    final = initial
    for op in operations:
        final = apply_operation(final, op)
    ↓
Step 5: Save final version to S3:
    s3.put("doc_123_v1.1", final)
    ↓
Step 6: Delete intermediate versions:
    s3.delete("doc_123_v1.0.1")
    s3.delete("doc_123_v1.0.2")
    s3.delete("doc_123_v1.0.3")
    ↓
Step 7: Delete operations from Operation DB:
    db.execute("DELETE FROM operations WHERE document_id = 'doc_123'")
    ↓
Step 8: Update Metadata DB:
    kafka.publish("version_created", {version_id: "v1.1"})
```

---

## 9. Interview Q&A

### Q1: Why WebSocket over HTTP?

**A:**

**HTTP (Polling):**
```
User types → POST /edit
User fetches → GET /document (every 1 second)
    ↓
100 requests/minute ❌
```

**WebSocket:**
```
User types → WebSocket.send()
User receives → WebSocket.onMessage()
    ↓
1 persistent connection ✅
```

**Benefits:**
- ✅ **Bidirectional:** Send and receive simultaneously
- ✅ **Low latency:** < 100ms
- ✅ **Efficient:** No polling overhead

### Q2: OT vs CRDT - Which to choose?

**A:**

| Feature | OT | CRDT |
|---------|----|----|
| **Conflict Resolution** | Transform operations | Sort by position |
| **Server Requirement** | Central server needed | Decentralized |
| **Memory** | Low | High (position metadata) |
| **Complexity** | High (transformation logic) | Low (just sort) |
| **Use Case** | Google Docs | Figma, Notion |

**Decision:**
- **OT:** If you have sticky WebSocket connections (Google Docs)
- **CRDT:** If you need offline-first (Figma, Notion)

### Q3: How to handle network disconnections?

**A:**

**Flow:**

```
User's network disconnects
    ↓
WebSocket connection drops
    ↓
Client buffers operations locally
    ↓
User reconnects
    ↓
Client sends buffered operations
    ↓
Server applies OT to reconcile
```

**Implementation:**
```javascript
// Client-side
let operationBuffer = [];

websocket.onclose = () => {
    // Buffer operations while offline
    document.addEventListener('input', (e) => {
        operationBuffer.push({
            action: 'insert',
            position: e.target.selectionStart,
            character: e.data
        });
    });
};

websocket.onopen = () => {
    // Send buffered operations
    operationBuffer.forEach(op => {
        websocket.send(JSON.stringify(op));
    });
    operationBuffer = [];
};
```

### Q4: Why store operations in database?

**A:**

**Three Reasons:**

1. **Audit Trail:**
   - Who changed what, when
   - Compliance requirements

2. **Replay:**
   - Reconstruct document from operations
   - Useful for debugging

3. **Reconciliation:**
   - Consolidate versions after session
   - Apply operations to initial version

### Q5: How to handle large documents?

**A:**

**Strategies:**

1. **Chunking:**
   - Split document into pages
   - Load only visible pages

2. **Lazy Loading:**
   - Load content as user scrolls
   - Reduce initial payload

3. **Compression:**
   - Gzip document content
   - Reduce network transfer

4. **Pagination:**
   - Limit operations per page
   - Reduce OT complexity

### Q6: Why Redis for canonical copy?

**A:**

**Requirements:**
- ✅ **Fast read/write:** < 1ms (for OT)
- ✅ **In-memory:** Low latency
- ✅ **TTL support:** Auto-expire inactive documents

**Why not database?**
- ❌ **Slow:** 10-50ms per query
- ❌ **Disk-based:** High latency

### Q7: How to prevent data loss during auto-save?

**A:**

**Strategy: Dual Write**

```
Auto-save triggers
    ↓
Write to Redis (canonical copy)
    ↓
Write to S3 (persistent storage)
    ↓
IF S3 write fails:
    Retry with exponential backoff
    ↓
IF Redis fails:
    Fallback to S3 for canonical copy
```

### Q8: How to handle version explosion?

**A:**

**Problem:**
```
Auto-save every 20 seconds
    ↓
1 hour session = 180 versions ❌
```

**Solution: Reconciliation Service**

```
After session ends:
    ↓
Keep only major versions (v1.0, v1.1, v1.2)
    ↓
Delete intermediate versions (v1.0.1, v1.0.2, ...)
    ↓
Result: 3 versions instead of 180 ✅
```

### Q9: How to scale WebSocket connections?

**A:**

**Strategies:**

1. **Sticky Sessions:**
   - Route same document to same server
   - Use consistent hashing

2. **Horizontal Scaling:**
   - Add more WebSocket servers
   - Use Redis for shared state

3. **Connection Pooling:**
   - Reuse WebSocket connections
   - Reduce handshake overhead

### Q10: Security considerations?

**A:**

1. **Authentication:**
   - JWT token in WebSocket handshake
   - Validate on every operation

2. **Authorization:**
   - Check edit permissions
   - Prevent unauthorized edits

3. **Rate Limiting:**
   - Max 100 operations/second/user
   - Prevent spam

4. **Input Validation:**
   - Sanitize operations
   - Prevent XSS attacks

---

## 10. Key Takeaways

✅ **WebSocket:** Bidirectional, persistent, low-latency (< 100ms)  
✅ **OT (Operational Transformation):** Transform operations to resolve conflicts  
✅ **CRDT:** Conflict-free data structure (alternative to OT)  
✅ **Delta Sync:** Send only changes, not entire file  
✅ **Canonical Copy:** Redis stores current state (in-memory)  
✅ **Auto-Save:** Every 10-20 seconds to S3  
✅ **Versioning:** Major versions only (reconciliation service)  
✅ **Operation DB:** Store all edits for audit and replay  
✅ **Cursor Tracking:** Redis with TTL  
✅ **Presence Awareness:** WebSocket Registry (Redis)  

---

## Summary

**Architecture Highlights:**
- 6+ microservices
- 3 databases (Cassandra x3: Metadata, Operations, Versions)
- Redis (canonical copy, cursors, presence)
- S3 + CDN (document storage)
- WebSocket Gateway (sticky connections)

**OT Algorithm:**
- Transform operations based on concurrent changes
- Requires central server
- Used by Google Docs

**CRDT Algorithm:**
- Assign sortable positions to characters
- Decentralized
- Used by Figma, Notion

**Edit Flow:**
```
User types → WebSocket → OT (server) → Redis (canonical) → Broadcast → OT (client) → UI update
```

**Versioning:**
- Auto-save every 10-20 seconds (intermediate versions)
- Reconciliation service (consolidate to major versions)
- Delete operations after reconciliation

**Performance:**
- < 100ms edit propagation
- Millions of users
- Billions of documents

**End of Lecture 10**
