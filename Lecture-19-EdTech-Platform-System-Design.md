# EdTech Platform System Design (like Udemy/Coursera)

> **Topic:** System Design  
> **Application:** Online Learning Platform  
> **Scale:** 100K Courses, 1M Students  
> **Difficulty:** High

---

## 📋 Table of Contents

1. [Brief Introduction of the Application](#1-brief-introduction-of-the-application)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Identification of Core Entities](#3-identification-of-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Interview-Style Design Discussion (Q&A Format)](#7-interview-style-design-discussion-qa-format)
8. [Additional Insights](#8-additional-insights)

---

<br>

## 1. Brief Introduction of the Application

### What is an EdTech Platform?

An EdTech platform is an online learning system where:
- **Instructors** upload courses (videos, quizzes, assignments)
- **Students** purchase and consume educational content
- **Moderators** validate content before publishing

### Key Actors

1. **Students/Users** - Browse, purchase, and consume courses
2. **Instructors** - Create and upload course content
3. **Moderators/Admins** - Validate and publish courses

### Problem It Solves

1. **Global Access:** Learn from anywhere, anytime
2. **Scalable Education:** One instructor can teach millions
3. **Progress Tracking:** Monitor learning journey
4. **Quality Control:** Moderated content ensures quality
5. **Monetization:** Platform for instructors to earn

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements (Student Side)

1. **User Account Management**
   - Students can create accounts and login
   - Profile management

2. **Course Discovery**
   - Search courses by category, rating, difficulty, price
   - Browse and filter courses

3. **Course Enrollment**
   - Enroll in free courses
   - Purchase paid courses via payment gateway

4. **Progress Tracking**
   - Track completion percentage
   - Resume from last watched position

5. **Assessment System**
   - Take quizzes and assignments
   - Submit answers and get results

6. **Review & Rating System**
   - Leave reviews and ratings for courses
   - Read other students' feedback

### 2.2 Functional Requirements (Instructor Side)

1. **Instructor Onboarding**
   - Register as instructor
   - Verification process (KYC)

2. **Course Creation**
   - Upload course metadata (title, description, category)
   - Upload videos, documents, assignments
   - Create quizzes and assessments
   - Set pricing (free/paid)

3. **Course Publishing**
   - Submit for moderation
   - Publish after approval

### 2.3 Non-Functional Requirements

1. **Scale**
   - Support 100,000+ courses
   - Handle 1 million+ students globally

2. **CAP Theorem - Availability > Consistency**
   - **High Availability:** Platform should be accessible 24/7
   - **Eventual Consistency:** Course updates can have slight delay
   - **Exception:** Payment flow must be strongly consistent

3. **Large File Support**
   - Support video files (1+ hour duration)
   - Handle large assignments and documents

4. **Low Latency for Video Streaming**
   - Video should start playing within 2 seconds
   - Smooth streaming with minimal buffering

5. **Distributed System**
   - Global CDN for content delivery
   - Partition tolerance is implicit

---

## 3. Identification of Core Entities

Based on functional requirements, core entities are:

1. **User** - Students and their profiles
2. **Instructor** - Course creators and their profiles
3. **Course** - Course metadata, chapters, lessons
4. **Payment** - Transaction records
5. **Enrollment** - Student-Course mapping
6. **Progress** - Learning progress tracking
7. **Review** - Ratings and comments
8. **Quiz/Assessment** - Questions and answers

---

## 4. API Design

### 4.1 Course APIs

```
# Search courses with filters
GET /api/v1/courses/search?category={category}&price={price}&rating={rating}&level={level}

# Get course details
GET /api/v1/courses/{courseId}

# Submit a course (Instructor)
POST /api/v1/courses

# Publish a course (Moderator)
POST /api/v1/courses/{courseId}/publish
```

### 4.2 Progress APIs

```
# Submit progress
POST /api/v1/progress
Body: {
  "userId": "123",
  "courseId": "456",
  "videoId": "789",
  "timestamp": 1234
}

# Get progress
GET /api/v1/progress/{userId}/{courseId}
```

### 4.3 Enrollment APIs

```
# Enroll in a course
POST /api/v1/enrollments
Body: {
  "userId": "123",
  "courseId": "456"
}

# Get enrolled students (Instructor)
GET /api/v1/courses/{courseId}/enrollments

# Get my enrollments (Student)
GET /api/v1/users/{userId}/enrollments
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────┐                    ┌─────────────────┐
│   Student   │                    │   Instructor    │
│   Client    │                    │     Client      │
└──────┬──────┘                    └────────┬────────┘
       │                                    │
       │                                    │
       ▼                                    ▼
┌─────────────────────────────────────────────────────┐
│           Load Balancer / API Gateway               │
└──────────────┬──────────────────────┬───────────────┘
               │                      │
       ┌───────▼────────┐    ┌────────▼──────────┐
       │  Student Flow  │    │  Instructor Flow  │
       └───────┬────────┘    └────────┬──────────┘
               │                      │
               │                      │
    ┌──────────┼──────────┐          │
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ User   │ │ Course │ │Playback│ │Catalog │
│Service │ │ Search │ │Service │ │Service │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────────────────────────────────────┐
│          Database Layer                │
│  - User DB                             │
│  - Course DB                           │
│  - Enrollment DB                       │
│  - Progress DB                         │
└────────────────────────────────────────┘
```

### 5.2 Key Services Overview

**Student Side:**
- **User Service:** Authentication, registration
- **Course Search Service:** Find courses with filters
- **Enrollment Service:** Enroll in courses
- **Payment Service:** Handle transactions
- **Video Playback Service:** Stream course videos
- **Progress Service:** Track learning progress
- **Review Service:** Ratings and comments

**Instructor Side:**
- **Catalog Service:** Upload course metadata
- **Media Uploader Service:** Upload videos/files
- **Moderator Service:** Content validation

---

## 6. Low-Level Design (LLD)

### 6.1 Complete Architecture Diagram

```
                    STUDENT FLOW
                    ============

┌──────────┐
│ Students │
└────┬─────┘
     │
     ▼
┌─────────────────┐
│ Load Balancer   │
│  API Gateway    │
└────┬────────────┘
     │
     ├──────────────────────────────────────────┐
     │                                          │
     ▼                                          ▼
┌──────────────┐                         ┌──────────────┐
│ User Service │                         │Course Search │
│              │                         │   Service    │
│ - Signup     │                         │              │
│ - Login      │                         │ Filters:     │
│ - Profile    │                         │ - Category   │
└──────┬───────┘                         │ - Rating     │
       │                                 │ - Price      │
       ▼                                 │ - Level      │
┌──────────────┐                         └──────┬───────┘
│   User DB    │                                │
│              │                                ▼
│ - userId     │                         ┌──────────────┐
│ - name       │                         │Elasticsearch │
│ - email      │                         │              │
│ - password   │                         │ Indexed:     │
└──────────────┘                         │ - Title      │
                                         │ - Category   │
     │                                   │ - Rating     │
     ▼                                   │ - Price      │
┌──────────────┐                         └──────┬───────┘
│ Enrollment   │                                │
│   Service    │                                │
└──────┬───────┘                         ┌──────▼───────┐
       │                                 │ Aggregator   │
       │                                 │ CDC Pipeline │
       ▼                                 │              │
┌──────────────┐                         │ Joins:       │
│  Payment     │                         │ - Course     │
│  Service     │                         │ - Reviews    │
│              │                         │ - Stats      │
│ Free? ──────►│                         └──────▲───────┘
│              │                                │
│ Paid? ──────►│                                │
└──────┬───────┘                         ┌──────┴───────┐
       │                                 │  Course DB   │
       ▼                                 │              │
┌──────────────┐                         │ Tables:      │
│   Payment    │                         │ - courses    │
│   Gateway    │                         │ - pricing    │
│              │                         │ - stats      │
│ - Razorpay   │                         │ - reviews    │
│ - Stripe     │                         └──────────────┘
└──────┬───────┘
       │
       ▼
┌──────────────────────────────┐
│      Payment DB              │
│                              │
│ Tables:                      │
│ 1. payment_table             │
│    - paymentId               │
│    - userId                  │
│    - courseId                │
│    - amount                  │
│    - status                  │
│                              │
│ 2. payment_outbox_table      │
│    - paymentId               │
│    - userId                  │
│    - courseId                │
│    - metadata                │
│    - consumed (boolean)      │
│    - timestamp               │
└──────┬───────────────────────┘
       │
       │ CDC Pipeline
       ▼
┌──────────────┐
│    Kafka     │
│   Broker     │
└──┬────────┬──┘
   │        │
   │        └──────────────────┐
   │                           │
   ▼                           ▼
┌──────────────┐        ┌──────────────┐
│ Notification │        │ Permission   │
│   Service    │        │ Sync Service │
│              │        │              │
│ - Email      │        │ Grants access│
│ - SMS        │        └──────┬───────┘
└──────────────┘               │
                               ▼
                        ┌──────────────┐
                        │ Enrollment   │
                        │      DB      │
                        │              │
                        │ - accessId   │
                        │ - userId     │
                        │ - courseId   │
                        │ - scope      │
                        │ - status     │
                        │ - purchaseDate│
                        └──────────────┘


                    VIDEO PLAYBACK FLOW
                    ===================

┌──────────┐
│ Student  │
│ (clicks  │
│  video)  │
└────┬─────┘
     │
     ▼
┌──────────────┐
│   Playback   │
│   Service    │
│              │
│ 1. Check     │
│    Access    │◄──────┐
└──────┬───────┘       │
       │               │
       ▼               │
┌──────────────┐       │
│ Enrollment   │       │
│      DB      │       │
│              │       │
│ Verify user  │       │
│ has access   │       │
└──────┬───────┘       │
       │               │
       │ Authorized    │
       ▼               │
┌──────────────┐       │
│   Return     │       │
│  Manifest    │       │
│    File      │       │
│              │       │
│ Contains:    │       │
│ - Video URLs │       │
│ - Resolutions│       │
│ - Chunks     │       │
└──────┬───────┘       │
       │               │
       ▼               │
┌──────────────┐       │
│   Client     │       │
│   Reads      │       │
│  Manifest    │       │
└──────┬───────┘       │
       │               │
       ▼               │
┌──────────────┐       │
│     CDN      │       │
│   (Local)    │       │
│              │       │
│ Cache Hit?   │       │
│  Yes ──► Stream      │
│              │       │
│  No ──► Fetch│       │
└──────┬───────┘       │
       │               │
       │ Cache Miss    │
       ▼               │
┌──────────────┐       │
│  Blob Storage│       │
│   (S3)       │       │
│              │       │
│ - Videos     │       │
│   (chunked)  │       │
│ - Multiple   │       │
│   resolutions│       │
│ - Thumbnails │       │
│ - Attachments│       │
└──────────────┘       │
                       │
                       │
    PROGRESS TRACKING  │
    =================  │
                       │
┌──────────────┐       │
│   Student    │       │
│  (watches    │       │
│   video)     │       │
└──────┬───────┘       │
       │               │
       │ Every 10s     │
       ▼               │
┌──────────────┐       │
│    Kafka     │       │
│   Broker     │       │
│              │       │
│ Events:      │       │
│ - userId     │       │
│ - courseId   │       │
│ - videoId    │       │
│ - timestamp  │       │
└──────┬───────┘       │
       │               │
       │ Batch Job     │
       │ (every 1 min) │
       ▼               │
┌──────────────┐       │
│  Consumer    │       │
│  Service     │       │
│              │       │
│ Reads last   │       │
│ event only   │       │
└──────┬───────┘       │
       │               │
       ▼               │
┌──────────────┐       │
│  Progress DB │       │
│              │       │
│ - userId     │       │
│ - courseId   │       │
│ - videoId    │       │
│ - duration   │       │
│ - lastTimestamp│     │
│ - percentage │       │
└──────────────┘       │
                       │
                       │
    INSTRUCTOR FLOW    │
    ===============    │
                       │
┌──────────┐           │
│Instructor│           │
└────┬─────┘           │
     │                 │
     ▼                 │
┌─────────────────┐    │
│ Load Balancer   │    │
│  API Gateway    │    │
└────┬────────────┘    │
     │                 │
     ├─────────────────┼──────────────┐
     │                 │              │
     ▼                 ▼              ▼
┌──────────┐    ┌──────────┐   ┌──────────┐
│  User    │    │ Catalog  │   │  Media   │
│ Service  │    │ Service  │   │ Uploader │
│          │    │          │   │ Service  │
│(Instructor│   │- Course  │   │          │
│onboarding)│   │  metadata│   │- Videos  │
└────┬─────┘    │- Quizzes │   │- Files   │
     │          │- Chapters│   │- Chunks  │
     ▼          └────┬─────┘   └────┬─────┘
┌──────────┐        │              │
│Instructor│        ▼              ▼
│   DB     │   ┌──────────┐   ┌──────────┐
│          │   │ Course   │   │  Blob    │
│- userId  │   │   DB     │   │ Storage  │
│- name    │   │          │   │  (S3)    │
│- verified│   │- courseId│   │          │
│- rating  │   │- title   │   │- Videos  │
│- students│   │- desc    │   │  (chunked│
│- courses │   │- category│   │   encoded)│
└──────────┘   │- level   │   │- Thumbnails│
               │- thumbnail│  │- Docs    │
               └────┬─────┘   └──────────┘
                    │
                    ▼
               ┌──────────┐
               │  Quiz DB │
               │          │
               │- quizId  │
               │- courseId│
               │- questions│
               │- answers │
               └────┬─────┘
                    │
                    ▼
               ┌──────────┐
               │Moderator │
               │ Service  │
               │          │
               │- Validate│
               │- Approve │
               │- Publish │
               └──────────┘
```

### 6.2 Database Schemas

#### User DB (PostgreSQL)
```sql
users_table:
- userId (PK)
- name
- email
- phoneNumber
- password (hashed)
- accountStatus
- createdAt
```

#### Instructor DB (PostgreSQL)
```sql
instructors_table:
- userId (PK)
- name
- verificationStatus (verified/pending/rejected)
- phoneNumber
- email
- password (hashed)
- averageRating
- totalStudents
- totalCourses
- accountStatus
- createdAt
```

#### Course DB (PostgreSQL)
```sql
courses_table:
- courseId (PK)
- instructorId (FK)
- title
- description
- category
- level (beginner/intermediate/advanced)
- thumbnailURL
- status (draft/published/archived)
- createdAt
- updatedAt

pricing_table:
- priceId (PK)
- courseId (FK)
- amount
- currency
- discountPercent

course_stats_table:
- courseId (PK)
- totalEnrollments
- averageRating
- totalReviews
- completionRate

reviews_table:
- reviewId (PK)
- courseId (FK)
- userId (FK)
- rating (1-5)
- comment
- createdAt
```

#### Quiz DB (PostgreSQL)
```sql
quiz_table:
- quizId (PK)
- courseId (FK)
- lessonId (FK)
- title
- passingScore
- totalMarks

questions_table:
- questionId (PK)
- quizId (FK)
- questionText
- questionType (MCQ/descriptive)
- options (JSON)
- correctAnswer
- marks
```

#### Enrollment DB (PostgreSQL)
```sql
enrollments_table:
- accessId (PK)
- userId (FK)
- courseId (FK)
- scope (lifetime/subscription)
- status (active/expired/revoked)
- purchaseDate
- expiryDate
```

#### Payment DB (PostgreSQL)
```sql
payment_table:
- paymentId (PK)
- userId (FK)
- courseId (FK)
- amount
- currency
- status (success/failed/pending)
- transactionId
- createdAt

payment_outbox_table:
- outboxId (PK)
- paymentId (FK)
- userId
- courseId
- eventType
- consumed (boolean)
- timestamp
```

#### Progress DB (PostgreSQL)
```sql
progress_table:
- progressId (PK)
- userId (FK)
- courseId (FK)
- videoId (FK)
- durationSeconds
- lastTimestamp
- completionPercentage
- updatedAt
```

### 6.3 Key Design Patterns

#### 6.3.1 Outbox Pattern for Payment

**Problem:** When payment succeeds, we need to:
1. Save payment record
2. Grant course access to user

If we update two databases separately, one might fail!

**Solution: Outbox Pattern**

```
Payment Success
      │
      ▼
┌─────────────────────────────┐
│   Atomic Transaction        │
│                             │
│  1. Insert into payment_table│
│  2. Insert into outbox_table │
└─────────────┬───────────────┘
              │
              ▼
        CDC Pipeline
              │
              ▼
         Kafka Broker
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
Notification    Permission Sync
  Service           Service
                      │
                      ▼
                Enrollment DB
```

**Benefits:**
- Atomic writes (both succeed or both fail)
- Resilient to failures
- Event-driven architecture
- No data loss

#### 6.3.2 CDC Pipeline for Elasticsearch

**Problem:** Course data is spread across multiple tables:
- courses_table
- pricing_table
- reviews_table
- course_stats_table

Joining at query time is slow!

**Solution: Aggregator CDC Pipeline**

```
Course DB (Multiple Tables)
      │
      ▼
Aggregator CDC Pipeline
      │
      │ Joins all tables
      │ Creates JSON document
      ▼
Elasticsearch
      │
      │ Indexed for fast search
      ▼
Course Search Service
```

**Example Aggregated Document:**
```json
{
  "courseId": "123",
  "title": "System Design Masterclass",
  "category": "Engineering",
  "level": "Advanced",
  "price": 4999,
  "rating": 4.8,
  "totalReviews": 1250,
  "instructor": "Interview with Bunny"
}
```

#### 6.3.3 Video Chunking & Adaptive Streaming

**Upload Flow:**
```
Instructor uploads 1-hour video
      │
      ▼
Media Uploader Service
      │
      ├─► Chunker (splits into 10s chunks)
      │
      ├─► Encoder (creates multiple resolutions)
      │   - 4K (2160p)
      │   - Full HD (1080p)
      │   - HD (720p)
      │   - SD (480p)
      │   - Mobile (360p)
      │
      └─► Upload to S3 (via signed URLs)
```

**Playback Flow:**
```
Student clicks video
      │
      ▼
Playback Service
      │
      ├─► Check access (Enrollment DB)
      │
      └─► Return manifest file
            │
            ▼
Client reads manifest
      │
      ├─► Detects bandwidth
      │
      └─► Requests appropriate resolution
            │
            ▼
      CDN (cache hit)
            │
            └─► S3 (cache miss)
```

**Manifest File Example:**
```json
{
  "courseId": "123",
  "videoId": "456",
  "resolutions": [
    {
      "quality": "1080p",
      "url": "https://cdn.example.com/video_1080p.m3u8"
    },
    {
      "quality": "720p",
      "url": "https://cdn.example.com/video_720p.m3u8"
    }
  ]
}
```

### 6.4 Scalability Considerations

#### 6.4.1 Read vs Write Traffic

**Observation:**
- 80% traffic: Video playback (read-heavy)
- 20% traffic: Course creation (write-heavy)

**Solution:**
- More Playback Service instances (5-6)
- Fewer Catalog Service instances (2-3)

#### 6.4.2 Caching Strategy

**CDN Caching:**
- Cache videos at edge locations
- Reduces S3 load by 90%
- Latency: 2ms (vs 200ms from S3)

**Redis Caching:**
- Cache course metadata
- Cache user enrollment status
- TTL: 1 hour

#### 6.4.3 Database Sharding

**Course DB Sharding:**
- Shard by `courseId`
- Consistent hashing
- Each shard: 10K courses

**User DB Sharding:**
- Shard by `userId`
- Geographic sharding (optional)

### 6.5 Instructor Verification Flow

**Problem:** How to verify instructor identity?

**Solution: Third-Party KYC Integration**

```
Instructor signs up
      │
      ▼
User Service
      │
      ▼
Instructor DB (status: pending)
      │
      ▼
KYC Service (e.g., UIDAI in India)
      │
      ├─► Verify Aadhaar
      ├─► Verify PAN
      └─► Verify Phone
            │
            ▼
      Update status: verified
```

### 6.6 Progress Tracking Optimization

**Problem:** Sending progress every 10 seconds = too many writes!

**Solution: Kafka + Batch Processing**

```
Client sends event every 10s
      │
      ▼
Kafka Broker (stores all events)
      │
      ▼
Consumer Service (batch job every 1 min)
      │
      ├─► Read all events for user
      ├─► Keep only LAST event
      └─► Update Progress DB
```

**Benefits:**
- Reduces DB writes by 6x (60s / 10s)
- No data loss (Kafka persists events)
- Can replay events if needed

---

## 7. Interview-Style Design Discussion (Q&A Format)

### **Q1: Why separate User Service and Instructor Service?**

**A:** We could use a single User Service with a `role` field:
```sql
users_table:
- userId
- role (student/instructor)
```

But separating has benefits:
- **Different schemas:** Instructors need `verificationStatus`, `totalCourses`
- **Different scaling:** More students than instructors
- **Security:** Separate authentication flows
- **Clarity:** Easier to maintain

For this design, we use **single User Service** with two tables for simplicity.

---

### **Q2: Why use Elasticsearch instead of database queries?**

**A:** 

**Without Elasticsearch:**
```sql
SELECT c.*, p.price, s.rating 
FROM courses c
JOIN pricing p ON c.courseId = p.courseId
JOIN course_stats s ON c.courseId = s.courseId
WHERE c.category = 'Engineering'
  AND p.price < 5000
  AND s.rating > 4.0
```
- **Latency:** 200-500ms (multiple joins)
- **Not scalable:** Slow for millions of courses

**With Elasticsearch:**
- Pre-aggregated data
- Inverted index for text search
- **Latency:** 10-50ms
- Supports fuzzy search, autocomplete

---

### **Q3: Why use Outbox Pattern for payments?**

**A:** 

**Without Outbox (Direct Approach):**
```
1. Save to Payment DB ✅
2. Save to Enrollment DB ❌ (fails)
```
Result: Payment recorded, but user has no access! 💥

**With Outbox Pattern:**
```
1. Atomic write to Payment DB + Outbox ✅
2. CDC publishes to Kafka ✅
3. Consumer updates Enrollment DB ✅
```
- If Kafka is down, CDC retries
- If consumer fails, Kafka retains event
- **Guaranteed delivery**

---

### **Q4: Why use CDN for videos?**

**A:** 

**Without CDN:**
- Student in India fetches video from S3 in US
- Latency: 200-500ms
- Bandwidth cost: High

**With CDN:**
- Video cached at edge location (Mumbai)
- Latency: 2-10ms
- Bandwidth cost: 70% lower
- Better user experience

---

### **Q5: How to handle video uploads (large files)?**

**A:** 

**Chunked Upload with Signed URLs:**

```
1. Instructor requests upload
      │
      ▼
2. Media Uploader generates signed URLs
      │
      ▼
3. Client splits video into chunks (10MB each)
      │
      ▼
4. Client uploads chunks directly to S3
      │
      ▼
5. S3 notifies Media Uploader on completion
      │
      ▼
6. Media Uploader triggers encoding pipeline
```

**Benefits:**
- No server bottleneck
- Resumable uploads
- Parallel chunk uploads

---

### **Q6: Why batch process for progress tracking?**

**A:** 

**Without Batching:**
- Event every 10s → 6 writes/min per user
- 1M users watching → 6M writes/min
- Database overload! 💥

**With Batching:**
- Collect events in Kafka
- Process every 1 min
- Keep only last event
- 1M users → 1M writes/min (6x reduction)

---

### **Q7: How to ensure payment consistency?**

**A:** 

Payment flow must be **strongly consistent**:

```
Payment Gateway → Payment Service
                       │
                       ▼
              Atomic Transaction:
              - payment_table
              - payment_outbox_table
```

**Why atomic?**
- Both writes succeed or both fail
- No partial state
- User either gets access or gets refund

**Other flows can be eventually consistent:**
- Course updates (can take 1-2 seconds to propagate)
- Review updates (not critical)

---

### **Q8: How to handle course moderation?**

**A:** 

```
Instructor uploads course
      │
      ▼
Status: DRAFT (not visible to students)
      │
      ▼
Moderator Service
      │
      ├─► Check metadata (Course DB)
      ├─► Check videos (S3)
      └─► Check quizzes (Quiz DB)
            │
            ▼
      Moderator approves
            │
            ▼
      Status: PUBLISHED
            │
            ▼
      Visible to students
```

**Moderator checks:**
- No inappropriate content
- Accurate course description
- Video quality
- Quiz validity

---

### **Q9: How to scale for 1M concurrent users?**

**A:** 

**Horizontal Scaling:**
- Load Balancer distributes traffic
- Auto-scaling groups for services
- Database read replicas

**Caching:**
- CDN for videos (90% cache hit rate)
- Redis for metadata (80% cache hit rate)

**Database Optimization:**
- Sharding by courseId, userId
- Indexing on frequently queried fields
- Read replicas for read-heavy queries

**Example:**
- 1M users watching videos
- 90% served by CDN (900K)
- 10% hit S3 (100K)
- S3 can handle 100K requests/sec easily

---

### **Q10: How to handle free vs paid courses?**

**A:** 

```
Student clicks "Enroll"
      │
      ▼
Enrollment Service checks pricing_table
      │
      ├─► Free course?
      │   └─► Direct insert to Enrollment DB
      │
      └─► Paid course?
          └─► Redirect to Payment Service
                │
                ▼
          Payment Gateway
                │
                ▼
          On success → Enrollment DB
```

**No payment processing for free courses!**

---

## 8. Additional Insights

### 8.1 Enterprise Features (Follow-up Question)

**Scenario:** Make this B2B (Enterprise) platform

**Requirements:**
1. Organizations (Google, Microsoft) can onboard
2. Each org has custom landing page
3. Org admins assign courses to employees
4. Track employee progress

**Solution:**

#### 8.1.1 Organization Onboarding
```sql
organizations_table:
- orgId (PK)
- orgName
- domain (e.g., @google.com)
- landingPageTemplate (JSON)
- subscriptionTier
- createdAt
```

#### 8.1.2 Custom Landing Pages
```
Employee logs in with SSO
      │
      ▼
Extract domain from email
      │
      ▼
Fetch orgId from organizations_table
      │
      ▼
Load landingPageTemplate
      │
      ▼
Render custom UI
```

**Template Example:**
```json
{
  "orgId": "google",
  "logo": "https://cdn.example.com/google-logo.png",
  "primaryColor": "#4285F4",
  "featuredCourses": ["course1", "course2"]
}
```

#### 8.1.3 Data Isolation (Multi-Tenancy)

**Option 1: Shared Database with Tenant ID**
```sql
courses_table:
- courseId (PK)
- orgId (Tenant ID)
- ...

enrollments_table:
- enrollmentId (PK)
- orgId (Tenant ID)
- ...
```

**Option 2: Database per Tenant (Sharding)**
```
google_db
  - courses
  - enrollments

microsoft_db
  - courses
  - enrollments
```

**Recommendation:** Option 2 for enterprise (better isolation)

#### 8.1.4 Subdomain Routing

**Problem:** Each org wants custom URL

**Solution:**
```
google.udemy.com  → Google's landing page
microsoft.udemy.com → Microsoft's landing page
```

**Implementation:**
- DNS CNAME records
- API Gateway routes by subdomain
- Load balancer extracts subdomain from Host header

---

### 8.2 Analytics & Reporting

**Instructor Dashboard:**
- Total enrollments
- Revenue generated
- Average rating
- Completion rate

**Student Dashboard:**
- Courses in progress
- Certificates earned
- Learning streak

**Implementation:**
- Separate analytics service
- Read from replica databases
- Pre-aggregated metrics in Redis

---

### 8.3 Notification System

**Triggers:**
- Course purchased → Email confirmation
- New course from followed instructor → Push notification
- Quiz deadline approaching → Email reminder

**Implementation:**
- Kafka consumers for each notification type
- Notification Service with email/SMS/push providers
- User preferences for notification settings

---

### 8.4 Certificate Generation

**Flow:**
```
Student completes course (100%)
      │
      ▼
Progress Service detects completion
      │
      ▼
Publish event to Kafka
      │
      ▼
Certificate Service
      │
      ├─► Generate PDF certificate
      ├─► Upload to S3
      └─► Send email with link
```

---

### 8.5 Content Recommendation

**Algorithm:**
- Collaborative filtering (users who took X also took Y)
- Content-based (similar categories/instructors)
- Trending courses (high enrollment rate)

**Implementation:**
- ML model trained on enrollment data
- Batch processing (daily)
- Results cached in Redis

---

### 8.6 Rate Limiting

**Prevent abuse:**
- Course creation: 10 courses/day per instructor
- Review submission: 1 review/course per user
- Video uploads: 100GB/day per instructor

**Implementation:**
- Redis with sliding window
- API Gateway enforces limits

---

### 8.7 Monitoring & Observability

**Key Metrics:**
- Video playback latency (p50, p95, p99)
- Payment success rate
- Course search latency
- CDN cache hit rate

**Tools:**
- Prometheus for metrics
- Grafana for dashboards
- ELK stack for logs
- Distributed tracing (Jaeger)

---

## Summary

This EdTech platform design covers:

✅ **Scalability:** Handles 1M+ students, 100K+ courses  
✅ **Reliability:** Outbox pattern, CDC pipelines  
✅ **Performance:** CDN, caching, Elasticsearch  
✅ **Consistency:** Strong consistency for payments  
✅ **Availability:** Distributed architecture, no SPOF  
✅ **Extensibility:** Enterprise features, multi-tenancy  

**Key Takeaways:**
1. Separate read-heavy (playback) from write-heavy (upload) services
2. Use Outbox Pattern for critical transactions
3. Leverage CDN for global video delivery
4. Batch process non-critical events (progress tracking)
5. Pre-aggregate data for fast search (Elasticsearch)
6. Design for multi-tenancy from day one (enterprise)

---

**Related System Designs:**
- [OTT Platform Design](Lecture-04-OTT-Platform-System-Design.md) - Video streaming patterns
- [Payment Gateway Design](Lecture-17-Payment-Gateway-System-Design.md) - Payment flows
- [Distributed Storage Design](Lecture-18-Distributed-Storage-Platform-System-Design.md) - File upload patterns

---

> **Interview Tip:** This is a complex design with multiple actors. Always clarify with interviewer which flow to focus on (student/instructor/moderator). Don't try to cover everything in 45 minutes!
