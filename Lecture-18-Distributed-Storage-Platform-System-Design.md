# Lecture 18: Distributed Storage Platform System Design

> **Lecture:** 18  
> **Topic:** System Design  
> **Application:** Google Drive, Dropbox  
> **Scale:** Millions of users, billions of files  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-17-Payment-Gateway-System-Design|Lecture 17: Payment Gateway]]

---

## 1. Functional and Non-Functional Requirements

### 1.1 Functional Requirements

1. **User Onboarding** - Create account
2. **File Upload** - Upload files/folders/documents
3. **File Download** - Download from anywhere
4. **Auto-Sync** - All devices sync automatically
5. **File Sharing** - Share with permissions (read/write)
6. **Directory Operations** - Create, delete, rename folders
7. **Storage Quota** - 15GB free (like Google Drive)

**Out of Scope:** Collaborative editing (covered in Lecture 10)

---

### 1.2 Non-Functional Requirements

1. **Scale:** Millions of users, billions of files
2. **CAP Theorem:** **Highly Available (AP)** with eventual consistency for sync
   - Upload/download must always work
   - Sync can have small delay (eventual consistency)
   - **Exception:** Collaborative editing needs strong consistency
3. **Large Files:** Support multi-GB uploads
4. **Latency:** Minimal upload/sync latency
5. **Durability:** Zero data loss, highly reliable

---

## 2. Core Entities

1. **User** - Account holder
2. **File Metadata** - Name, size, type, permissions, version
3. **File/Folder** - Actual data
4. **Chunk** - File split into pieces (4-5MB each)
5. **Permission** - Access control (read/write)
6. **Version** - File version tracking

---

## 3. Key Concept: Folders Are Metadata!

**Critical Insight:** Google Drive doesn't create real directories. Everything is metadata!

```json
{
  "folderId": "folder_123",
  "name": "Photos",
  "type": "FOLDER",
  "parentId": "root",
  "children": [
    {
      "fileId": "file_456",
      "name": "vacation.jpg",
      "type": "FILE",
      "size": 2048000
    },
    {
      "folderId": "folder_789",
      "name": "2024",
      "type": "FOLDER",
      "children": [...]
    }
  ]
}
```

**Operations:**
- **Create folder:** Insert metadata with `type: FOLDER`
- **Rename:** Update `name` field
- **Move file:** Change `parentId`
- **Delete:** Remove metadata entry

---

## 4. API Design

### 4.1 Folder Management

```
POST /v1/folders
Body: {name: "Photos", parentId: "root", type: "FOLDER"}

GET /v1/folders/{folderId}
Response: {folderId, name, parentId, createdBy, createdAt}

GET /v1/folders/{folderId}/content
Response: {children: [...]}
```

---

### 4.2 File Upload (5-Step Process)

**Step 1: Initialize Upload**
```
POST /v1/files/upload/init
Request:
{
  "fileName": "video.mp4",
  "fileSize": 16777216,  // 16 MB
  "parentFolderId": "folder_123"
}

Response:
{
  "fileId": "file_abc",
  "uploadId": "upload_xyz",
  "chunkSize": 5242880,  // 5 MB
  "existingChunks": []  // Empty for fresh upload
}
```

**Step 2: Get Chunk Upload URLs**
```
POST /v1/files/upload/chunk-url
Request:
{
  "uploadId": "upload_xyz",
  "chunkId": 1,
  "chunkHash": "sha256_hash_of_chunk"
}

Response:
{
  "signedUrl": "https://s3.amazonaws.com/bucket/upload_xyz/chunk_1?signature=..."
}
```

**Step 3: Upload Chunks (Direct to S3)**
```
PUT {signedUrl}
Body: <binary chunk data>
```

**Step 4: Commit Upload**
```
POST /v1/files/upload/commit
Body: {uploadId: "upload_xyz"}
```

**Step 5: Resume Upload (if network fails)**
```
POST /v1/files/upload/init
Request:
{
  "fileName": "video.mp4",
  "fileSize": 16777216,
  "resumeUploadId": "upload_xyz"  // Previous upload ID
}

Response:
{
  "uploadId": "upload_xyz",
  "existingChunks": [1, 2, 3]  // Already uploaded
}
```

---

## 5. Client Components

```
┌─────────────────────────────────────┐
│          CLIENT (Desktop App)        │
├─────────────────────────────────────┤
│  1. Watcher Service                 │
│     - Monitors file changes          │
│                                      │
│  2. Chunker Service                 │
│     - Splits files into chunks       │
│     - Generates SHA-256 hash         │
│                                      │
│  3. Upload Manager                  │
│     - Manages upload flow            │
│     - Handles retries                │
│                                      │
│  4. Sync Engine                     │
│     - Keeps devices in sync          │
│                                      │
│  5. Local Metadata Index            │
│     - SQLite database                │
│     - Stores file versions locally   │
└─────────────────────────────────────┘
```

