# Lecture 5: Hotel Booking System Design (like Airbnb)

> **Lecture:** 05  
> **Topic:** System Design  
> **Application:** Hotel Booking System (Airbnb, Booking.com, MakeMyTrip)  
> **Scale:** 50M Users, 1M Hotels  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-04-OTT-Platform-System-Design|Lecture 4: OTT Platform]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Architecture Diagram](#7-complete-architecture-diagram)
8. [Interview-Style Q&A](#8-interview-style-qa)
9. [Key Takeaways](#9-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Hotel Booking System?

A **hotel booking platform** allows users to search for hotels based on location and dates, view available rooms, and make reservations.

**Examples:** Airbnb, Booking.com, MakeMyTrip, Expedia, Hotels.com

### Problem It Solves

1. **Hotel Discovery:** Find hotels by location, name, dates
2. **Room Availability:** Check real-time room availability
3. **Secure Booking:** Reserve rooms with payment
4. **Booking Management:** View past and upcoming bookings
5. **Reviews:** Read and write hotel reviews

### Interview Expectations

⚠️ **Important Context:**

Since this problem is **straightforward**, the interviewer expects:
- ✅ **Detailed design** of each component
- ✅ **Deep dive** into search optimization
- ✅ **Concurrency control** (prevent double booking)
- ✅ **Database schema** design
- ❌ Surface-level architecture (not enough!)

**Key Challenge:** **No two users should book the same room for overlapping dates!**

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **User Registration & Login**
   - Create account
   - Login/logout
   - Update profile

2. **Search Hotels**
   - Search by location (city, area)
   - Search by hotel name
   - Search by date range (check-in, check-out)

3. **View Hotel Details**
   - Hotel metadata (name, address, amenities)
   - Available rooms for selected dates
   - Room prices (varies by date)
   - Hotel images

4. **Book Room**
   - Select room
   - Make payment
   - Confirm booking

5. **View Booking History**
   - Past bookings
   - Upcoming bookings

6. **Submit Reviews**
   - Rate hotel
   - Write comments
   - Upload images

### 2.2 Non-Functional Requirements

1. **Scale**
   - **50 million users** globally
   - **1 million hotels** (listings)
   - **High traffic** during peak seasons

2. **CAP Theorem - Split Approach**

   **High Availability (Search):**
   - ✅ Search should always work
   - ✅ Users can browse hotels anytime
   - ❌ Slight delay in availability updates is acceptable

   **High Consistency (Booking):**
   - ✅ **No double booking** (critical!)
   - ✅ Payment must be consistent
   - ✅ Room availability must be accurate

   **Summary:** **AP for search, CP for booking**

3. **Low Latency**
   - Search results: < 500ms
   - Booking confirmation: < 2 seconds

4. **Concurrency Control**
   - **Critical:** Prevent race conditions
   - **Scenario:** Two users booking same room simultaneously

---

## 3. Core Entities

1. **User** - Account holder
2. **Hotel** - Property listing
3. **Room** - Individual rooms in a hotel
4. **Booking** - Reservation record
5. **Payment** - Transaction details
6. **Review** - User feedback

---

## 4. API Design

### 4.1 User Registration

```
POST /v1/users/register
```

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "hashed_password",
  "phone": "+1234567890"
}
```

**Additional APIs** (mention verbally):
- `POST /v1/users/login` - Login
- `POST /v1/users/logout` - Logout
- `PUT /v1/users/profile` - Update profile

---

### 4.2 Search Hotels

```
GET /v1/hotels/search?name={name}&location={location}&checkIn={date}&checkOut={date}
```

**Query Parameters:**
- `name` (optional): Hotel name
- `location` (required): City or area
- `checkIn` (required): Check-in date (YYYY-MM-DD)
- `checkOut` (required): Check-out date (YYYY-MM-DD)

**Response:**
```json
{
  "hotels": [
    {
      "hotelId": "hotel_123",
      "name": "Grand Plaza Hotel",
      "location": "Bangalore, India",
      "thumbnail": "https://cdn.example.com/hotel_123.jpg",
      "rating": 4.5,
      "priceFrom": 2000
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 10
  }
}
```

⚠️ **Important:** Use pagination!

---

### 4.3 Get Hotel Details

```
GET /v1/hotels/{hotelId}?checkIn={date}&checkOut={date}
```

**Response:**
```json
{
  "hotelId": "hotel_123",
  "name": "Grand Plaza Hotel",
  "address": "123 MG Road, Bangalore",
  "location": {
    "latitude": 12.9716,
    "longitude": 77.5946
  },
  "amenities": ["WiFi", "Pool", "Gym"],
  "images": ["url1", "url2"],
  "rating": 4.5,
  "availableRooms": [
    {
      "roomId": "room_456",
      "type": "Deluxe",
      "capacity": 2,
      "price": 2500,
      "currency": "INR"
    }
  ]
}
```

⚠️ **Key Point:** Date is required to show available rooms and prices!

---

### 4.4 Book Room

```
POST /v1/bookings
```

**Request Body:**
```json
{
  "hotelId": "hotel_123",
  "roomId": "room_456",
  "checkIn": "2026-02-01",
  "checkOut": "2026-02-05",
  "userId": "user_789"  // From JWT token
}
```

**Response:**
```json
{
  "bookingId": "booking_101",
  "status": "confirmed",
  "amount": 10000,
  "currency": "INR"
}
```

⚠️ **Note:** This is a **two-step process** (book → pay)

---

### 4.5 Get Booking History

```
GET /v1/users/{userId}/bookings
```

**Response:**
```json
{
  "bookings": [
    {
      "bookingId": "booking_101",
      "hotelName": "Grand Plaza Hotel",
      "checkIn": "2026-02-01",
      "checkOut": "2026-02-05",
      "status": "confirmed",
      "amount": 10000
    }
  ]
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  50M Global Users                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                │                 │
                │ - Auth/AuthZ    │
                │ - Routing       │
                │ - Rate Limiting │
                │ - Round Robin   │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┬──────────┐
        │                │                │          │
        ▼                ▼                ▼          ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Search  │    │ Booking  │ │  Review  │
  │ Service  │    │ Service  │    │ Service  │ │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘ └────┬─────┘
       │               │                │            │
       ▼               ▼                ▼            ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Hotel   │    │ Booking  │ │  Review  │
  │   DB     │    │   DB     │    │   DB     │ │   DB     │
  │ (MySQL)  │    │(Postgres)│    │ (MySQL)  │ │ (MySQL)  │
  └──────────┘    └──────────┘    └──────────┘ └──────────┘
```

### 5.2 Service Responsibilities

**1. User Service:**
- Registration, login, logout
- JWT token generation
- Profile management

**2. Search Service:**
- Query hotels by location, name, dates
- Return available hotels

**3. Booking Service:**
- Handle room reservations
- Call payment service
- Ensure no double booking

**4. Review Service:**
- Store and retrieve reviews
- Calculate ratings

---

## 6. Low-Level Design (LLD)

### 6.1 User Service

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    phone VARCHAR(20),
    created_at TIMESTAMP
);
```

**Flow:**
1. User registers → Store in MySQL
2. User logs in → Validate → Return JWT token
3. JWT token used in all subsequent API calls

---

### 6.2 Hotel Database (PostgreSQL)

**Why PostgreSQL?**
- **Relational data:** Hotels → Rooms → Availability
- **ACID compliance:** Critical for booking consistency
- **GIS extension:** Supports geolocation search

#### **Table 1: Hotels**

```sql
CREATE TABLE hotels (
    hotel_id UUID PRIMARY KEY,
    name VARCHAR(255),
    address TEXT,
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    amenities TEXT[],  -- Array: ['WiFi', 'Pool', 'Gym']
    created_at TIMESTAMP
);
```

#### **Table 2: Rooms**

```sql
CREATE TABLE rooms (
    room_id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(hotel_id),
    type VARCHAR(50),  -- Deluxe, Standard, Suite
    capacity INT,      -- Number of guests
    description TEXT,
    created_at TIMESTAMP
);
```

#### **Table 3: Room Prices**

**Why separate table?**
- Prices **vary by date** (festivals, weekends, peak season)
- Flexible pricing strategy

```sql
CREATE TABLE room_prices (
    price_id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(hotel_id),
    room_id UUID REFERENCES rooms(room_id),
    price DECIMAL(10, 2),
    currency VARCHAR(3),
    start_date DATE,
    end_date DATE
);
```

**Example:**
```
Room 101 → Jan 1 to Jan 31 → ₹2000/night
Room 101 → Feb 1 to Feb 14 → ₹3000/night (Valentine's Day)
Room 101 → Feb 15 to Feb 28 → ₹2000/night
```

#### **Table 4: Room Availability (Two Approaches)**

**Approach 1: Daily Records (Optimized for Search)**

```sql
CREATE TABLE room_availability_daily (
    availability_id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(hotel_id),
    room_id UUID REFERENCES rooms(room_id),
    date DATE,
    status VARCHAR(20)  -- available, booked
);
```

**Pros:**
- ✅ **Fast search:** Simple query (`WHERE date BETWEEN ? AND ?`)
- ✅ **No computation:** Direct lookup

**Cons:**
- ❌ **High storage:** 365 records/room/year
- ❌ **Example:** 300 rooms × 365 days = 109,500 records/hotel

**When to use:** High search frequency, storage not a concern

---

**Approach 2: Date Ranges (Optimized for Storage)**

```sql
CREATE TABLE room_availability_range (
    availability_id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(hotel_id),
    room_id UUID REFERENCES rooms(room_id),
    start_date DATE,
    end_date DATE,
    status VARCHAR(20)  -- available, booked
);
```

**Example:**
```
Room 101 → Jan 1 to Mar 31 → available
Room 101 → Apr 1 to Apr 5 → booked
Room 101 → Apr 6 to Dec 31 → available
```

**Pros:**
- ✅ **Low storage:** Only store ranges
- ✅ **Efficient updates:** Merge adjacent ranges

**Cons:**
- ❌ **Complex queries:** Range overlap checks
- ❌ **Update complexity:** Split/merge ranges on booking

**When to use:** Storage-constrained, fewer searches

---

**Recommendation:** **Approach 1 (Daily Records)** for hotel booking (search-heavy)

---

### 6.3 Image Storage (S3 Bucket)

**What to store:**
- Hotel images
- Room images
- Review images (user-uploaded)

**Why S3?**
- **Scalable:** Handle millions of images
- **CDN integration:** Fast delivery
- **Cost-effective:** Pay per use

**Flow:**
```
Upload Image → S3 Bucket → CDN → User
```

---

### 6.4 Search Service (The Core!)

#### **Challenge: Multi-Criteria Search**

Users search by:
1. **Location** (proximity search)
2. **Hotel name** (text search)
3. **Date range** (availability check)

**Problem:** Joining 5+ tables is **slow!**

---

#### **Solution: Elasticsearch + Redis Cache**

```
Search Request
    │
    ▼
Search Service
    │
    ▼
Elasticsearch (Location + Name)
    │
    ▼
Get matching hotel IDs (e.g., 100 hotels)
    │
    ▼
Redis Cache (Rooms + Prices + Availability)
    │
    ▼
Join data → Return results
```

---

#### **Proximity Search: 4 Approaches**

**1. Elasticsearch (Geospatial)**
- ✅ Built-in geospatial queries
- ✅ Supports text search
- ✅ Fast (< 100ms)
- ❌ Requires indexing

**2. Quad Tree**
- ✅ Very fast (in-memory)
- ❌ Requires local cache (high memory)
- ❌ No text search support
- ❌ Complex implementation

**3. PostgreSQL + GIS Extension**
- ✅ No extra infrastructure
- ✅ Geospatial indexing
- ❌ Slower than Elasticsearch
- ❌ No text search optimization

**4. Geohash**
- ✅ Efficient encoding
- ✅ Proximity grouping
- ❌ No text search support
- ❌ Requires custom implementation

---

#### **Why Elasticsearch Wins?**

| Feature | Elasticsearch | Quad Tree | PostgreSQL GIS | Geohash |
|---------|---------------|-----------|----------------|---------|
| Text Search | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Geo Search | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| Speed | ⚡ Fast | ⚡⚡ Fastest | 🐢 Slow | ⚡ Fast |
| Memory | Low | High | Low | Low |
| Complexity | Low | High | Medium | Medium |

**Decision:** **Elasticsearch** (supports both text + geo search)

⚠️ **Note:** Elasticsearch internally uses **Geohash** for geospatial indexing!

---

#### **Quad Tree Explained (Brief)**

**Concept:** Recursively divide world map into 4 quadrants

```
┌─────────────────────────────────┐
│ NW          │          NE       │
│             │                   │
│      ┌──────┼──────┐            │
│      │ NW   │  NE  │            │
│──────┼──────┼──────┼────────────│
│      │ SW   │  SE  │            │
│      └──────┼──────┘            │
│             │                   │
│ SW          │          SE       │
└─────────────────────────────────┘
```

**How it works:**
1. Divide world into 4 quadrants
2. Recursively divide each quadrant
3. Stop when area < 500m
4. Store hotels in leaf nodes
5. Search: Check current + 8 neighboring quadrants

**When to use:** Uber/Ola (real-time driver location)

---

#### **Search Service Implementation**

**Step 1: Elasticsearch (Hotel + Location)**

**Index Structure:**
```json
{
  "hotelId": "hotel_123",
  "name": "Grand Plaza Hotel",
  "location": {
    "lat": 12.9716,
    "lon": 77.5946
  }
}
```

**Query:**
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Grand Plaza"
          }
        },
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 12.9716,
              "lon": 77.5946
            }
          }
        }
      ]
    }
  }
}
```

**Result:** List of hotel IDs (e.g., 100 hotels)

---

**Step 2: Redis Cache (Rooms + Prices + Availability)**

**Why Redis?**
- **Fast:** In-memory (< 10ms)
- **Reduce DB load:** Avoid joining 5 tables
- **TTL:** Auto-expire stale data

**Cached Data:**
```json
{
  "hotel_123": {
    "rooms": [
      {
        "roomId": "room_456",
        "type": "Deluxe",
        "capacity": 2
      }
    ],
    "prices": {
      "2026-02-01": 2500,
      "2026-02-02": 2500
    },
    "availability": {
      "2026-02-01": "available",
      "2026-02-02": "available"
    }
  }
}
```

**Flow:**
1. Elasticsearch returns 100 hotel IDs
2. For each hotel ID, query Redis
3. Join room + price + availability data
4. Return to user

---

**Step 3: CDC Pipeline (PostgreSQL → Elasticsearch)**

**Problem:** How to keep Elasticsearch in sync with PostgreSQL?

**Solution:** CDC (Change Data Capture)

```
PostgreSQL (Hotels table)
    │
    │ CDC (Debezium)
    ▼
