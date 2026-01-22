# Lecture 13: Social Media Platform System Design (Facebook/Instagram)

> **Lecture:** 13  
> **Topic:** System Design  
> **Application:** Social Media Platform (Facebook, Instagram, Twitter)  
> **Scale:** 500 Million Daily Active Users  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-12-Stock-Trading-Platform-System-Design|Lecture 12: Stock Trading Platform]]

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

### What is a Social Media Platform?

A **social media platform** enables users to create and share content, connect with friends/followers, and consume personalized feeds based on their network and interests.

**Examples:** Facebook, Instagram, Twitter, LinkedIn

### Problem It Solves

1. **Content Sharing:** Post text, images, videos
2. **Social Connections:** Follow/friend other users
3. **Engagement:** Like, comment, share posts
4. **Personalized Feed:** See relevant content from connections
5. **Discovery:** Find new content and people

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **User Onboarding**
   - Register, login, update profile

2. **Content Creation**
   - Create posts (text, image, video)
   - Edit, delete posts

3. **Social Graph**
   - Follow/unfollow users
   - (Friend requests for Facebook - simplified to follow)

4. **Engagement**
   - Like/unlike posts
   - Comment on posts

5. **Feed Generation** ⭐ (Most Critical)
   - View personalized feed of followed users
   - Infinite scrolling

### 2.2 Out of Scope

- ❌ Stories
- ❌ Reels/Short videos
- ❌ Messaging (covered in Lecture 08)

### 2.3 Non-Functional Requirements

1. **Scale**
   - **500 million daily active users**
   - **Millions of posts/day**

2. **CAP Theorem**
   - **Highly Available (AP):** Eventual consistency acceptable
   
   **Why?**
   - ✅ Feed delay of 1-2 mins is acceptable
   - ✅ Downtime = Bad user experience

3. **Low Latency**
   - **Post upload:** < 500ms
   - **Feed load:** < 1 second

---

## 3. Core Entities

1. **User** - Profile, metadata
2. **Post** - Content (text/image/video)
3. **Follower** - Social graph (who follows whom)
4. **Like** - Engagement
5. **Comment** - Engagement
6. **Feed** - Personalized content stream

---

## 4. API Design

### 4.1 User APIs

```
POST /v1/users/register
POST /v1/users/login
GET /v1/users/{userId}
PUT /v1/users/{userId}
```

---

### 4.2 Post APIs

#### **Create Post**

```
POST /v1/posts
```

**Request Body:**
```json
{
  "userId": "user_123",
  "contentType": "IMAGE",  // TEXT, IMAGE, VIDEO
  "contentText": "Beautiful sunset!",
  "mediaUrl": "https://s3.../image.jpg"
}
```

---

#### **Get Post**

```
GET /v1/posts/{postId}
```

---

#### **Edit/Delete Post**

```
PUT /v1/posts/{postId}
DELETE /v1/posts/{postId}
```

---

#### **Get User Feed (Paginated)**

```
GET /v1/feed?page=1&limit=50
```

**Response:**
```json
{
  "posts": [
    {
      "postId": "post_456",
      "userId": "user_789",
      "username": "john_doe",
      "contentType": "IMAGE",
      "contentText": "Beautiful sunset!",
      "mediaUrl": "https://cdn.../image.jpg",
      "likeCount": 234,
      "commentCount": 45,
      "createdAt": "2026-01-21T10:30:00Z"
    }
  ],
  "nextPage": 2
}
```

---

#### **Get User's Posts**

```
GET /v1/users/{userId}/posts?page=1&limit=20
```

---

### 4.3 Engagement APIs

```
POST /v1/posts/{postId}/like
DELETE /v1/posts/{postId}/like

GET /v1/posts/{postId}/comments
POST /v1/posts/{postId}/comments
```

---

### 4.4 Social Graph APIs

```
POST /v1/users/{userId}/follow
DELETE /v1/users/{userId}/unfollow
GET /v1/users/{userId}/followers
GET /v1/users/{userId}/following
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                │  Load Balancer  │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   User   │    │ Content  │    │  Feed    │
  │ Service  │    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ User DB  │    │ Post DB  │    │ Feed     │
  │(Postgres)│    │(Cassandra│    │  Cache   │
  └──────────┘    └──────────┘    └──────────┘


  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Follower  │    │Engagement│    │  Fan-out │
  │ Service  │    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Follower  │    │Like/     │    │  Kafka   │
  │   DB     │    │Comment DB│    │          │
  └──────────┘    └──────────┘    └──────────┘
```