---

## 6. Complete Upload Flow

### 6.1 Flow Diagram

```
USER DROPS FILE (16 MB)
    ↓
WATCHER SERVICE detects change
    ↓
UPLOAD MANAGER → POST /v1/files/upload/init
    ↓
BACKEND:
  - Check storage quota (User DB)
  - Create upload session
  - Return: {uploadId, chunkSize: 5MB, fileId}
    ↓
CHUNKER SERVICE:
  - Split 16MB into 4 chunks (5MB, 5MB, 5MB, 1MB)
  - Generate SHA-256 for each chunk
    ↓
FOR EACH CHUNK:
  UPLOAD MANAGER → POST /v1/files/upload/chunk-url
  BACKEND → Return signed S3 URL
  CLIENT → PUT {signedUrl} (direct to S3)
    ↓
UPLOAD MANAGER → POST /v1/files/upload/commit
    ↓
BACKEND:
  - Validate all chunks (hash verification)
  - Save metadata to DB
  - Publish sync event to Kafka
    ↓
SYNC SERVICE:
  - Notify all connected devices
  - Trigger download on other devices
```

---

## 7. Architecture (Low-Level Design)

### 7.1 Complete System

```
┌─────────────────┐
│  CLIENT (App)   │
│  - Watcher      │
│  - Chunker      │
│  - Upload Mgr   │
│  - Sync Engine  │
│  - Local Index  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Load Balancer  │
└────────┬────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│  User  │ │ Upload │ │Metadata│ │  Sync  │
│Service │ │Service │ │Service │ │Service │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│User DB │ │ Redis  │ │Metadata│ │ Kafka  │
│(PG)    │ │ Cache  │ │  DB    │ │        │
└────────┘ └────────┘ └───┬────┘ └────────┘
                          │
         ┌────────────────┴─────────┐
         │                          │
    ┌────▼────┐              ┌──────▼──────┐
    │   S3    │              │  Validator  │
    │ Bucket  │              │  Service    │
    └─────────┘              └─────────────┘
```

---

### 7.2 Upload Service (Deep Dive)

**Redis Cache Schema:**
```
Key: upload:{uploadId}
Value: {
  "fileId": "file_abc",
  "fileName": "video.mp4",
  "totalChunks": 4,
  "uploadedChunks": [1, 2, 3],  // Bitmap
  "chunkHashes": {
    "1": "sha256_hash_1",
    "2": "sha256_hash_2",
    "3": "sha256_hash_3",
    "4": "sha256_hash_4"
  },
  "retryCount": 0
}
TTL: 86400  // 24 hours
```

**Why Redis?**
- ✅ Fast validation during upload
- ✅ Short-lived data (TTL)
- ✅ High read/write frequency

---

### 7.3 Metadata DB (PostgreSQL)

**Schema:**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255),
    storage_quota BIGINT DEFAULT 16106127360,  -- 15 GB
    storage_used BIGINT DEFAULT 0
);

CREATE TABLE folders (
    folder_id UUID PRIMARY KEY,
    parent_folder_id UUID,
    owner_id UUID,
    name VARCHAR(255),
    type VARCHAR(10),  -- 'FOLDER'
    created_at TIMESTAMP
);

CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    parent_folder_id UUID,
    owner_id UUID,
    name VARCHAR(255),
    size BIGINT,
    version INT DEFAULT 1,
    created_at TIMESTAMP
);

CREATE TABLE chunks (
    chunk_id UUID PRIMARY KEY,
    file_id UUID,
    chunk_index INT,
    s3_path VARCHAR(500),
    chunk_size BIGINT,
    checksum VARCHAR(64),  -- SHA-256
    version INT
);

CREATE TABLE file_chunks (
    file_id UUID,
    chunk_id UUID,
    chunk_index INT,
    version_id INT,
    PRIMARY KEY (file_id, chunk_index, version_id)
);

CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY,
    resource_type VARCHAR(10),  -- 'FILE' or 'FOLDER'
    resource_id UUID,
    user_id UUID,
    role VARCHAR(10),  -- 'READ', 'WRITE', 'OWNER'
    created_at TIMESTAMP
);
```

---

## 8. Chunk Validation

**Problem:** How to ensure chunks uploaded correctly?

**Solution:** Hash verification

**Client Side (Chunking):**
```java
void chunkFile(File file) {
    int chunkSize = 5 * 1024 * 1024;  // 5 MB
    int chunkIndex = 0;
    
    while (offset < file.length()) {
        byte[] chunkData = readChunk(file, offset, chunkSize);
        
        // Generate hash
        String hash = SHA256(chunkData);
        
        // Store locally
        chunkMetadata.put(chunkIndex, new Chunk(chunkIndex, hash, chunkData));
        
        offset += chunkSize;
        chunkIndex++;
    }
}
```

**Server Side (Validation):**
```java
void validateUpload(String uploadId) {
    // Get expected hashes from Redis
    Map<Integer, String> expectedHashes = redis.get("upload:" + uploadId).getChunkHashes();
    
    // Read actual chunks from S3
    for (int i = 0; i < expectedHashes.size(); i++) {
        byte[] chunkData = s3.getObject("upload_" + uploadId + "/chunk_" + i);
        String actualHash = SHA256(chunkData);
        
        if (!actualHash.equals(expectedHashes.get(i))) {
            // Mismatch! Request re-upload
            requestRetry(uploadId, i);
        }
    }
}
```

---

## 9. Sync Mechanism

### 9.1 Two Approaches

**1. Pull (Refresh)**
```
Client (every 30s):
    POST /v1/sync/check
    Body: {files: [{fileId: "abc", version: 1}, ...]}
    
Server:
    - Compare with Metadata DB
    - Return outdated files
    
Response:
    {outdatedFiles: [{fileId: "abc", latestVersion: 2, downloadUrl: "..."}]}
```

**2. Push (Fan-out)**
```
User uploads file
    ↓
Upload Service → Kafka (sync_event topic)
    ↓
Sync Service consumes event
    ↓
Notify all connected clients via WebSocket
    ↓
Clients download new version
```

---

### 9.2 Local Metadata Index (SQLite)

```sql
CREATE TABLE local_files (
    file_id VARCHAR(100),
    file_name VARCHAR(255),
    version INT,
    last_modified TIMESTAMP,
    local_path VARCHAR(500),
    sync_status VARCHAR(20)  -- 'SYNCED', 'PENDING', 'CONFLICT'
);
```

**Usage:**
```java
void checkSync() {
    List<LocalFile> localFiles = sqlite.getAllFiles();
    
    SyncRequest req = new SyncRequest();
    req.setFiles(localFiles);
    
    SyncResponse resp = syncService.check(req);
    
    for (OutdatedFile file : resp.getOutdatedFiles()) {
        downloadManager.download(file.getDownloadUrl());
        sqlite.updateVersion(file.getFileId(), file.getLatestVersion());
    }
}
```

---

## 10. File Versioning

**Scenario:** User edits `document.txt` (16 MB)

**Initial Upload (Version 1):**
```
Chunk 1 (5 MB) → hash_1
Chunk 2 (5 MB) → hash_2
Chunk 3 (5 MB) → hash_3
Chunk 4 (1 MB) → hash_4
```

**User edits file (changes only first 5 MB):**

**Smart Upload (Version 2):**
```
Chunker compares with local metadata:
  Chunk 1: hash_1_new (CHANGED) → Upload ✅
  Chunk 2: hash_2 (SAME) → Skip ❌
  Chunk 3: hash_3 (SAME) → Skip ❌
  Chunk 4: hash_4 (SAME) → Skip ❌
  
Only upload Chunk 1!
```

**Database:**
```sql
INSERT INTO file_chunks VALUES 
  ('file_abc', 'chunk_new', 0, 2),  -- Version 2, Chunk 0
  ('file_abc', 'chunk_2', 1, 2),    -- Version 2, Chunk 1 (reused)
  ('file_abc', 'chunk_3', 2, 2),    -- Version 2, Chunk 2 (reused)
  ('file_abc', 'chunk_4', 3, 2);    -- Version 2, Chunk 3 (reused)
```

---

## 11. Deduplication

**Problem:** Two users upload same file → waste storage

**Solution:** Content-based addressing (hash)

**Flow:**
```
User A uploads cat.jpg (10 MB)
    ↓
Chunker: Chunk 1 → hash_abc
    ↓
Upload Service checks S3:
    Key: chunks/hash_abc
    EXISTS? ✅
    ↓
Skip upload, reference existing chunk!
```

**Database:**
```sql
-- User A
INSERT INTO file_chunks VALUES ('file_userA', 'chunk_hash_abc', 0, 1);

-- User B (same file)
INSERT INTO file_chunks VALUES ('file_userB', 'chunk_hash_abc', 0, 1);

-- Only ONE chunk in S3!
s3://bucket/chunks/hash_abc
```

**Storage Savings:**
```
Without dedup: 2 users × 10 MB = 20 MB
With dedup:    1 chunk = 10 MB
Savings:       50%
```

---

## 12. Optimization: Validator Service

**Problem:** Validating millions of chunks during upload is expensive

**Wrong Approach:**
```
For each chunk uploaded:
    Validate hash ❌ (millions of validations!)
```

**Correct Approach:**
```
Upload all chunks
    ↓