Kafka
    │
    ▼
Consumer Service
    │
    ▼
Elasticsearch
```

**What gets synced:**
- Hotel name
- Location (lat/lon)
- Amenities

**What doesn't get synced:**
- Rooms (stored in Redis)
- Prices (stored in Redis)
- Availability (stored in Redis)

---

### 6.5 Booking Service (Concurrency Control!)

#### **Challenge: Prevent Double Booking**

**Scenario:**
- **User A** books Room 101 at **T=0s**
- **User B** books Room 101 at **T=5s**
- **Problem:** Both see room as available!

**Solution:** **Two-phase verification + Redis Lock**

---

#### **Architecture:**

```
User clicks "Book"
    │
    ▼
Booking Service
    │
    ├─> Check availability (Availability Service)
    │   │
    │   └─> Query PostgreSQL (room_availability table)
    │       │
    │       └─> If available → Acquire Redis Lock
    │
    ├─> Redirect to Payment Service
    │   │
    │   └─> Payment Gateway (Razorpay, Stripe)
    │       │
    │       └─> Payment Success/Failure
    │
    └─> Publish event to Kafka
        │
        ├─> Notification Consumer → Send email
        │
        ├─> Booking Consumer → Update Booking DB
        │
        └─> Availability Consumer → Update Hotel DB
