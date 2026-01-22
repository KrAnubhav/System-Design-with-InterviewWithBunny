# Lecture 18: Distributed Storage Platform System Design

> **Lecture:** 18  
> **Topic:** System Design  
> **Application:** Google Drive, Dropbox, OneDrive  
> **Scale:** Millions of Users, Billions of Files  
> **Difficulty:** Very Hard (Complex Client + Backend Coordination)  
> **Previous:** [[Lecture-17-Payment-Gateway-System-Design|Lecture 17: Payment Gateway System Design]]

---

## 📋 Table of Contents

1. [Introduction](#1-introduction)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Core Entities](#4-core-entities)
5. [Critical Concept: Folders Are Metadata](#5-critical-concept-folders-are-metadata)
6. [API Design](#6-api-design)
7. [High-Level Design](#7-high-level-design)
8. [Client Architecture (Deep Dive)](#8-client-architecture-deep-dive)
9. [Upload Flow (6-Step Process)](#9-upload-flow-6-step-process)
10. [Metadata Database Schema](#10-metadata-database-schema)
11. [Sync Mechanism](#11-sync-mechanism)
12. [File Versioning & Delta Upload](#12-file-versioning--delta-upload)
13. [Download & Sharing](#13-download--sharing)
14. [Optimizations & Follow-ups](#14-optimizations--follow-ups)
15. [Interview Q&A](#15-interview-qa)

---

<br>

## 1. Introduction

Designing a **Distributed Cloud Storage Platform** like Google Drive or Dropbox is one of the most complex system design problems because:

1. **Client Complexity:** Unlike typical web apps, significant logic resides in the **client application** (desktop/mobile).
2. **Synchronization:** Real-time sync across multiple devices with conflict resolution.
3. **Scale:** Billions of files, petabytes of storage.
4. **Reliability:** Zero data loss is non-negotiable.

### What We're Building

A cloud platform where users can:
- Upload files, folders, documents (any size)
- Download from anywhere
- **Auto-sync** across all connected devices
- Share with permissions (read/write)
- Create directory structures

---

<br>

## 2. Functional Requirements

| # | Requirement | Description |
|---|-------------|-------------|
| **1** | **User Onboarding** | Create account, sign up/sign in |
| **2** | **File Upload** | Upload files, folders, documents of any size |
| **3** | **File Download** | Download files from anywhere in the world |
| **4** | **Auto-Sync** | All devices linked to same account sync automatically |
| **5** | **File Sharing** | Share files/folders with permissions (read/write) |
| **6** | **Directory Management** | Create, delete, rename folders (CRUD operations) |
| **7** | **Storage Quota** | 15GB free tier (like Google Drive), enforce limits |

### Out of Scope

❌ **Collaborative Editing:** Real-time document editing (covered in Lecture 10: Google Docs)  
❌ **Version History UI:** Showing all previous versions in UI  
❌ **Trash/Recycle Bin:** Soft delete functionality

---

<br>

## 3. Non-Functional Requirements

### 3.1 Scale

- **Users:** Millions of users
- **Files:** Billions of files
- **Storage:** Petabytes of data

### 3.2 CAP Theorem

**For File Upload/Sync:**
- **Highly Available (AP):** System must always accept uploads
- **Eventual Consistency:** Sync can have small delay (few seconds acceptable)

**For Metadata/Permissions:**
- **Highly Consistent (CP):** Permission changes must be immediate
- **ACID Compliance:** File structure integrity

**Why AP for Sync?**
```
Scenario: User uploads photo on Device A
- Device A uploads immediately ✅
- Device B syncs after 2-3 seconds ✅ (Acceptable!)
- Better than: System down, can't upload ❌
```

### 3.3 Large File Support

- Support files of **any size** (2GB, 10GB, 100GB)
- Only limited by user's storage quota

### 3.4 Latency

1. **Upload Latency:** Limited by network bandwidth, minimize system overhead
2. **Sync Latency:** Should feel "real-time" to user (< 5 seconds)

### 3.5 Reliability & Durability

- **Zero Data Loss:** 99.999999999% durability (11 nines)
- **Replication:** Multiple copies across data centers
- **Disaster Recovery:** Files survive server crashes

---

<br>

## 4. Core Entities

| Entity | Description |
|--------|-------------|
| **User** | Account holder |
| **File** | Actual binary data (blob) |
| **Metadata** | File information (name, size, type, path, owner) |
| **Folder** | Virtual grouping (metadata only, not real directory) |
| **Chunk** | File piece (4-5MB blocks) |
| **Version** | Tracks file changes over time |
| **Permission** | Access control (read/write) |
| **Device/Client** | Physical device syncing data |

---

<br>

## 5. Critical Concept: Folders Are Metadata!

🔥 **KEY INSIGHT:** Google Drive does NOT create real directories. Everything is metadata!

### 5.1 How It Works

When you "create a folder," you're just inserting a JSON object in the database:

```json
{
  "folder_id": "folder_123",
  "name": "Vacation Photos",
  "type": "FOLDER",
  "parent_id": "root",
  "owner_id": "user_456",
  "created_at": "2026-01-21T10:00:00Z",
  "children": [
    {
      "file_id": "file_789",
      "name": "beach.jpg",
      "type": "FILE",
      "size": 2048000
    },
    {
      "folder_id": "folder_999",
      "name": "2024",
      "type": "FOLDER",
      "children": []
    }
  ]
}
```

### 5.2 Operations Are Metadata Changes

| Operation | What Actually Happens |
|-----------|----------------------|
| **Create Folder** | Insert metadata with `type: "FOLDER"` |
| **Rename File** | Update `name` field in metadata |
| **Move File** | Change `parent_id` in metadata |
| **Delete File** | Set `is_deleted: true` (soft delete) |
| **Copy File** | Create new metadata entry, reference same chunks |

**Example: Moving a File**

```
UI: Drag "beach.jpg" from "Vacation Photos" to "2024" folder

Backend:
UPDATE files 
SET parent_folder_id = 'folder_999' 
WHERE file_id = 'file_789';

No actual file movement! Just metadata update.
```

---

<br>

## 6. API Design

### 6.1 Folder Management APIs

**1. Create Folder**
```http
POST /v1/folders
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Vacation Photos",
  "parent_folder_id": "root",
  "type": "FOLDER"
}

Response:
{
  "folder_id": "folder_123",
  "name": "Vacation Photos",
  "created_at": "2026-01-21T10:00:00Z"
}
```

**2. Get Folder Metadata**
```http
GET /v1/folders/{folder_id}

Response:
{
  "folder_id": "folder_123",
  "name": "Vacation Photos",
  "parent_folder_id": "root",
  "owner_id": "user_456",
  "created_at": "2026-01-21T10:00:00Z",
  "permissions": ["READ", "WRITE"]
}
```

**3. Get Folder Content (List Files)**
```http
GET /v1/folders/{folder_id}/content

Response:
{
  "folder_id": "folder_123",
  "children": [
    {
      "file_id": "file_789",
      "name": "beach.jpg",
      "type": "FILE",
      "size": 2048000,
      "modified_at": "2026-01-21T10:30:00Z"
    },
    {
      "folder_id": "folder_999",
      "name": "2024",
      "type": "FOLDER"
    }
  ]
}
```

**4. Rename Folder**
```http
PUT /v1/folders/{folder_id}

{
  "name": "Summer Vacation 2024"
}
```

**5. Delete Folder**
```http
DELETE /v1/folders/{folder_id}
```

---

### 6.2 File Upload APIs (Complex 6-Step Flow)

**Step 1: Initialize Upload**
```http
POST /v1/files/upload/init
Content-Type: application/json

Request:
{
  "filename": "movie.mp4",
  "size": 16777216,  // 16 MB
  "parent_folder_id": "folder_123",
  "checksum": "sha256_final_file_hash"
}

Response:
{
  "file_id": "file_abc123",
  "upload_id": "upload_xyz789",
  "chunk_size": 5242880,  // 5 MB
  "existing_chunks": []  // Empty for fresh upload
}
```

**Step 1b: Resume Upload (After Network Failure)**
```http
POST /v1/files/upload/init

Request:
{
  "filename": "movie.mp4",
  "size": 16777216,
  "parent_folder_id": "folder_123",
  "resume_upload_id": "upload_xyz789"  // Previous upload ID
}

Response:
{
  "file_id": "file_abc123",
  "upload_id": "upload_xyz789",
  "chunk_size": 5242880,
  "existing_chunks": [0, 1, 2]  // Already uploaded, skip these!
}
```

**Step 2: Get Signed URL for Each Chunk**
```http
POST /v1/files/upload/chunk-url

Request:
{
  "upload_id": "upload_xyz789",
  "chunk_index": 0,
  "chunk_hash": "sha256_chunk_0_hash"
}

Response:
{
  "signed_url": "https://s3.amazonaws.com/bucket/upload_xyz789/chunk_0?signature=..."
}
```

**Step 3: Upload Chunk Directly to S3**
```http
PUT https://s3.amazonaws.com/bucket/upload_xyz789/chunk_0?signature=...
Content-Type: application/octet-stream

<binary chunk data>

Response: 200 OK
```

**Step 4: Commit Upload**
```http
POST /v1/files/upload/commit

{
  "upload_id": "upload_xyz789"
}

Response:
{
  "file_id": "file_abc123",
  "status": "PROCESSING",  // Validation in progress
  "message": "File uploaded successfully. Processing..."
}
```

---

### 6.3 File Download APIs

**1. Get Download URL**
```http
GET /v1/files/{file_id}/download

Response:
{
  "download_url": "https://s3.amazonaws.com/bucket/file_abc123?signature=...",
  "expires_at": "2026-01-21T11:00:00Z"  // 1 hour
}
```

---

### 6.4 Sync APIs

**1. Check for Updates (Pull)**
```http
POST /v1/sync/check

Request:
{
  "device_id": "device_123",
  "files": [
    {"file_id": "file_abc", "version": 1},
    {"file_id": "file_def", "version": 2}
  ]
}

Response:
{
  "outdated_files": [
    {
      "file_id": "file_abc",
      "latest_version": 2,
      "download_url": "https://s3.amazonaws.com/..."
    }
  ],
  "new_files": [
    {
      "file_id": "file_ghi",
      "name": "new_photo.jpg",
      "download_url": "https://s3.amazonaws.com/..."
    }
  ]
}
```

---

<br>

## 7. High-Level Design

### 7.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│              CLIENT (Desktop/Mobile App)                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │  Watcher   │  │  Chunker   │  │   Upload   │        │
│  │  Service   │  │  Service   │  │  Manager   │        │
│  └────────────┘  └────────────┘  └────────────┘        │
│  ┌────────────┐  ┌────────────┐                        │
│  │   Sync     │  │   Local    │                        │
│  │  Engine    │  │  Metadata  │                        │
│  │            │  │   Index    │                        │
│  └────────────┘  └────────────┘                        │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
    ┌─────────────────┐
    │  Load Balancer  │
    │   API Gateway   │
    └────────┬────────┘
             │
    ┌────────┴────────┬──────────────┬──────────────┐
    ▼                 ▼              ▼              ▼
┌────────┐      ┌────────┐    ┌────────┐    ┌────────┐
│  User  │      │ Upload │    │Metadata│    │  Sync  │
│Service │      │Service │    │Service │    │Service │
└───┬────┘      └───┬────┘    └───┬────┘    └───┬────┘
    │               │              │              │
    ▼               ▼              ▼              ▼
┌────────┐      ┌────────┐    ┌────────┐    ┌────────┐
│User DB │      │ Redis  │    │Metadata│    │ Kafka  │
│  (PG)  │      │ Cache  │    │   DB   │    │        │
└────────┘      └────────┘    └───┬────┘    └────────┘
                                  │
                 ┌────────────────┴────────────┐
                 │                             │
            ┌────▼────┐                  ┌─────▼──────┐
            │   S3    │                  │ Validator  │
            │ Bucket  │                  │  Service   │
            │ (Blob)  │                  │ (Async)    │
            └─────────┘                  └────────────┘
```

### 7.2 Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| **User Service** | Authentication, authorization, quota management |
| **Upload Service** | Orchestrate upload, generate signed URLs, validate quota |
| **Metadata Service** | Manage file/folder structure, permissions, versioning |
| **Sync Service** | Notify devices of changes (Push/Pull hybrid) |
| **Validator Service** | Async hash validation after upload |
| **Redis Cache** | Store upload session state (chunk bitmap, hashes) |
| **S3/Blob Storage** | Store actual file chunks |
| **Kafka** | Event streaming for sync notifications |

---

<br>

## 8. Client Architecture (Deep Dive)

The client is NOT a dumb terminal. It's a sophisticated engine with multiple components.

### 8.1 Client Components

```
┌─────────────────────────────────────────────────────────┐
│                  CLIENT APPLICATION                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. WATCHER SERVICE                                      │
│     - Monitors OS file system hooks                      │
│     - Detects file changes (create, modify, delete)      │
│     - Triggers upload/sync on changes                    │
│                                                          │
│  2. CHUNKER SERVICE                                      │
│     - Splits large files into 4-5MB chunks               │
│     - Generates SHA-256 hash for each chunk              │
│     - Enables resume, deduplication, delta sync          │
│                                                          │
│  3. UPLOAD MANAGER                                       │
│     - Orchestrates 6-step upload flow                    │
│     - Handles retries on network failure                 │
│     - Uploads 3 chunks in parallel                       │
│                                                          │
│  4. SYNC ENGINE                                          │
│     - Polls server for updates (Pull)                    │
│     - Receives WebSocket notifications (Push)            │
│     - Downloads new/updated files                        │
│                                                          │
│  5. LOCAL METADATA INDEX (SQLite)                        │
│     - Stores file metadata locally                       │
│     - Tracks versions, sync status                       │
│     - Enables offline detection                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 8.2 Watcher Service (OS File System Monitoring)

**How It Works:**

```java
public class WatcherService {
    
    // Monitors local file system using OS hooks
    // Linux: inotify, Mac: FSEvents, Windows: ReadDirectoryChangesW
    
    public void onFileChange(File file) {
        if (file.isDirectory()) return;
        
        // 1. Check Local Metadata Index
        LocalFileRecord record = localDb.get(file.getPath());
        
        // 2. Compute hash of current file
        String newHash = SHA256.hash(file);
        
        // 3. Deduplication check
        if (record != null && record.getHash().equals(newHash)) {
            // False alarm - timestamp changed but content didn't
            return;
        }
        
        // 4. Trigger upload
        uploadManager.addToQueue(new UploadTask(file));
    }
}
```

### 8.3 Chunker Service

**Purpose:** Split large files into manageable pieces

**Code Example:**

```java
public class ChunkerService {
    
    private static final int CHUNK_SIZE = 5 * 1024 * 1024; // 5 MB
    
    public List<Chunk> chunkFile(File file) {
        List<Chunk> chunks = new ArrayList<>();
        
        try (FileInputStream fis = new FileInputStream(file)) {
            byte[] buffer = new byte[CHUNK_SIZE];
            int chunkIndex = 0;
            int bytesRead;
            
            while ((bytesRead = fis.read(buffer)) != -1) {
                // Create chunk
                byte[] chunkData = Arrays.copyOf(buffer, bytesRead);
                
                // Generate SHA-256 hash
                String hash = SHA256.hash(chunkData);
                
                // Create chunk object
                Chunk chunk = new Chunk(chunkIndex, hash, chunkData);
                chunks.add(chunk);
                
                chunkIndex++;
            }
        }
        
        return chunks;
    }
}
```

**Why Chunking?**

1. **Resume Uploads:** Network fails → resume from last chunk
2. **Parallel Uploads:** Upload 3 chunks simultaneously
3. **Deduplication:** Reuse identical chunks across users
4. **Delta Sync:** Only upload changed chunks on file edit

### 8.4 Local Metadata Index (SQLite)

**Schema:**

```sql
CREATE TABLE local_files (
    file_id VARCHAR(100) PRIMARY KEY,
    file_path VARCHAR(500),
    file_name VARCHAR(255),
    file_hash VARCHAR(64),  -- SHA-256 of entire file
    version INT,
    size BIGINT,
    last_modified TIMESTAMP,
    sync_status VARCHAR(20),  -- 'SYNCED', 'PENDING', 'SYNCING', 'CONFLICT'
    created_at TIMESTAMP
);

CREATE TABLE local_chunks (
    chunk_id VARCHAR(100) PRIMARY KEY,
    file_id VARCHAR(100),
    chunk_index INT,
    chunk_hash VARCHAR(64),  -- SHA-256 of chunk
    size INT,
    FOREIGN KEY (file_id) REFERENCES local_files(file_id)
);
```

**Usage:**

```java
public class LocalMetadataIndex {
    
    private Connection sqliteConn;
    
    public void updateFileStatus(String fileId, String status) {
        String sql = "UPDATE local_files SET sync_status = ? WHERE file_id = ?";
        PreparedStatement stmt = sqliteConn.prepareStatement(sql);
        stmt.setString(1, status);
        stmt.setString(2, fileId);
        stmt.executeUpdate();
    }
    
    public List<LocalFile> getOutdatedFiles() {
        String sql = "SELECT * FROM local_files WHERE sync_status != 'SYNCED'";
        // Execute query and return results
    }
}
```

---

<br>

## 9. Upload Flow (6-Step Process)

This is the **most critical flow** to understand!

### 9.1 Complete Flow Diagram

```
USER DROPS FILE (16 MB) INTO DROPBOX FOLDER
    ↓
WATCHER SERVICE detects file change
    ↓
Check LOCAL METADATA INDEX
    - Is this file already uploaded?
    - Is this a new version?
    ↓
UPLOAD MANAGER initiates upload
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 1: INITIALIZE UPLOAD                                │
└─────────────────────────────────────────────────────────┘
    │
    ▼
POST /v1/files/upload/init
Request: {filename, size, parent_folder_id}
    ↓
UPLOAD SERVICE:
  1. Check User Quota (User DB)
     - Available: 15GB - 10GB = 5GB
     - File Size: 16MB
     - OK? ✅
  2. Generate upload_id, file_id
  3. Return chunk_size (5MB)
    ↓
Response: {file_id, upload_id, chunk_size: 5MB, existing_chunks: []}
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 2: CHUNK FILE                                       │
└─────────────────────────────────────────────────────────┘
    │
    ▼
CHUNKER SERVICE:
  - Split 16MB into chunks:
    * Chunk 0: 5MB → hash_0
    * Chunk 1: 5MB → hash_1
    * Chunk 2: 5MB → hash_2
    * Chunk 3: 1MB → hash_3
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 3: GET SIGNED URLs                                  │
└─────────────────────────────────────────────────────────┘
    │
    ▼
FOR EACH CHUNK (0, 1, 2, 3):
    POST /v1/files/upload/chunk-url
    Request: {upload_id, chunk_index, chunk_hash}
        ↓
    UPLOAD SERVICE:
      1. Validate upload_id (Redis)
      2. Store chunk metadata in Redis:
         Key: upload:upload_xyz789
         Value: {
           chunk_bitmap: [0, 1, 2, 3],
           chunk_hashes: {
             "0": "hash_0",
             "1": "hash_1",
             "2": "hash_2",
             "3": "hash_3"
           }
         }
      3. Generate S3 pre-signed URL
        ↓
    Response: {signed_url: "https://s3.amazonaws.com/..."}
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 4: UPLOAD CHUNKS (DIRECT TO S3)                     │
└─────────────────────────────────────────────────────────┘
    │
    ▼
UPLOAD MANAGER:
  - Upload 3 chunks in parallel
  - PUT {signed_url_0} <chunk_0_data>
  - PUT {signed_url_1} <chunk_1_data>
  - PUT {signed_url_2} <chunk_2_data>
  - PUT {signed_url_3} <chunk_3_data>
    ↓
S3 BUCKET:
  - Stores chunks:
    * s3://bucket/upload_xyz789/chunk_0
    * s3://bucket/upload_xyz789/chunk_1
    * s3://bucket/upload_xyz789/chunk_2
    * s3://bucket/upload_xyz789/chunk_3
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 5: COMMIT UPLOAD                                    │
└─────────────────────────────────────────────────────────┘
    │
    ▼
POST /v1/files/upload/commit
Request: {upload_id}
    ↓
UPLOAD SERVICE:
  1. Mark upload as "PROCESSING"
  2. Trigger VALIDATOR SERVICE (async)
    ↓
┌─────────────────────────────────────────────────────────┐
│ STEP 6: VALIDATION & METADATA PERSISTENCE (ASYNC)        │
└─────────────────────────────────────────────────────────┘
    │
    ▼
VALIDATOR SERVICE:
  1. Read chunks from S3
  2. Compute SHA-256 for each chunk
  3. Compare with expected hashes (Redis)
  4. If match ✅:
     - Save metadata to Metadata DB
     - Publish sync event to Kafka
  5. If mismatch ❌:
     - Request re-upload of corrupted chunks
    ↓
METADATA SERVICE:
  - INSERT INTO files (file_id, name, size, version, ...)
  - INSERT INTO chunks (chunk_id, file_id, chunk_index, s3_path, checksum, ...)
    ↓
KAFKA:
  - Publish event: {event: "FILE_CREATED", file_id, user_id}
    ↓
SYNC SERVICE:
  - Notify all connected devices via WebSocket
    ↓
OTHER DEVICES:
  - Receive notification
  - Download new file
```

### 9.2 Step-by-Step Breakdown

#### **Step 1: Initialize Upload**

**Client Request:**
```json
POST /v1/files/upload/init

{
  "filename": "vacation_video.mp4",
  "size": 16777216,  // 16 MB
  "parent_folder_id": "folder_123"
}
```

**Server Logic:**

```java
public InitUploadResponse initUpload(InitUploadRequest req) {
    // 1. Check quota
    User user = userDb.getUser(req.getUserId());
    long available = user.getStorageQuota() - user.getStorageUsed();
    
    if (req.getSize() > available) {
        throw new QuotaExceededException("Upgrade to premium!");
    }
    
    // 2. Generate IDs
    String fileId = UUID.randomUUID().toString();
    String uploadId = UUID.randomUUID().toString();
    
    // 3. Return response
    return new InitUploadResponse(
        fileId,
        uploadId,
        CHUNK_SIZE,  // 5 MB
        new ArrayList<>()  // existing_chunks (empty for fresh upload)
    );
}
```

**Server Response:**
```json
{
  "file_id": "file_abc123",
  "upload_id": "upload_xyz789",
  "chunk_size": 5242880,  // 5 MB
  "existing_chunks": []
}
```

---

#### **Step 2: Chunk File**

**Client Logic:**

```java
List<Chunk> chunks = chunkerService.chunkFile(file);

// Result:
// Chunk 0: 5MB, hash: "a1b2c3..."
// Chunk 1: 5MB, hash: "d4e5f6..."
// Chunk 2: 5MB, hash: "g7h8i9..."
// Chunk 3: 1MB, hash: "j0k1l2..."
```

---

#### **Step 3: Get Signed URLs**

**Client Request (for each chunk):**
```json
POST /v1/files/upload/chunk-url

{
  "upload_id": "upload_xyz789",
  "chunk_index": 0,
  "chunk_hash": "a1b2c3..."
}
```

**Server Logic:**

```java
public ChunkUrlResponse getChunkUrl(ChunkUrlRequest req) {
    // 1. Validate upload_id
    String uploadId = req.getUploadId();
    if (!redis.exists("upload:" + uploadId)) {
        throw new InvalidUploadException();
    }
    
    // 2. Store chunk metadata in Redis
    redis.hset("upload:" + uploadId, "chunk_" + req.getChunkIndex(), req.getChunkHash());
    
    // 3. Generate S3 pre-signed URL
    String s3Key = uploadId + "/chunk_" + req.getChunkIndex();
    String signedUrl = s3Client.generatePresignedUrl(
        BUCKET_NAME,
        s3Key,
        Date.from(Instant.now().plus(1, ChronoUnit.HOURS)),
        HttpMethod.PUT
    );
    
    return new ChunkUrlResponse(signedUrl);
}
```

**Redis State:**
```
Key: upload:upload_xyz789
Value: {
  "chunk_0": "a1b2c3...",
  "chunk_1": "d4e5f6...",
  "chunk_2": "g7h8i9...",
  "chunk_3": "j0k1l2..."
}
TTL: 86400  // 24 hours
```

**Server Response:**
```json
{
  "signed_url": "https://s3.amazonaws.com/bucket/upload_xyz789/chunk_0?AWSAccessKeyId=...&Signature=...&Expires=..."
}
```

---

#### **Step 4: Upload Chunks**

**Client Logic:**

```java
// Upload 3 chunks in parallel
ExecutorService executor = Executors.newFixedThreadPool(3);

for (Chunk chunk : chunks) {
    executor.submit(() -> {
        String signedUrl = getSignedUrl(uploadId, chunk.getIndex(), chunk.getHash());
        uploadChunkToS3(signedUrl, chunk.getData());
    });
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.HOURS);
```

**HTTP Request:**
```http
PUT https://s3.amazonaws.com/bucket/upload_xyz789/chunk_0?signature=...
Content-Type: application/octet-stream

<binary chunk data>
```

---

#### **Step 5: Commit Upload**

**Client Request:**
```json
POST /v1/files/upload/commit

{
  "upload_id": "upload_xyz789"
}
```

**Server Logic:**

```java
public CommitResponse commitUpload(CommitRequest req) {
    // 1. Mark as processing
    String uploadId = req.getUploadId();
    redis.set("upload:" + uploadId + ":status", "PROCESSING");
    
    // 2. Trigger async validation
    validatorService.validateUploadAsync(uploadId);
    
    return new CommitResponse("PROCESSING", "File uploaded. Validating...");
}
```

---

#### **Step 6: Validation & Persistence (Async)**

**Validator Service Logic:**

```java
public void validateUploadAsync(String uploadId) {
    // 1. Get expected hashes from Redis
    Map<String, String> expectedHashes = redis.hgetall("upload:" + uploadId);
    
    // 2. Read chunks from S3 and validate
    for (int i = 0; i < expectedHashes.size(); i++) {
        String s3Key = uploadId + "/chunk_" + i;
        byte[] chunkData = s3Client.getObject(BUCKET_NAME, s3Key);
        
        String actualHash = SHA256.hash(chunkData);
        String expectedHash = expectedHashes.get("chunk_" + i);
        
        if (!actualHash.equals(expectedHash)) {
            // Corruption detected!
            requestRetry(uploadId, i);
            return;
        }
    }
    
    // 3. All chunks valid - save metadata
    metadataService.saveFileMetadata(uploadId);
    
    // 4. Publish sync event
    kafka.publish("sync_events", new FileCreatedEvent(uploadId));
    
    // 5. Cleanup Redis
    redis.del("upload:" + uploadId);
}
```

---

### 9.3 Resume Upload (Network Failure Handling)

**Scenario:** User uploads 20MB file (4 chunks), network fails after chunk 2

**Client Logic:**

```java
// On network failure, client stores upload_id locally
localDb.saveUploadSession(uploadId, fileId, uploadedChunks: [0, 1, 2]);

// When network recovers, resume upload
POST /v1/files/upload/init
{
  "filename": "vacation_video.mp4",
  "size": 20971520,
  "resume_upload_id": "upload_xyz789"  // Previous upload ID
}

// Server checks Redis, finds existing chunks
Response:
{
  "upload_id": "upload_xyz789",
  "chunk_size": 5242880,
  "existing_chunks": [0, 1, 2]  // Already uploaded
}

// Client only uploads chunk 3!
```

---

<br>

## 10. Metadata Database Schema

### 10.1 Database Choice

**PostgreSQL** for metadata (ACID compliance, strong consistency)

### 10.2 Schema

**1. Users Table**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    storage_quota BIGINT DEFAULT 16106127360,  -- 15 GB in bytes
    storage_used BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**2. Folders Table**
```sql
CREATE TABLE folders (
    folder_id UUID PRIMARY KEY,
    parent_folder_id UUID,  -- NULL for root
    owner_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(10) DEFAULT 'FOLDER',
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_folder_id) REFERENCES folders(folder_id)
);

CREATE INDEX idx_folders_owner ON folders(owner_id);
CREATE INDEX idx_folders_parent ON folders(parent_folder_id);
```

**3. Files Table**
```sql
CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    parent_folder_id UUID,
    owner_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    size BIGINT NOT NULL,
    version INT DEFAULT 1,
    checksum VARCHAR(64),  -- SHA-256 of entire file
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_folder_id) REFERENCES folders(folder_id)
);

CREATE INDEX idx_files_owner ON files(owner_id);
CREATE INDEX idx_files_parent ON files(parent_folder_id);
```

**4. Chunks Table**
```sql
CREATE TABLE chunks (
    chunk_id UUID PRIMARY KEY,
    chunk_hash VARCHAR(64) UNIQUE NOT NULL,  -- SHA-256 (for deduplication)
    s3_bucket VARCHAR(100),
    s3_key VARCHAR(500),
    size INT NOT NULL,
    ref_count INT DEFAULT 1,  -- For garbage collection
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_chunks_hash ON chunks(chunk_hash);
```

**5. File_Chunks Mapping Table**
```sql
CREATE TABLE file_chunks (
    file_id UUID,
    chunk_id UUID,
    chunk_index INT,
    version_id INT,
    PRIMARY KEY (file_id, chunk_index, version_id),
    FOREIGN KEY (file_id) REFERENCES files(file_id),
    FOREIGN KEY (chunk_id) REFERENCES chunks(chunk_id)
);

CREATE INDEX idx_file_chunks_file ON file_chunks(file_id);
```

**6. Permissions Table**
```sql
CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY,
    resource_type VARCHAR(10),  -- 'FILE' or 'FOLDER'
    resource_id UUID NOT NULL,
    user_id UUID NOT NULL,
    role VARCHAR(10),  -- 'READ', 'WRITE', 'OWNER'
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_permissions_resource ON permissions(resource_id);
CREATE INDEX idx_permissions_user ON permissions(user_id);
```

---

<br>

## 11. Sync Mechanism

### 11.1 Two Approaches: Push vs Pull

**1. Pull (Polling)**
```
Client polls server every 30 seconds:
    POST /v1/sync/check
    {files: [{file_id, version}, ...]}
    
Server compares with Metadata DB:
    - Outdated files
    - New files
    
Response:
    {outdated_files: [...], new_files: [...]}
    
Client downloads updates

Latency: 30-60 seconds
Pros: Simple, works offline
Cons: Inefficient, delayed
```

**2. Push (WebSocket/Long Polling)**
```
User uploads file on Device A
    ↓
Upload Service → Kafka (sync_events topic)
    ↓
Sync Service consumes event
    ↓
Sync Service → WebSocket → All connected devices
    ↓
Devices download new file

Latency: < 1 second
Pros: Real-time, efficient
Cons: Requires persistent connection
```

**3. Hybrid (Best of Both Worlds)**
```
- Push for online devices (WebSocket)
- Pull as fallback for offline/disconnected devices
```

### 11.2 Sync Flow Diagram

```
DEVICE A: User uploads "photo.jpg"
    ↓
Upload Service saves metadata
    ↓
Publish to Kafka:
    {
      "event": "FILE_CREATED",
      "file_id": "file_123",
      "user_id": "user_456",
      "file_name": "photo.jpg"
    }
    ↓
Sync Service consumes event
    ↓
┌─────────────────────────────────────────┐
│ Check which devices belong to user_456  │
│ - Device A (uploader) - Skip            │
│ - Device B (iPad) - Online ✅           │
│ - Device C (Phone) - Offline ❌         │
└─────────────────────────────────────────┘
    ↓
WebSocket → Device B:
    {
      "event": "NEW_FILE",
      "file_id": "file_123",
      "download_url": "https://s3.amazonaws.com/..."
    }
    ↓
Device B downloads file
    ↓
Device C (when comes online):
    - Polls: POST /v1/sync/check
    - Discovers new file
    - Downloads
```

### 11.3 Sync Engine (Client-Side)

```java
public class SyncEngine {
    
    private WebSocketClient wsClient;
    private ScheduledExecutorService pollExecutor;
    
    public void start() {
        // 1. Connect to WebSocket for push notifications
        wsClient.connect("wss://api.drive.com/sync");
        wsClient.onMessage(this::handlePushNotification);
        
        // 2. Start polling as fallback (every 30s)
        pollExecutor.scheduleAtFixedRate(
            this::pollForUpdates,
            0, 30, TimeUnit.SECONDS
        );
    }
    
    private void handlePushNotification(String message) {
        SyncEvent event = JSON.parse(message);
        
        if (event.getType().equals("NEW_FILE")) {
            downloadFile(event.getFileId(), event.getDownloadUrl());
        } else if (event.getType().equals("FILE_UPDATED")) {
            downloadFile(event.getFileId(), event.getDownloadUrl());
        } else if (event.getType().equals("FILE_DELETED")) {
            deleteLocalFile(event.getFileId());
        }
    }
    
    private void pollForUpdates() {
        List<LocalFile> localFiles = localDb.getAllFiles();
        
        SyncCheckRequest req = new SyncCheckRequest();
        req.setFiles(localFiles.stream()
            .map(f -> new FileVersion(f.getFileId(), f.getVersion()))
            .collect(Collectors.toList()));
        
        SyncCheckResponse resp = syncService.check(req);
        
        // Download outdated files
        for (OutdatedFile file : resp.getOutdatedFiles()) {
            downloadFile(file.getFileId(), file.getDownloadUrl());
            localDb.updateVersion(file.getFileId(), file.getLatestVersion());
        }
        
        // Download new files
        for (NewFile file : resp.getNewFiles()) {
            downloadFile(file.getFileId(), file.getDownloadUrl());
            localDb.insert(file);
        }
    }
}
```

---

<br>

## 12. File Versioning & Delta Upload

### 12.1 Scenario: User Edits File

**Initial Upload (Version 1):**
```
File: document.pdf (16 MB)
Chunks:
  - Chunk 0 (5 MB) → hash_a1
  - Chunk 1 (5 MB) → hash_b2
  - Chunk 2 (5 MB) → hash_c3
  - Chunk 3 (1 MB) → hash_d4
```

**User edits first page (changes first 5 MB):**

**Smart Upload (Version 2):**
```
Chunker re-chunks file:
  - Chunk 0 (5 MB) → hash_a1_NEW (CHANGED!)
  - Chunk 1 (5 MB) → hash_b2 (SAME)
  - Chunk 2 (5 MB) → hash_c3 (SAME)
  - Chunk 3 (1 MB) → hash_d4 (SAME)

Client compares with Local Metadata Index:
  - Chunk 0: hash_a1_NEW != hash_a1 → UPLOAD ✅
  - Chunk 1: hash_b2 == hash_b2 → SKIP ❌
  - Chunk 2: hash_c3 == hash_c3 → SKIP ❌
  - Chunk 3: hash_d4 == hash_d4 → SKIP ❌

Only upload Chunk 0!
Bandwidth saved: 11 MB (69%)
```

**Database State:**

```sql
-- Version 1
INSERT INTO file_chunks VALUES 
  ('file_123', 'chunk_a1', 0, 1),
  ('file_123', 'chunk_b2', 1, 1),
  ('file_123', 'chunk_c3', 2, 1),
  ('file_123', 'chunk_d4', 3, 1);

-- Version 2 (reuse chunks b2, c3, d4)
INSERT INTO file_chunks VALUES 
  ('file_123', 'chunk_a1_NEW', 0, 2),  -- New chunk
  ('file_123', 'chunk_b2', 1, 2),      -- Reused
  ('file_123', 'chunk_c3', 2, 2),      -- Reused
  ('file_123', 'chunk_d4', 3, 2);      -- Reused
```

### 12.2 Watcher Service Triggers Re-upload

```java
public class WatcherService {
    
    public void onFileModified(File file) {
        // 1. Get previous version from local DB
        LocalFile prevVersion = localDb.get(file.getPath());
        
        // 2. Chunk new version
        List<Chunk> newChunks = chunker.chunkFile(file);
        
        // 3. Compare hashes
        List<Chunk> changedChunks = new ArrayList<>();
        for (int i = 0; i < newChunks.size(); i++) {
            Chunk newChunk = newChunks.get(i);
            Chunk oldChunk = prevVersion.getChunks().get(i);
            
            if (!newChunk.getHash().equals(oldChunk.getHash())) {
                changedChunks.add(newChunk);
            }
        }
        
        // 4. Upload only changed chunks
        uploadManager.uploadChunks(changedChunks);
    }
}
```

---

<br>

## 13. Download & Sharing

### 13.1 Download Flow

```
User clicks "Download" on file_123
    ↓
Client → GET /v1/files/file_123/download
    ↓
Download Service:
  1. Check permissions (Metadata DB)
     - Does user have READ access?
  2. Get chunk list from Metadata DB
  3. Generate signed URLs for each chunk
    ↓
Response:
{
  "chunks": [
    {"index": 0, "url": "https://s3.amazonaws.com/..."},
    {"index": 1, "url": "https://s3.amazonaws.com/..."},
    {"index": 2, "url": "https://s3.amazonaws.com/..."}
  ]
}
    ↓
Client downloads chunks in parallel
    ↓
Client stitches chunks together
    ↓
Save to local disk
```

### 13.2 Sharing Flow

**1. User shares folder with friend**
```
POST /v1/folders/folder_123/share

{
  "user_email": "friend@example.com",
  "role": "READ"
}
```

**Server Logic:**
```java
public void shareFolder(String folderId, String email, String role) {
    // 1. Find user by email
    User user = userDb.findByEmail(email);
    
    // 2. Insert permission
    Permission perm = new Permission();
    perm.setResourceType("FOLDER");
    perm.setResourceId(folderId);
    perm.setUserId(user.getUserId());
    perm.setRole(role);
    
    permissionDb.insert(perm);
    
    // 3. Notify user
    notificationService.send(user.getUserId(), "You have access to a new folder!");
}
```

**2. Friend accesses folder**
```
GET /v1/folders/folder_123/content

Server checks permissions:
  SELECT * FROM permissions 
  WHERE resource_id = 'folder_123' 
    AND user_id = 'friend_user_id'
    AND role IN ('READ', 'WRITE', 'OWNER');

If exists → Return folder content
If not exists → 403 Forbidden
```

---

<br>

## 14. Optimizations & Follow-ups

### 14.1 Deduplication (Storage Optimization)

**Problem:** Two users upload same file → waste storage

**Solution:** Content-addressable storage (CAS)

**How It Works:**

```
User A uploads cat.jpg (10 MB)
    ↓
Chunker: Chunk 0 → hash_abc123
    ↓
Upload Service checks Chunks table:
    SELECT * FROM chunks WHERE chunk_hash = 'hash_abc123';
    
If EXISTS:
  - Skip upload
  - Reference existing chunk
  - INSERT INTO file_chunks VALUES ('file_userA', 'chunk_abc123', 0, 1);
  - UPDATE chunks SET ref_count = ref_count + 1 WHERE chunk_hash = 'hash_abc123';

If NOT EXISTS:
  - Upload chunk to S3
  - INSERT INTO chunks VALUES ('chunk_abc123', 'hash_abc123', 's3://...', 1);
```

**Savings:**
```
Without dedup: 2 users × 10 MB = 20 MB
With dedup:    1 chunk = 10 MB
Savings:       50%
```

### 14.2 Validator Service (Async Validation)

**Problem:** Validating millions of chunks during upload is expensive

**Wrong Approach:**
```
For each chunk uploaded:
    Validate hash ❌ (millions of validations!)
    CPU exhausted!
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

**Code:**
```java
@Async
public void validateUploadAsync(String uploadId) {
    // Run in background thread pool
    Map<String, String> expectedHashes = redis.hgetall("upload:" + uploadId);
    
    for (int i = 0; i < expectedHashes.size(); i++) {
        byte[] chunkData = s3.getObject("upload_" + uploadId + "/chunk_" + i);
        String actualHash = SHA256.hash(chunkData);
        
        if (!actualHash.equals(expectedHashes.get("chunk_" + i))) {
            requestRetry(uploadId, i);
            return;
        }
    }
    
    metadataService.saveFileMetadata(uploadId);
}
```

### 14.3 Additional Async Services

```
┌─────────────────────────────────────────┐
│         S3 BUCKET (Chunks)               │
└────────┬───────┬───────┬───────┬────────┘
         │       │       │       │
    ┌────▼──┐ ┌──▼───┐ ┌▼────┐ ┌▼────────┐
    │Virus  │ │Thumb │ │De-  │ │Validator│
    │Scanner│ │Gen   │ │dup  │ │Service  │
    └───────┘ └──────┘ └─────┘ └─────────┘
```

**1. Virus Scanner**
```java
@Async
public void scanFile(String fileId) {
    List<Chunk> chunks = getChunks(fileId);
    
    for (Chunk chunk : chunks) {
        byte[] data = s3.getObject(chunk.getS3Key());
        
        if (virusScanner.scan(data).isInfected()) {
            quarantineFile(fileId);
            notifyUser(fileId, "File contains malware!");
            return;
        }
    }
}
```

**2. Thumbnail Generator**
```java
@Async
public void generateThumbnail(String fileId) {
    if (!isImage(fileId)) return;
    
    byte[] imageData = downloadFile(fileId);
    byte[] thumbnail = ImageUtils.resize(imageData, 200, 200);
    
    s3.putObject("thumbnails/" + fileId, thumbnail);
}
```

**3. Deduplication Service**
```java
@Async
public void deduplicateChunks() {
    // Find duplicate chunks
    List<Chunk> duplicates = db.query(
        "SELECT chunk_hash, COUNT(*) as count " +
        "FROM chunks " +
        "GROUP BY chunk_hash " +
        "HAVING count > 1"
    );
    
    for (Chunk dup : duplicates) {
        // Keep one, delete others, update references
        consolidateChunks(dup.getChunkHash());
    }
}
```

### 14.4 Garbage Collection

**Problem:** Deleted files still occupy S3 storage

**Solution:** Soft delete + GC job

**Flow:**
```
User deletes file
    ↓
UPDATE files SET is_deleted = TRUE WHERE file_id = 'file_123';
    ↓
Chunks remain in S3 (for 30 days)
    ↓
GC Job (runs daily):
    SELECT * FROM files WHERE is_deleted = TRUE AND deleted_at < NOW() - INTERVAL '30 days';
    
    For each file:
        - Get chunks
        - Decrement ref_count
        - If ref_count == 0:
            DELETE FROM chunks WHERE chunk_id = 'chunk_123';
            s3.deleteObject(chunk.getS3Key());
```

---

<br>

## 15. Interview Q&A

### Q1: Why use signed URLs instead of uploading via backend?

**A:**

**Via Backend:**
```
Client → Load Balancer → Upload Service → S3
16 MB     16 MB           16 MB
Total: 48 MB bandwidth used ❌
```

**Signed URL:**
```
Client → S3
16 MB only
Savings: 66% ✅
```

**Additional Benefits:**
- Reduces backend load
- Faster uploads (direct to S3)
- Scalable (S3 handles millions of requests)

---

### Q2: Why chunk files?

**A:**

1. **Resume uploads:** Network fails → resume from last chunk
2. **Parallel uploads:** Upload 3 chunks simultaneously → 3x faster
3. **Deduplication:** Reuse identical chunks across users
4. **Delta sync:** Only upload changed chunks on file edit
5. **Bandwidth optimization:** 69% savings on edits

---

### Q3: How to handle storage quota?

**A:**

```java
void checkQuota(String userId, long fileSize) {
    User user = db.getUser(userId);
    
    long available = user.getStorageQuota() - user.getStorageUsed();
    
    if (fileSize > available) {
        throw new QuotaExceededException("Upgrade to premium!");
    }
    
    // Reserve space during upload
    user.setStorageUsed(user.getStorageUsed() + fileSize);
    db.update(user);
}
```

**On upload failure:**
```java
void rollbackQuota(String userId, long fileSize) {
    User user = db.getUser(userId);
    user.setStorageUsed(user.getStorageUsed() - fileSize);
    db.update(user);
}
```

---

### Q4: How to handle conflicts (two devices edit same file offline)?

**A:**

**Scenario:**
```
Device A (offline): Edits doc.txt → v2_A
Device B (offline): Edits doc.txt → v2_B
Both come online
```

**Solution:**
```
1. Detect conflict (both have parent version v1)
2. Server keeps one as v2 (e.g., last write wins)
3. Create "Conflicted Copy":
   - doc.txt (Device A's conflicted copy).txt
4. Notify user to manually resolve
```

**Code:**
```java
if (hasConflict(fileId)) {
    File original = getLatestVersion(fileId);
    File conflicted = getConflictedVersion(fileId);
    
    // Rename conflicted version
    conflicted.setName(conflicted.getName() + " (Conflicted Copy)");
    saveAsNewFile(conflicted);
    
    notifyUser("Conflict detected. Please review.");
}
```

---

### Q5: How to scale to millions of users?

**A:**

| Component | Scaling Strategy |
|-----------|------------------|
| **Upload Service** | Horizontal scaling (stateless) |
| **Metadata DB** | Sharding by user_id |
| **Redis** | Cluster mode (partitioning) |
| **S3** | Infinite (managed by AWS) |
| **Kafka** | Partitioning by user_id |

**Calculation:**
```
1M users × 10 GB = 10 PB storage
S3: $23/TB/month = $230,000/month
```

---

### Q6: How to ensure zero data loss?

**A:**

1. **S3 Replication:** Cross-region replication (99.999999999% durability)
2. **Metadata DB:** Master-slave replication, daily backups
3. **Hash Validation:** SHA-256 ensures integrity
4. **Soft Delete:** 30-day grace period before permanent deletion

---

### Q7: Security considerations?

**A:**

1. **At Rest:** S3 server-side encryption (SSE-S3/KMS)
2. **In Transit:** HTTPS/TLS 1.3
3. **Signed URLs:** Expire in 1 hour, scoped to specific object
4. **Permissions:** Check ACL before generating download URL
5. **Virus Scanning:** Async post-upload

---

### Q8: Why Redis for upload session state?

**A:**

| Feature | Redis | PostgreSQL |
|---------|-------|------------|
| **Latency** | < 1ms | 10-50ms |
| **Durability** | No (TTL) | Yes |
| **Use Case** | Temp upload state | Permanent metadata |

**During upload:** Redis (fast, temporary)  
**After commit:** PostgreSQL (durable, permanent)

---

### Q9: How to handle large files (100GB)?

**A:**

**Same chunking strategy:**
```
100 GB file
    ↓
Chunk size: 5 MB
    ↓
Total chunks: 20,480
    ↓
Upload 3 chunks in parallel
    ↓
Time: ~10 hours (on 10 Mbps connection)
```

**Optimizations:**
- Increase chunk size to 10 MB (reduce API calls)
- Upload 5 chunks in parallel (faster)
- Resume on failure (no restart from scratch)

---

### Q10: Cost optimization strategies?

**A:**

1. **Deduplication:** 50% storage savings
2. **Compression:** Gzip chunks before upload (30-40% savings)
3. **Cold Storage:** Move old files to S3 Glacier (90% cheaper)
4. **CDN:** Cache frequently accessed files (reduce S3 GET costs)

**Example:**
```
Without optimization: 10 PB × $23/TB = $230,000/month
With dedup (50%):     5 PB × $23/TB = $115,000/month
With Glacier (80%):   1 PB × $23/TB + 4 PB × $4/TB = $39,000/month

Total savings: 83% ($191,000/month)
```

---

<br>

## 16. Summary

### Key Takeaways

✅ **Folders = Metadata:** No real directories, just JSON  
✅ **Chunking:** 4-5 MB chunks for large files  
✅ **Signed URLs:** Direct upload to S3 (bypass backend)  
✅ **Hash Validation:** SHA-256 for integrity  
✅ **Redis Cache:** Temporary upload state  
✅ **Versioning:** Only upload changed chunks  
✅ **Deduplication:** Content-based addressing (50% savings)  
✅ **Sync:** Push (WebSocket) + Pull (polling) hybrid  
✅ **Local Index:** SQLite for client-side metadata  
✅ **Async Jobs:** Validation, dedup, virus scan, thumbnails  

### Complete Upload Flow

```
1. Init: Check quota, return uploadId + chunkSize
2. Chunk: Split file, generate hashes
3. URLs: Get signed URLs for each chunk
4. Upload: Direct to S3 (parallel)
5. Commit: Validate hashes, save metadata
6. Sync: Notify all devices via Kafka
```

### Architecture

- **6 core services:** User, Upload, Metadata, Sync, Validator, Download
- **Redis:** Temp state, PostgreSQL for metadata
- **S3:** Blob storage
- **Kafka:** Sync events
- **Async validators:** Hash, virus, dedup, thumbnails

### Performance

- **Millions of users**
- **Chunked uploads:** Resume support
- **Deduplication:** 50% savings
- **Eventual consistency:** Sync in < 5 seconds

---

**End of Lecture 18**