User sends COMMIT
    ↓
Validator Service (async):
    - Reads all chunks from S3
    - Validates hashes
    - Updates metadata DB
    
User sees: "Processing..." (few seconds)
```

---

## 13. Additional Services (Async)

```
┌─────────────────────────────────────┐
│         S3 BUCKET                    │
└────────┬───────┬───────┬───────┬────┘
         │       │       │       │
    ┌────▼──┐ ┌──▼───┐ ┌▼────┐ ┌▼────────┐
    │Virus  │ │Thumb │ │De-  │ │Validator│
    │Scanner│ │Gen   │ │dup  │ │Service  │
    └───────┘ └──────┘ └─────┘ └─────────┘
```

All run async post-upload!

---

## 14. Interview Q&A

### Q1: Why use signed URLs instead of uploading via backend?

**A:**

**Via Backend:**
```
Client → Load Balancer → Upload Service → S3
  |         |              |
  ▼         ▼              ▼
16 MB     16 MB          16 MB
```
**Bandwidth:** 48 MB total!

**Signed URL:**
```
Client → S3
  |
  ▼
16 MB only
```
**Savings:** 66%!

### Q2: Why chunk files?

**A:**

1. **Resume uploads:** Network fails → resume from last chunk
2. **Parallel uploads:** 3 chunks simultaneously
3. **Deduplication:** Reuse identical chunks
4. **Versioning:** Only upload changed chunks

### Q3: Redis vs Database for upload metadata?

**A:**

| Feature | Redis | PostgreSQL |
|---------|-------|------------|
| **Latency** | < 1ms | 10-50ms |
| **Durability** | No (TTL) | Yes |
| **Use Case** | Temp upload state | Permanent metadata |

**During upload:** Redis (fast, temporary)  
**After commit:** PostgreSQL (durable, permanent)

### Q4: How to handle storage quota?

**A:**

```java
void checkQuota(String userId, long fileSize) {
    User user = db.getUser(userId);
    
    long available = user.getStorageQuota() - user.getStorageUsed();
    
    if (fileSize > available) {
        throw new QuotaExceededException();
    }
    
    // Reserve space
    user.setStorageUsed(user.getStorageUsed() + fileSize);
    db.update(user);
}
```

### Q5: Sync latency?

**A:**

**Push (WebSocket):** < 1 sec  
**Pull (Polling):** 30-60 sec  

**Hybrid:** Push for online clients, Pull as fallback

### Q6: Conflict resolution?

**A:**

**Scenario:** User edits same file on 2 devices offline

**Solution:**
```
Device A: file_v2_deviceA.txt
Device B: file_v2_deviceB.txt

User manually resolves conflict
```

(Detailed solution in Lecture 10: Collaborative Editor)

### Q7: Scale to millions of users?

**A:**

| Component | Scaling Strategy |
|-----------|------------------|
| **Upload Service** | Horizontal scaling (stateless) |
| **Redis** | Cluster mode (partitioning) |
| **S3** | Infinite (managed by AWS) |
| **Metadata DB** | Sharding by user_id |

**Calculation:**
```
1M users × 10 GB = 10 PB storage
S3: $23/TB/month = $230,000/month
```

### Q8: Security?

**A:**

1. **Signed URLs:** Expire in 1 hour
2. **Permission checks:** Before generating download URL
3. **Encryption at rest:** S3 server-side encryption
4. **Virus scanning:** Async post-upload

---

## 15. Key Takeaways

✅ **Folders = Metadata:** No real directories, just JSON  
✅ **Chunking:** 4-5 MB chunks for large files  
✅ **Signed URLs:** Direct upload to S3 (bypass backend)  
✅ **Hash Validation:** SHA-256 for integrity  
✅ **Redis Cache:** Temporary upload state  
✅ **Versioning:** Only upload changed chunks  
✅ **Deduplication:** Content-based addressing  
✅ **Sync:** Push (WebSocket) + Pull (polling)  
✅ **Local Index:** SQLite for client-side metadata  
✅ **Async Jobs:** Validation, dedup, virus scan  

---

## Summary

**Upload Flow:**
```
1. Init: Check quota, return uploadId + chunkSize
2. Chunk: Split file, generate hashes
3. URLs: Get signed URLs for each chunk
4. Upload: Direct to S3 (parallel)
5. Commit: Validate hashes, save metadata
6. Sync: Notify all devices via Kafka
```

**Architecture:**
- 4 core services (User, Upload, Metadata, Sync)
- Redis for temp state, PostgreSQL for metadata
- S3 for blob storage
- Kafka for sync events
- Async validators

**Performance:**
- Millions of users
- Chunked uploads (resume support)
- Deduplication (50% savings)
- Eventual consistency for sync

**End of Lecture 18**