```

---

#### **Step 1: Availability Check**

**Availability Service:**

```python
def check_availability(hotel_id, room_id, check_in, check_out):
    # Query PostgreSQL
    query = """
        SELECT * FROM room_availability_daily
        WHERE hotel_id = ? AND room_id = ?
        AND date BETWEEN ? AND ?
        AND status = 'available'
    """
    
    results = db.execute(query, [hotel_id, room_id, check_in, check_out])
    
    # All dates must be available
    required_days = (check_out - check_in).days
    if len(results) == required_days:
        return True
    else:
        return False
```

---

#### **Step 2: Redis Lock (Prevent Race Condition)**

**Problem:**
- **User A** checks availability at **T=0s** → Available ✅
- **User B** checks availability at **T=0.1s** → Available ✅
- **Both proceed to payment!** ❌

**Solution:** **Distributed Lock with Redis**

```python
def acquire_lock(hotel_id, room_id, check_in, check_out):
    lock_key = f"lock:{hotel_id}:{room_id}:{check_in}:{check_out}"
    
    # Try to acquire lock (TTL = 5 minutes)
    success = redis.set(lock_key, "locked", nx=True, ex=300)
    
    if success:
        return True  # Lock acquired
    else:
        return False  # Already locked by another user
```

**Flow:**
1. User A checks availability → Available → **Acquire lock** ✅
2. User B checks availability → Available → **Try to acquire lock** → **Fails** ❌
3. User B gets error: "Room is being booked by another user"

**TTL = 5 minutes:**
- If User A doesn't complete payment in 5 minutes → Lock expires
- Room becomes available again

---

#### **Step 3: Payment Service**

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    booking_id UUID,
    amount DECIMAL(10, 2),
    currency VARCHAR(3),
    status VARCHAR(20),  -- success, failed, pending
    transaction_id VARCHAR(255),  -- From payment gateway
    created_at TIMESTAMP
);
```