### 5.2 Service Responsibilities

**User Service:**
- Registration, login, profile management

**Content Service:**
- Post creation, upload to S3, content moderation

**Feed Service:**
- Generate personalized feed from cache

**Follower Service:**
- Manage social graph (follow/unfollow)

**Engagement Service:**
- Track likes, comments

**Fan-out Service:**
- Update follower feeds when user posts

---

## 6. Low-Level Design (LLD)

### 6.1 User Service (Simple)

**User DB (PostgreSQL):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    phone VARCHAR(20),
    bio TEXT,
    profile_picture_url VARCHAR(500),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);
```

---

### 6.2 Content Service (Post Creation)

#### **Problem: High Volume**

```
Millions of posts/day
    ↓
Need asynchronous processing
```

#### **Solution: Kafka + Moderator Service**

---

#### **Content Upload Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
│              POST /v1/posts                              │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Content Service │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │  (raw_posts)    │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Moderator     │
                │    Service      │
                │   (AI/ML)       │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Kafka   │    │  Kafka   │    │  Kafka   │
  │(filtered)│    │(blocked) │    │(filtered)│
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │   Post   │    │ Notifi-  │
  │ Consumer │    │ cation   │
  │ Service  │    │ Service  │
  └────┬─────┘    └──────────┘
       │
       ├──────────────┬──────────────┐
       │              │              │
       ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Post DB  │  │    S3    │  │ Fan-out  │
  │(Cassandra│  │ (Media)  │  │ Service  │
  └──────────┘  └──────────┘  └──────────┘
```

---

#### **Flow:**

```
Step 1: User uploads post
    POST /v1/posts {text, image}
    ↓
Step 2: Content Service → Kafka (raw_posts)
    ↓
Step 3: Moderator Service (AI/ML) checks content
    - Violence? Nudity? Spam?
    ↓
Step 4a: If VALID → Kafka (filtered_posts)
Step 4b: If BLOCKED → Kafka (blocked_posts)
    ↓
Step 5a: Post Consumer Service
    - Store in Post DB (Cassandra)
    - Upload media to S3
    ↓
Step 5b: Notification Service
    - Notify user "Post blocked due to policy violation"
```

---

#### **Post DB (Cassandra - Write-Heavy):**

```sql
CREATE TABLE posts (
    post_id UUID PRIMARY KEY,
    user_id UUID,
    post_type VARCHAR(20),  -- TEXT, IMAGE, VIDEO
    content_text TEXT,
    media_url VARCHAR(500),
    thumbnail_url VARCHAR(500),
    like_count BIGINT DEFAULT 0,
    comment_count BIGINT DEFAULT 0,
    share_count BIGINT DEFAULT 0,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX ON posts(user_id);
CREATE INDEX ON posts(created_at);
```

**Why Cassandra?**
- ✅ Write-heavy (millions of posts/day)
- ✅ Fast writes (< 10ms)
- ✅ Horizontal scaling

---

### 6.3 Follower Service (Social Graph)

**Follower DB (PostgreSQL):**

```sql
CREATE TABLE followers (
    follow_id UUID PRIMARY KEY,
    follower_id UUID,  -- User who follows
    following_id UUID,  -- User being followed
    status VARCHAR(20),  -- PENDING, ACCEPTED (for friend requests)
    created_at TIMESTAMP
);

CREATE INDEX idx_follower ON followers(follower_id);
CREATE INDEX idx_following ON followers(following_id);
CREATE UNIQUE INDEX idx_follow_pair ON followers(follower_id, following_id);
```

**Why PostgreSQL (not Graph DB)?**
- ✅ Simple relationship (follower → following)
- ✅ No complex graph queries needed
- ✅ PostgreSQL handles this scale well

**When to use Graph DB?**
- If you need "Friend of Friend" suggestions
- Complex graph traversal

---

### 6.4 Engagement Service (Likes & Comments)

#### **Problem: High Write Volume**