**Flow:**
1. User enters card details
2. Payment Service calls **Payment Gateway** (Razorpay, Stripe)
3. Gateway processes payment
4. Gateway returns success/failure
5. Update `payments` table

---

#### **Step 4: Event-Driven Updates (Kafka)**

**Why Kafka?**
- **Decoupled:** Booking service doesn't directly update all DBs
- **Reliable:** Guaranteed message delivery
- **Asynchronous:** No blocking

**Event:**
```json
{
  "eventType": "BOOKING_CONFIRMED",
  "bookingId": "booking_101",
  "userId": "user_789",
  "hotelId": "hotel_123",
  "roomId": "room_456",
  "checkIn": "2026-02-01",
  "checkOut": "2026-02-05",
  "amount": 10000
}
```

**Consumers:**

**1. Notification Consumer:**
- Send email: "Booking confirmed!"
- Send SMS

**2. Booking Consumer:**
- Insert record into `bookings` table

**3. Availability Consumer:**
- Update `room_availability_daily` table
- Set status = 'booked' for dates

---

### 6.6 Booking Database

**Schema:**
```sql
CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    hotel_id UUID REFERENCES hotels(hotel_id),
    room_ids UUID[],  -- Array: User can book multiple rooms
    check_in DATE,
    check_out DATE,
    amount DECIMAL(10, 2),
    currency VARCHAR(3),
    status VARCHAR(20),  -- confirmed, cancelled, completed
    created_at TIMESTAMP
);
```

---

### 6.7 Review Service

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE reviews (
    review_id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(hotel_id),
    user_id UUID REFERENCES users(user_id),
    rating INT,  -- 1-5
    comment TEXT,
    images TEXT[],  -- S3 URLs
    created_at TIMESTAMP
);
```

**Flow:**
1. User submits review
2. Store text in MySQL
3. Store images in S3
4. Update hotel rating (average)

---

### 6.8 Booking Info Service

**Purpose:** Fetch user's booking history

**Query:**
```sql
SELECT * FROM bookings
WHERE user_id = ?
ORDER BY created_at DESC;
```

**Simple and straightforward!**

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    50M Users                             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┬──────────┐
        │                │                │          │
        ▼                ▼                ▼          ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Search  │    │ Booking  │ │  Review  │
  │ Service  │    │ Service  │    │ Service  │ │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘ └────┬─────┘
       │               │                │            │
       ▼               ▼                │            ▼
  ┌──────────┐    ┌──────────┐         │       ┌──────────┐
  │ User DB  │    │Elastic   │         │       │ Review   │
  │ (MySQL)  │    │ Search   │         │       │   DB     │
  └──────────┘    └────▲─────┘         │       └──────────┘
                       │               │
                       │ CDC           │
                       │               │
                  ┌────┴─────┐         │
                  │  Hotel   │◄────────┤
                  │   DB     │         │
                  │(Postgres)│         │
                  └────▲─────┘         │
                       │               │
                  ┌────┴─────┐         │
                  │  Redis   │◄────────┤
                  │  Cache   │         │
                  └──────────┘         │
                                       │
                  ┌──────────┐         │
                  │    S3    │         │
                  │  Bucket  │         │
                  └──────────┘         │
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Availability    │
                              │   Service       │
                              └────────┬────────┘
                                       │
                                  ┌────┴─────┐
                                  │  Redis   │
                                  │  Lock    │
                                  └──────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   Payment       │
                              │   Service       │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  Payment        │
                              │  Gateway        │
                              └────────┬────────┘
                                       │
                                       ▼
                                  Kafka Broker
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
              ┌──────────┐      ┌──────────┐      ┌──────────┐
              │Notifica  │      │ Booking  │      │Availabil │
              │   -tion  │      │ Consumer │      │   -ity   │
              │ Consumer │      │          │      │ Consumer │
              └──────────┘      └────┬─────┘      └────┬─────┘
                                     │                  │
                                     ▼                  ▼
                                ┌──────────┐      ┌──────────┐
                                │ Booking  │      │  Hotel   │
                                │   DB     │      │   DB     │
                                │ (MySQL)  │      │(Postgres)│
                                └──────────┘      └──────────┘
```