```
Millions of likes/comments per second
    ↓
Need Kafka buffer
```

---

#### **Engagement Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
│         POST /v1/posts/{id}/like                         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │ (likes/comments)│
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Engagement     │
                │    Service      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Like DB  │    │Comment DB│    │ Post DB  │
  │(Postgres)│    │(Cassandra│    │ (Update  │
  └──────────┘    └──────────┘    │  count)  │
                                  └──────────┘
```

---

#### **Like DB (PostgreSQL):**

```sql
CREATE TABLE likes (
    like_id UUID PRIMARY KEY,
    post_id UUID,
    comment_id UUID,  -- NULL if like on post
    user_id UUID,
    reaction_type VARCHAR(20),  -- LIKE, LOVE, HAHA, WOW, SAD, ANGRY
    created_at TIMESTAMP
);

CREATE INDEX idx_post_id ON likes(post_id);
CREATE INDEX idx_user_id ON likes(user_id);
CREATE UNIQUE INDEX idx_user_post ON likes(user_id, post_id);
```

---

#### **Comment DB (Cassandra - Write-Heavy):**

```sql
CREATE TABLE comments (
    comment_id UUID PRIMARY KEY,
    post_id UUID,
    user_id UUID,
    comment_text TEXT,
    like_count BIGINT DEFAULT 0,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX ON comments(post_id);
CREATE INDEX ON comments(user_id);
```

---

#### **Aggregation (Like Count):**

```python
def update_like_count(post_id):
    # Periodically (or via Kafka consumer)
    like_count = db.count("SELECT COUNT(*) FROM likes WHERE post_id = ?", post_id)
    
    # Update Post DB
    db.execute("UPDATE posts SET like_count = ? WHERE post_id = ?", like_count, post_id)
```

---

### 6.5 Feed Service ⭐ (Most Critical!)

#### **Naive Approach (DOESN'T SCALE!):**

```
User opens app
    ↓
Query: SELECT posts FROM posts
       WHERE user_id IN (SELECT following_id FROM followers WHERE follower_id = ?)
       ORDER BY created_at DESC
       LIMIT 50
    ↓
Problem:
- User has 10,000 followers
- Need to JOIN 10,000 users' posts
- Query takes 5+ seconds ❌
```

---

#### **Solution: Fan-out Model (Precompute Feed)**

**Key Concept:** Generate feed BEFORE user opens app!

---

### 6.6 Fan-out Model: Push vs Pull

#### **Push Model (Regular Users):**

```
User A posts content
    ↓
Fan-out Service updates feeds of ALL followers
    ↓
User B opens app → Feed already ready ✅
```

**When to use:**
- User has < 5,000 followers
- Acceptable to update all followers' feeds

---

#### **Pull Model (Celebrities):**

```
Celebrity posts content
    ↓
Do NOT update 10 million followers' feeds ❌
    ↓
User opens app → Query celebrity's recent posts (Backfill)
```

**When to use:**
- User has > 100,000 followers
- Too expensive to update all feeds

---

### 6.7 Feed Generation Architecture

```
┌─────────────────────────────────────────────────────────┐
│              USER POSTS CONTENT                          │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │ (filtered_posts)│
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   Post   │    │ Fan-out  │    │   Post   │
  │ Consumer │    │ Service  │    │Materializer
  └──────────┘    └────┬─────┘    └────┬─────┘
                       │               │
                       │               ▼
                       │         ┌──────────┐
                       │         │ Recent   │
                       │         │Posts     │
                       │         │Cache     │
                       │         │(Redis)   │
                       │         └──────────┘
                       │
                       ├──── Get follower list
                       │
                       ▼
              ┌─────────────────┐
              │  Follower Cache │
              │     (Redis)     │
              │  (Top followers)│
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │     Kafka       │
              │ (feed_updates)  │
              │ {userId, postId}│
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Fan-out       │
              │   Consumer      │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Feed DB  │  │  Feed    │  │          │
  │          │  │  Cache   │  │          │
  │          │  │ (Redis)  │  │          │
  └──────────┘  └──────────┘  └──────────┘
```

---

### 6.8 Feed Components Explained

#### **1. Post Materializer Service**

**Purpose:** Keep cache of recent posts per user

```python
def on_post_created(post):
    user_id = post.user_id
    
    # Store last 100 posts per user in Redis
    redis.lpush(f"recent_posts:{user_id}", post.post_id)
    redis.ltrim(f"recent_posts:{user_id}", 0, 99)  # Keep last 100
    
    # Set expiry
    redis.expire(f"recent_posts:{user_id}", 86400)  # 24 hours
```

**Redis Structure:**
```
recent_posts:user_123 → [post_999, post_998, ..., post_900]
```

---

#### **2. Follower Cache (Top Followers)**

**Purpose:** Store users you interact with most

```python
def update_top_followers(user_id):
    # ML model determines top followers based on:
    # - Like frequency
    # - Comment frequency
    # - View time
    
    top_followers = ml_model.predict_top_followers(user_id, limit=500)
    
    redis.delete(f"top_followers:{user_id}")
    redis.sadd(f"top_followers:{user_id}", *top_followers)
    redis.expire(f"top_followers:{user_id}", 3600)  # 1 hour
```

**Redis Structure:**
```
top_followers:user_123 → {user_456, user_789, user_101, ...}
```

---

#### **3. Fan-out Service (Push Model)**

```python
def fanout_post(post):
    user_id = post.user_id
    
    # Check if user is celebrity
    follower_count = get_follower_count(user_id)
    
    if follower_count > 100000:
        # Celebrity: Skip fan-out (use Pull model)
        return
    
    # Regular user: Fan-out to all followers
    followers = redis.smembers(f"top_followers:{user_id}")
    
    # Publish to Kafka for each follower
    for follower_id in followers:
        kafka.produce("feed_updates", {
            "userId": follower_id,
            "postId": post.post_id
        })
```

---

#### **4. Fan-out Consumer Service**

```python
def consume_feed_updates():
    while True:
        message = kafka.consume("feed_updates")
        user_id = message['userId']
        post_id = message['postId']
        
        # 1. Get post details
        post = db.get_post(post_id)
        
        # 2. Update Feed Cache (Redis)
        redis.lpush(f"feed:{user_id}", post.to_json())
        redis.ltrim(f"feed:{user_id}", 0, 99)  # Keep last 100
        
        # 3. Update Feed DB (persistent storage)
        db.insert_feed(user_id, post_id)
```

**Feed Cache Structure:**
```
feed:user_123 → [post_999_json, post_998_json, ..., post_900_json]
```

---

#### **5. Feed Service (User Opens App)**

```python
def get_feed(user_id, page=1, limit=50):
    # Try to fetch from Feed Cache (Redis)
    cached_feed = redis.lrange(f"feed:{user_id}", 0, limit-1)
    
    if cached_feed:
        return cached_feed  # Fast! < 10ms
    
    # Fallback: Fetch from Feed DB
    feed = db.query("""
        SELECT * FROM feed
        WHERE user_id = ?
        ORDER BY created_at DESC
        LIMIT ?
    """, user_id, limit)
    
    return feed
```

---

### 6.9 Backfill Service (Real-Time Feed Generation)

#### **When is Backfill Triggered?**

```
User scrolls past precomputed feed (100 posts)
    ↓
Client detects: "Reached end of cached feed"
    ↓
Trigger Backfill API
```

---

#### **Backfill Flow:**

```
┌─────────────────────────────────────────────────────────┐
│              USER SCROLLS PAST CACHED FEED               │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │Backfill Service │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Follower │    │ Recent   │    │          │
  │  Cache   │    │  Posts   │    │          │
  │ (Redis)  │    │  Cache   │    │          │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       └───────┬───────┘
               │
               ▼
      ┌─────────────────┐
      │  Generate Feed  │
      │   (In-memory)   │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │     Kafka       │
      │ (feed_updates)  │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │   Fan-out       │
      │   Consumer      │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │  Feed Cache     │
      │   (Updated)     │
      └─────────────────┘
```

---

#### **Backfill Implementation:**

```python
def backfill_feed(user_id):
    # Step 1: Get top followers (from cache)
    top_followers = redis.smembers(f"top_followers:{user_id}")
    
    # Step 2: For each follower, get recent posts (from cache)
    posts = []
    for follower_id in top_followers:
        recent_post_ids = redis.lrange(f"recent_posts:{follower_id}", 0, 9)  # Last 10 posts
        
        for post_id in recent_post_ids:
            post = db.get_post(post_id)  # Or cache
            posts.append(post)
    
    # Step 3: Sort by timestamp, rank by ML model
    posts.sort(key=lambda p: p.created_at, reverse=True)
    ranked_posts = ranking_algorithm(posts)  # ML-based ranking
    
    # Step 4: Publish to Kafka for caching
    for post in ranked_posts[:50]:
        kafka.produce("feed_updates", {
            "userId": user_id,
            "postId": post.post_id
        })
    
    # Step 5: Return immediately to user
    return ranked_posts[:50]
```

---

### 6.10 Complete Feed Flow

#### **Scenario 1: Regular User Posts**

```
User A (1,000 followers) posts photo
    ↓
Step 1: Content Service → Kafka (raw_posts)
    ↓
Step 2: Moderator Service → Kafka (filtered_posts)
    ↓
Step 3: Post Consumer → Post DB + S3
    ↓
Step 4: Fan-out Service
    - Fetch followers from Follower Cache
    - Publish to Kafka: {follower_1, post_id}, {follower_2, post_id}, ...
    ↓
Step 5: Fan-out Consumer (for each follower)
    - Update Feed Cache
    - Update Feed DB
    ↓
Result: All 1,000 followers see post when they open app ✅
```

---

#### **Scenario 2: Celebrity Posts**

```
Celebrity (10M followers) posts video
    ↓
Step 1-3: Same (Content → Moderator → Post DB)
    ↓
Step 4: Fan-out Service
    - Detects follower_count > 100,000
    - SKIP fan-out (too expensive!)
    ↓
Step 5: Post Materializer
    - Add to Recent Posts Cache
    ↓
Result: Followers see via Backfill (Pull model) ✅
```

---

#### **Scenario 3: User Opens App**

```
User B opens app
    ↓
Feed Service checks Feed Cache
    ↓
Cache HIT: Return 100 precomputed posts (< 10ms) ✅
    ↓
User scrolls past 100 posts
    ↓
Backfill Service triggered
    - Fetch top followers (cache)
    - Fetch recent posts (cache)
    - Generate next 50 posts
    - Return + cache for next time
```

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                │  Load Balancer  │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│   User   │      │ Content  │          │  Feed    │
│ Service  │      │ Service  │          │ Service  │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     ▼                 ▼                     ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│ User DB  │      │  Kafka   │          │  Feed    │
│(Postgres)│      │          │          │  Cache   │
└──────────┘      └────┬─────┘          └──────────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Moderator │  │ Fan-out  │  │   Post   │
  │ Service  │  │ Service  │  │Materializer
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │
       ▼             ▼             ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Post      │  │Follower  │  │ Recent   │
  │Consumer  │  │  Cache   │  │Posts     │
  └────┬─────┘  └──────────┘  │Cache     │
       │                      └──────────┘
       ├─────────┬─────────┐
       │         │         │
       ▼         ▼         ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Post DB  │ │    S3    │ │Engagement│
  │(Cassandra│ │ (Media)  │ │ Service  │
  └──────────┘ └──────────┘ └────┬─────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
               ┌──────────┐ ┌──────────┐ ┌──────────┐
               │ Like DB  │ │Comment DB│ │Follower  │
               │(Postgres)│ │(Cassandra│ │   DB     │
               └──────────┘ └──────────┘ └──────────┘
```

---

## 8. Interview Q&A

### Q1: Why Cassandra for Post DB?

**A:**

**Requirements:**
- Millions of posts/day
- Write-heavy workload

**Why Cassandra?**
- ✅ Optimized for writes (< 10ms)
- ✅ Horizontal scaling
- ✅ High availability

### Q2: Why precompute feeds? Why not generate on-demand?

**A:**

**On-Demand Problem:**
```
User has 10,000 followers
Query JOIN 10,000 users' posts
    ↓
Takes 5+ seconds ❌
```

**Precompute Solution:**
```
Feed already in cache
    ↓
Return in < 10ms ✅
```

### Q3: Fan-out Push vs Pull - when to use which?

**A:**

| Model | When | Why |
|-------|------|-----|
| **Push** | < 5,000 followers | Fast feed delivery |
| **Pull** | > 100,000 followers | Too expensive to update all |

**Hybrid:**
- Push for regular users
- Pull (Backfill) for celebrities

### Q4: What if user follows celebrity?

**A:**

**Flow:**
```
User opens app
    ↓
Feed Cache has posts from regular users (Push)
    ↓
Backfill Service fetches celebrity posts (Pull)
    ↓
Merge both feeds
```

### Q5: How to rank feed posts?

**A:**

**Ranking Factors:**
1. **Recency:** Newer posts ranked higher
2. **Engagement:** More likes/comments = higher
3. **User Affinity:** Posts from close friends ranked higher
4. **Content Type:** Video > Image > Text
5. **ML Model:** Predicted engagement

```python
def rank_posts(posts, user_id):
    for post in posts:
        score = 0
        score += recency_score(post)
        score += engagement_score(post)
        score += affinity_score(user_id, post.user_id)
        post.rank_score = score
    
    return sorted(posts, key=lambda p: p.rank_score, reverse=True)
```

### Q6: How to handle Backfill latency?

**A:**

**Optimizations:**
1. **Cache Everything:**
   - Top followers in Redis
   - Recent posts in Redis
2. **Pre-fetch:**
   - Start backfill when user scrolls to 80% of cached feed
3. **Limit:**
   - Only fetch from top 500 followers

### Q7: Why Kafka for engagement (likes/comments)?

**A:**

**Benefits:**
1. **Buffer:** Handle millions of likes/second
2. **Guaranteed Delivery:** No lost likes
3. **Decoupling:** Engagement service can process at own pace

### Q8: How to prevent duplicate likes?

**A:**

**DB Constraint:**
```sql
CREATE UNIQUE INDEX idx_user_post ON likes(user_id, post_id);
```

**Application Logic:**
```python
def like_post(user_id, post_id):
    try:
        db.insert_like(user_id, post_id)
    except DuplicateKeyError:
        # Already liked, ignore
        pass
```

### Q9: How to scale Feed Service?

**A:**

**Strategies:**
1. **Shard Feed Cache by user_id:**
   - user_id % 100 → Cache shard
2. **Horizontal Scaling:**
   - Add more Feed Service instances
3. **CDN:**
   - Cache static content (images/videos)

### Q10: Security & Privacy?

**A:**

1. **Private Accounts:** Only followers see posts
2. **Blocked Users:** Filter from feed
3. **Content Moderation:** AI/ML to detect harmful content
4. **Rate Limiting:** Prevent spam
5. **GDPR:** Data deletion, export

---

## 9. Key Takeaways

✅ **Fan-out Model:** Push (regular users) + Pull (celebrities)  
✅ **Precompute Feeds:** Don't generate on-demand  
✅ **Kafka:** Buffer for high write volume (posts, likes, comments)  
✅ **Redis Caches:** Feed Cache, Top Followers, Recent Posts  
✅ **Cassandra:** Write-heavy (posts, comments)  
✅ **PostgreSQL:** Relational data (users, followers, likes)  
✅ **Backfill Service:** Real-time feed generation when cache empty  
✅ **Content Moderation:** AI/ML to filter harmful content  
✅ **Eventual Consistency:** Feed delay of 1-2 mins acceptable  
✅ **S3 + CDN:** Store and serve media files  

---

## Summary

**Architecture Highlights:**
- 10+ microservices
- 6 databases (PostgreSQL, Cassandra)
- Redis for caching (Feed, Followers, Posts)
- Kafka for async processing
- S3 for media storage

**Feed Generation:**
```
Post → Moderator → Fan-out → Update Follower Feeds → User sees in cache
```

**Backfill (Pull):**
```
User scrolls → Backfill Service → Fetch from caches → Generate feed → Return + cache
```

**Key Optimizations:**
- Precompute feeds (Fan-out Push)
- Cache recent posts (Redis)
- Cache top followers (Redis)
- Backfill for celebrities (Pull)
- Kafka for async processing

**Performance:**
- Feed load: < 1 second (from cache)
- Post upload: < 500ms
- 500M daily active users

**End of Lecture 13**