---

## 8. Interview-Style Q&A

### Q1: Why split CAP theorem (AP for search, CP for booking)?

**A:**
- **Search:** Users browsing hotels → Availability matters more
  - If availability data is 5 minutes stale → Acceptable
  - Better than search being down
  
- **Booking:** Money involved → Consistency critical
  - **Cannot** allow double booking
  - **Must** ensure payment accuracy

### Q2: How to prevent double booking?

**A:**
**Two-phase approach:**

1. **Availability Check:**
   - Query PostgreSQL for room availability
   - If unavailable → Return error immediately

2. **Redis Lock:**
   - If available → Acquire distributed lock
   - Lock key: `lock:{hotel}:{room}:{dates}`
   - TTL: 5 minutes
   - If lock fails → Another user is booking

3. **Payment:**
   - User completes payment within 5 minutes
   - On success → Update availability

4. **Lock Expiry:**
   - If payment not completed in 5 minutes → Lock expires
   - Room becomes available again

### Q3: Why two approaches for availability table?

**A:**

**Approach 1 (Daily Records):**
- ✅ Fast search (simple `BETWEEN` query)
- ❌ High storage (365 records/room/year)
- **Use when:** Search frequency is high

**Approach 2 (Date Ranges):**
- ✅ Low storage (only store ranges)
- ❌ Complex queries (range overlap)
- ❌ Complex updates (split/merge ranges)
- **Use when:** Storage is expensive

**Recommendation:** **Approach 1** for hotel booking (search-heavy)

### Q4: Why Elasticsearch over Quad Tree for proximity search?

**A:**

| Requirement | Elasticsearch | Quad Tree |
|-------------|---------------|-----------|
| Text search (hotel name) | ✅ Yes | ❌ No |
| Geo search (location) | ✅ Yes | ✅ Yes |
| Memory usage | Low | High (entire tree in cache) |
| Implementation | Simple | Complex |

**Decision:** Elasticsearch (supports both text + geo)

**When to use Quad Tree:** Uber/Ola (real-time driver location, no text search needed)

### Q5: Why Redis cache for rooms/prices/availability?

**A:**
- **Problem:** Joining 5 tables (hotels, rooms, prices, availability) is slow
- **Solution:** Cache denormalized data in Redis
- **Benefit:** Reduce query time from 500ms → 50ms

**Flow:**
1. Elasticsearch returns 100 hotel IDs
2. For each hotel, query Redis (parallel)
3. Join data → Return results

### Q6: How to handle payment failures?

**A:**
**Scenario:** User acquires lock → Payment fails

**Solution:**
1. Payment fails → Release Redis lock immediately
2. Don't wait for TTL expiry
3. Room becomes available for others

**Implementation:**
```python
try:
    payment_status = payment_gateway.process()
    if payment_status == "failed":
        redis.delete(lock_key)  # Release lock
except Exception as e:
    redis.delete(lock_key)  # Release lock
```

### Q7: Why Kafka for booking updates?

**A:**
**Without Kafka (Direct DB updates):**
- Booking Service updates 3 DBs directly
- If one update fails → Inconsistency
- Tight coupling

**With Kafka (Event-driven):**
- ✅ Decoupled: Booking Service only publishes event
- ✅ Reliable: Kafka guarantees delivery
- ✅ Retry: Consumers can retry on failure
- ✅ Asynchronous: No blocking

### Q8: How to handle high traffic during peak season?

**A:**
**Strategies:**

1. **Horizontal Scaling:**
   - Add more instances of Booking Service
   - Load balancer distributes traffic

2. **Redis Lock:**
   - Prevents race conditions even with 100 instances

3. **Database Sharding:**
   - Shard Hotel DB by location (India, US, Europe)
   - Reduce load per shard

4. **CDN for Images:**
   - Serve hotel images from CDN
   - Reduce S3 load

### Q9: How to calculate hotel ratings?

**A:**
**Two approaches:**

**Approach 1: Calculate on-the-fly**
```sql
SELECT AVG(rating) FROM reviews WHERE hotel_id = ?
```
- ❌ Slow for hotels with 10,000+ reviews

**Approach 2: Precompute (Recommended)**
- Store `average_rating` and `review_count` in `hotels` table
- Update on each new review:
  ```python
  new_avg = (old_avg * old_count + new_rating) / (old_count + 1)
  ```
- ✅ Fast (no aggregation needed)

### Q10: Security considerations?

**A:**
1. **JWT Tokens:** Authenticate all API calls
2. **HTTPS:** Encrypt communication
3. **PCI Compliance:** For payment data
4. **Rate Limiting:** Prevent abuse (API Gateway)
5. **SQL Injection:** Use parameterized queries
6. **Redis Lock Expiry:** Prevent deadlocks

---

## 9. Key Takeaways

✅ **Split CAP:** AP for search, CP for booking  
✅ **Elasticsearch:** Text + geo search  
✅ **Redis Lock:** Prevent double booking  
✅ **Event-driven:** Kafka for decoupled updates  
✅ **Two-phase verification:** Check availability + acquire lock  
✅ **Daily records:** Better for search-heavy workloads  
✅ **Redis cache:** Denormalize data for fast queries  
✅ **S3 + CDN:** Efficient image delivery  

---

## Summary

**Architecture Highlights:**
- 6 microservices (User, Search, Booking, Payment, Availability, Review)
- 5 databases (MySQL x3, PostgreSQL x1, Elasticsearch x1)
- Redis for caching + distributed locks
- Kafka for event-driven updates
- S3 for image storage

**Concurrency Control:**
- Availability check (PostgreSQL)
- Redis distributed lock (TTL = 5 minutes)
- Two-phase commit prevention

**Search Optimization:**
- Elasticsearch for text + geo search
- Redis cache for rooms/prices/availability
- CDC pipeline for data sync

**Performance:**
- Search: < 500ms
- Booking: < 2 seconds
- Zero double bookings

**Scale:**
- 50M users globally
- 1M hotels
- Horizontal scaling with load balancer

**End of Lecture 5**
