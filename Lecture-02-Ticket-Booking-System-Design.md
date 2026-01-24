# Lecture 2: Ticket Booking System Design (like BookMyShow)

> **Lecture:** 02  
> **Topic:** System Design  
> **Application:** Ticket Booking System (BookMyShow, Paytm, TicketMaster)  
> **Scale:** 100M Daily Active Users  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-01-URL-Shortener-System-Design|Lecture 1: URL Shortener]]

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

### What is a Ticket Booking System?

A ticket booking system is an **online service** that allows users to book tickets for:
- **Movies** (theaters, cinema halls)
- **Events** (concerts, comedy shows, sports)
- **Shows** (theater performances, live events)

**Examples:** BookMyShow, Paytm Insider, TicketMaster

### Problem It Solves

1. **Centralized Platform:** Users can browse and book tickets from a single platform
2. **Real-Time Availability:** See available seats in real-time
3. **Convenience:** Book from anywhere, anytime (no need to visit physical counters)
4. **Seat Selection:** Choose specific seats based on preference
5. **Payment Integration:** Secure online payment processing
6. **Avoid Overbooking:** Ensure no two users book the same seat

---

## 2. Functional and Non-Functional Requirements
<img width="756" height="287" alt="image" src="https://github.com/user-attachments/assets/9654d845-c243-45b1-accb-14341100298c" />


### 2.1 Functional Requirements

The functional requirements are straightforward - they represent the steps a user takes to book a ticket:

1. **Search for Events/Movies**
   - User should be able to search based on:
     - **Keyword** (movie name, event name)
     - **Location** (city, theater, venue)
     - **Date** (show date/time)

2. **View Event Details**
   - Once user finds an event, they should see:
     - Event metadata (title, description, cast, performers)
     - Venue information
     - Available seats layout
     - Pricing information

3. **Book Tickets**
   - User should be able to:
     - Select specific seats
     - Reserve seats temporarily
     - Make payment
     - Confirm booking

### 2.2 Non-Functional Requirements

1. **Scale**
   - Support **100 million daily active users**
   - Handle high traffic during peak times (movie releases, popular concerts)

2. **CAP Theorem - Mixed Approach**

   **Important Discussion:**
   
   Unlike the URL Shortener where we chose Availability > Consistency, here we need **BOTH** for different parts of the system:

   **High Availability (for Search & Viewing):**
   - Search service must always be available
   - Users should always be able to browse events
   - Viewing event details should be fast and reliable
   - Small delays in showing latest seat availability is acceptable

   **High Consistency (for Booking):**
   - **Critical:** No two users should book the same seat
   - Booking must be ACID compliant
   - Payment and seat allocation must be atomic
   - Read-after-write consistency is mandatory

   **Why this split?**
   - Browsing/searching is read-heavy (80% of traffic)
   - Booking is write-heavy and requires strict consistency (20% of traffic)
   - We can optimize each part differently

3. **Low Latency**
   - Search results: < 500ms
   - Event details loading: < 300ms
   - Seat booking: < 2 seconds (including payment)

4. **Reliability**
   - No double bookings
   - Payment failures should release reserved seats
   - System should handle concurrent bookings gracefully

---

## 3. Identification of Core Entities

Based on the functional requirements, we can identify **four core entities**:

1. **User** - The person booking tickets
2. **Event/Movie** - The show/movie/concert being booked
3. **Venue** - The location where the event happens (theater, stadium, hall)
4. **Ticket/Seat** - The specific seat being booked

---

## 4. API Design

Since we have clear functional requirements, API design is straightforward (one-to-one mapping):

### 4.1 API Endpoint 1: Search Events

```
GET /v1/events/search?term={searchTerm}&location={lat,long}&date={date}
```

**Query Parameters:**
- `term` (optional): Free text search (movie name, event name, performer)
- `location` (optional): Latitude/Longitude or city name
- `date` (optional): Date of the event (YYYY-MM-DD)

**Response:**
```json
{
  "events": [
    {
      "eventId": "evt_123",
      "title": "Spider-Man: No Way Home",
      "thumbnail": "https://cdn.example.com/spiderman.jpg",
      "description": "Marvel superhero movie...",
      "venue": "PVR Cinemas, Bangalore",
      "date": "2026-01-25",
      "availableSeats": 45
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 10,
    "totalResults": 100
  }
}
```

**Important Notes:**
- Results should be **paginated** (don't return all results at once)
- Include basic metadata for displaying search results
- Discuss with interviewer: What other search filters are needed?

---

### 4.2 API Endpoint 2: View Event Details

```
GET /v1/events/{eventId}
```

**Response:**
```json
{
  "eventDetails": {
    "eventId": "evt_123",
    "title": "Spider-Man: No Way Home",
    "description": "Full description...",
    "cast": ["Tom Holland", "Zendaya"],
    "director": "Jon Watts",
    "duration": "148 minutes",
    "rating": "PG-13"
  },
  "venue": {
    "venueId": "venue_456",
    "name": "PVR Cinemas",
    "location": "Bangalore, India",
    "address": "123 MG Road"
  },
  "seats": [
    {
      "seatId": "L12",
      "row": "L",
      "number": 12,
      "status": "available",  // available, reserved, booked
      "price": 250
    },
    {
      "seatId": "L13",
      "row": "L",
      "number": 13,
      "status": "booked",
      "price": 250
    }
  ]
}
```

---

### 4.3 API Endpoint 3: Book Tickets (Two-Step Process)

**Step 1: Reserve Seats**

```
POST /v1/bookings/reserve
```

**Request Headers:**
```
Authorization: Bearer {JWT_TOKEN}  // User ID extracted from token
```

**Request Body:**
```json
{
  "eventId": "evt_123",
  "seats": ["L12", "L13"]  // List of seat IDs
}
```

**Response:**
```json
{
  "bookingId": "booking_789",
  "status": "reserved",
  "expiresAt": "2026-01-21T16:20:00Z",  // 10 minutes from now
  "totalAmount": 500
}
```

**Step 2: Confirm Booking (After Payment)**

```
POST /v1/bookings/confirm
```

**Request Body:**
```json
{
  "bookingId": "booking_789",
  "paymentDetails": {
    "paymentMethod": "card",
    "transactionId": "txn_abc123",
    "amount": 500
  }
}
```

**Response:**
```json
{
  "bookingId": "booking_789",
  "status": "confirmed",
  "tickets": ["ticket_001", "ticket_002"],
  "qrCode": "https://cdn.example.com/qr/booking_789.png"
}
```

**Important Notes:**
- **User ID** should be in the request header (JWT token), not in the body (security best practice)
- Booking is a **two-step process**: Reserve → Pay → Confirm
- Reserved seats have a **TTL (Time To Live)** of 10 minutes

---

## 5. High-Level Design (HLD)
<img width="585" height="226" alt="image" src="https://github.com/user-attachments/assets/16904b65-b47e-478a-a95f-7671978232e5" />


### 5.1 High-Level Design Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                         100M Daily Active Users                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   API Gateway   │
                    │                 │
                    │  - Auth/AuthZ   │
                    │  - Rate Limiting│
                    │  - Routing      │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │  Search  │      │  Event   │      │ Booking  │
    │ Service  │      │ Service  │      │ Service  │
    │          │      │          │      │          │
    │ (Read)   │      │ (Read)   │      │ (Write)  │
    └────┬─────┘      └────┬─────┘      └────┬─────┘
         │                 │                  │
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Database   │
                    │              │
                    │ - Events     │
                    │ - Venues     │
                    │ - Bookings   │
                    │ - Users      │
                    └──────────────┘
```

### 5.2 High-Level Design Discussion

**Q: Why do we need an API Gateway?**

Since we're targeting 100M daily active users, we need a **microservices architecture** (not monolithic). A single server cannot handle this scale.

**API Gateway responsibilities:**
1. **Authentication & Authorization:** Verify JWT tokens
2. **Rate Limiting:** Prevent abuse (e.g., max 100 requests/minute per user)
3. **Routing:** Direct requests to appropriate microservices
4. **Load Balancing:** Distribute traffic evenly

---

**Q: Why separate Search, Event, and Booking services?**

**Separation of Concerns:**
- Each service has a **single responsibility**
- Easier to scale independently
- Easier to maintain and debug

**Different Traffic Patterns:**
- **Search Service:** High read traffic (users browsing)
- **Event Service:** High read traffic (viewing details)
- **Booking Service:** Low write traffic (actual bookings)

**Scaling Strategy:**
- Scale Search & Event services more (80% of traffic)
- Scale Booking service less (20% of traffic)
- Saves cost and resources

---

**Q: What does each service do?**

**1. Search Service:**
- Receives search queries (keyword, location, date)
- Queries database for matching events
- Returns paginated results

**2. Event Service:**
- Receives event ID
- Fetches complete event metadata from database
- Returns event details, venue info, and seat availability

**3. Booking Service:**
- Handles seat reservation
- Integrates with payment gateway
- Confirms booking after successful payment
- Updates database

---

**Q: What data is stored in the database?**

**User Table:**
- User ID, name, email, phone
- Payment methods
- Booking history

**Event Table:**
- Event ID, title, description
- Performers, cast, director
- Venue ID, date, time
- Total seats, available seats

**Venue Table:**
- Venue ID, name, location
- Seat layout, capacity

**Booking Table:**
- Booking ID, user ID, event ID
- Seats booked, payment details
- Status (reserved, confirmed, cancelled)

---

## 6. Low-Level Design (LLD)

### 6.1 Key Discussion Before LLD Diagram

Before diving into the detailed design, let's address critical questions:

#### **1. Which Database to Use?**

We need **TWO types of databases** for different purposes:

**For Event Metadata (Movies, Shows, Concerts):**
- **Database Choice:** Cassandra (NoSQL)
- **Why Cassandra?**
  - **High Availability** (CAP theorem - we chose AP for search/viewing)
  - **Write-optimized** (frequent updates to seat availability)
  - **Scalable** (can handle millions of events)
  - **Distributed** (no single point of failure)

**For Booking & Payment Data:**
- **Database Choice:** PostgreSQL or MySQL (Relational)
- **Why SQL?**
  - **High Consistency** (ACID properties)
  - **No double bookings** (transactions ensure atomicity)
  - **Data integrity** (foreign keys, constraints)
  - **Payment data requires strict consistency**

---

#### **2. How to Optimize Search?**

**Problem:**
- Cassandra has millions of event records
- Cassandra is **write-optimized, not read-optimized**
- Searching through millions of records is slow
- Users expect search results in < 500ms

**Solution: Elasticsearch**

**Why Elasticsearch?**
- **Highly optimized for search** (inverted index)
- **Full-text search** (search by keywords, partial matches)
- **Fast** (sub-second response time)
- **Supports complex queries** (filters, aggregations)

**Architecture:**
```
Cassandra (Source of Truth)
    │
    │ CDC Pipeline (Change Data Capture)
    │
    ▼
Elasticsearch (Search Index)
    │
    │ Search Queries
    │
    ▼
Search Service
```

---

#### **3. What is CDC (Change Data Capture)?**

**Problem:**
- Event data is stored in Cassandra
- Search queries go to Elasticsearch
- How to keep Elasticsearch in sync with Cassandra?

**Solution: CDC Pipeline**

**How CDC Works:**

```
Cassandra DB
    │
    │ (Detects changes)
    ▼
Debezium (CDC Tool)
    │
    │ (Publishes change events)
    ▼
Kafka (Streaming Service)
    │
    │ (Consumes events)
    ▼
Ingestion Job
    │
    │ (Updates index)
    ▼
Elasticsearch
```

**Real-Time Example:**
1. User books 2 seats for Spider-Man movie
2. Cassandra updates: `availableSeats: 20 → 18`
3. Debezium detects change in Cassandra
4. Publishes event to Kafka: `{"eventId": "evt_123", "availableSeats": 18}`
5. Ingestion job consumes from Kafka
6. Updates Elasticsearch index
7. Next user searching sees **18 seats available** (real-time!)

**Why This Approach?**
- **Real-time sync** (near-instant updates)
- **Decoupled** (Cassandra and Elasticsearch don't directly talk)
- **Reliable** (Kafka ensures no data loss)
- **Scalable** (can handle high throughput)

---

#### **4. How to Prevent Double Bookings?**

This is the **most critical** part of the system!

**Scenario:**
- Spider-Man movie has seats L12 and L13 available
- User 1 tries to book L12, L13 at 4:00:00 PM
- User 2 tries to book L12, L13 at 4:00:01 PM
- **Only one should succeed!**

**Two Approaches:**

---

**Approach 1: Cron Job (Not Recommended)**

**Flow:**
1. User 1 requests seats L12, L13
2. Booking service checks Cassandra: `status = available`
3. Update Cassandra: `status = reserved` (for 10 minutes)
4. Redirect User 1 to payment gateway
5. User 2 requests same seats
6. Booking service checks Cassandra: `status = reserved`
7. Return error: "Seats already reserved"

**Handling Expired Reservations:**
- Cron job runs **every minute**
- Scans entire database for `status = reserved`
- Checks if reservation time > 10 minutes
- If yes, update `status = available`

**Problems:**
❌ **Not real-time** (runs every minute, not instant)  
❌ **Database overhead** (scans entire database every minute)  
❌ **Not scalable** (millions of records to scan)  
❌ **Inefficient** (most seats are not reserved)  

---

**Approach 2: Redis with TTL (Recommended)**

**Flow:**
1. User 1 requests seats L12, L13
2. Booking service checks Redis: `GET L12, L13`
3. If not in Redis → seats are available
4. Add to Redis: `SET L12 "user_1" EX 600` (TTL = 10 minutes)
5. Add to Redis: `SET L13 "user_1" EX 600`
6. Redirect User 1 to payment gateway

7. User 2 requests same seats (1 second later)
8. Booking service checks Redis: `GET L12, L13`
9. Found in Redis → seats are reserved
10. Return error: "Seats already reserved"

**If Payment Succeeds:**
- Delete from Redis: `DEL L12, L13`
- Update PostgreSQL: `status = confirmed`
- Update Cassandra: `availableSeats -= 2`

**If Payment Fails or Times Out:**
- Redis **automatically deletes** after 10 minutes (TTL expired)
- No cron job needed!
- Next user can book immediately

**Advantages:**
✅ **Real-time** (instant check, no delays)  
✅ **Automatic cleanup** (TTL handles expiration)  
✅ **Fast** (Redis is in-memory, < 2ms latency)  
✅ **Scalable** (only active reservations in Redis)  
✅ **No database scans** (efficient)  

---

### 6.2 Low-Level Design Diagram (Final Architecture)
<img width="634" height="269" alt="image" src="https://github.com/user-attachments/assets/42a77e2e-fef3-4062-81d8-8ca4ff868b5c" />


```
┌────────────────────────────────────────────────────────────────────┐
│                      100M Daily Active Users                        │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │   API Gateway   │
                      │                 │
                      │  - Auth/AuthZ   │
                      │  - Rate Limiting│
                      │  - Routing      │
                      └────────┬────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
      ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
      │   Search     │  │    Event     │  │   Booking    │
      │   Service    │  │   Service    │  │   Service    │
      │              │  │              │  │              │
      │ (Scale: 10x) │  │ (Scale: 10x) │  │ (Scale: 3x)  │
      └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
             │                 │                  │
             │                 │                  │
             ▼                 ▼                  │
      ┌──────────────┐  ┌──────────────┐         │
      │Elasticsearch │  │  Cassandra   │         │
      │              │  │   Database   │         │
      │ - Fast Search│  │              │         │
      │ - Inverted   │  │ - Events     │         │
      │   Index      │  │ - Venues     │         │
      │              │  │ - Seats      │         │
      └──────────────┘  └──────┬───────┘         │
             ▲                 │                  │
             │                 │                  │
             │          ┌──────▼───────┐          │
             │          │   Debezium   │          │
             │          │  (CDC Tool)  │          │
             │          └──────┬───────┘          │
             │                 │                  │
             │          ┌──────▼───────┐          │
             │          │    Kafka     │          │
             │          │  (Streaming) │          │
             │          └──────┬───────┘          │
             │                 │                  │
             │          ┌──────▼───────┐          │
             │          │  Ingestion   │          │
             │          │     Job      │          │
             └──────────┴──────────────┘          │
                                                  │
                                                  ▼
                                          ┌──────────────┐
                                          │    Redis     │
                                          │    Cache     │
                                          │              │
                                          │ - Seat Lock  │
                                          │ - TTL: 10min │
                                          └──────┬───────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │  PostgreSQL  │
                                          │   Database   │
                                          │              │
                                          │ - Bookings   │
                                          │ - Payments   │
                                          │ - Users      │
                                          └──────┬───────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │   Payment    │
                                          │   Gateway    │
                                          │              │
                                          │ - Razorpay   │
                                          │ - Stripe     │
                                          └──────────────┘
```

---

### 6.3 Complete Data Flow

#### **Flow 1: Searching for Events**

```
User enters: "Spider-Man Bangalore 2026-01-25"
    │
    ▼
API Gateway (Auth check)
    │
    ▼
Search Service
    │
    ▼
Elasticsearch (Query: title=Spider-Man, location=Bangalore, date=2026-01-25)
    │
    ▼
Returns: [evt_123, evt_456, evt_789]
    │
    ▼
Search Service (Fetch metadata for display)
    │
    ▼
User sees search results (paginated)
```

**Latency:** ~300-500ms

---

#### **Flow 2: Viewing Event Details**

```
User clicks on "Spider-Man: No Way Home"
    │
    ▼
API Gateway
    │
    ▼
Event Service (GET /events/evt_123)
    │
    ▼
Cassandra Database (Fetch event metadata + seat availability)
    │
    ▼
Returns: {title, description, cast, venue, seats: [{L12: available}, {L13: booked}]}
    │
    ▼
User sees event details page with seat layout
```

**Latency:** ~200-300ms

---

#### **Flow 3: Booking Tickets (Happy Path)**

**Step 1: Reserve Seats**

```
User selects seats L12, L13 and clicks "Book"
    │
    ▼
API Gateway (Extract user_id from JWT token)
    │
    ▼
Booking Service (POST /bookings/reserve)
    │
    ▼
Check Redis: GET L12, GET L13
    │
    ├─ If found in Redis → Return error "Seats already reserved"
    │
    └─ If NOT found in Redis:
        │
        ▼
    Add to Redis:
        SET L12 "user_1" EX 600  (TTL = 10 minutes)
        SET L13 "user_1" EX 600
        │
        ▼
    Create booking record in PostgreSQL:
        booking_id: booking_789
        status: "reserved"
        expires_at: current_time + 10 minutes
        │
        ▼
    Return to user:
        {
          "bookingId": "booking_789",
          "status": "reserved",
          "expiresAt": "2026-01-21T16:20:00Z",
          "totalAmount": 500
        }
        │
        ▼
    User redirected to Payment Gateway
```

**Step 2: Payment & Confirmation**

```
User completes payment on Payment Gateway
    │
    ▼
Payment Gateway sends callback to Booking Service
    │
    ▼
Booking Service (POST /bookings/confirm)
    │
    ▼
Verify payment with Payment Gateway
    │
    ├─ If payment failed:
    │   │
    │   └─ Keep seats in Redis (will auto-expire after 10 min)
    │       Return error to user
    │
    └─ If payment succeeded:
        │
        ▼
    Delete from Redis: DEL L12, DEL L13
        │
        ▼
    Update PostgreSQL:
        UPDATE bookings SET status='confirmed' WHERE booking_id='booking_789'
        │
        ▼
    Update Cassandra:
        UPDATE events SET availableSeats = availableSeats - 2 WHERE event_id='evt_123'
        │
        ▼
    CDC Pipeline triggers:
        Debezium detects change → Kafka → Ingestion Job → Elasticsearch
        │
        ▼
    Return to user:
        {
          "bookingId": "booking_789",
          "status": "confirmed",
          "tickets": ["ticket_001", "ticket_002"]
        }
        │
        ▼
    User receives confirmation email + tickets
```

**Latency:** ~1-2 seconds (including payment processing)

---

#### **Flow 4: Booking Tickets (Payment Timeout)**

```
User reserves seats L12, L13
    │
    ▼
Seats added to Redis with TTL = 10 minutes
    │
    ▼
User redirected to Payment Gateway
    │
    ▼
User abandons payment (closes browser)
    │
    ▼
10 minutes pass...
    │
    ▼
Redis automatically deletes L12, L13 (TTL expired)
    │
    ▼
PostgreSQL booking record still shows "reserved"
    │
    ▼
Next user tries to book L12, L13
    │
    ▼
Check Redis: NOT FOUND (expired)
    │
    ▼
Seats are available again!
    │
    ▼
New user can book successfully
```

**No cron job needed! Redis TTL handles everything automatically.**

---

### 6.4 Database Schema Design

#### **Cassandra Database (Event Metadata)**

**Events Table:**
```
event_id (Primary Key)
title
description
performers (List)
venue_id
date
time
total_seats
available_seats
created_at
updated_at
```

**Venues Table:**
```
venue_id (Primary Key)
name
location
address
city
capacity
seat_layout (JSON)
```

**Seats Table:**
```
seat_id (Primary Key)
event_id
row
number
price
status (available, reserved, booked)
```

---

#### **PostgreSQL Database (Bookings & Payments)**

**Users Table:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20),
    created_at TIMESTAMP
);
```

**Bookings Table:**
```sql
CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    event_id UUID,
    seats TEXT[],  -- Array of seat IDs
    total_amount DECIMAL(10, 2),
    status VARCHAR(20),  -- reserved, confirmed, cancelled
    reserved_at TIMESTAMP,
    expires_at TIMESTAMP,
    confirmed_at TIMESTAMP,
    payment_id UUID
);

CREATE INDEX idx_bookings_user_id ON bookings(user_id);
CREATE INDEX idx_bookings_status ON bookings(status);
```

**Payments Table:**
```sql
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    booking_id UUID REFERENCES bookings(booking_id),
    amount DECIMAL(10, 2),
    payment_method VARCHAR(50),
    transaction_id VARCHAR(255),
    status VARCHAR(20),  -- pending, success, failed
    created_at TIMESTAMP
);
```

---

#### **Redis Cache (Seat Locks)**

**Key-Value Structure:**
```
Key: seat:{event_id}:{seat_id}
Value: {user_id}
TTL: 600 seconds (10 minutes)

Example:
Key: seat:evt_123:L12
Value: user_1
TTL: 600
```

**Commands:**
```redis
# Reserve seat
SET seat:evt_123:L12 user_1 EX 600

# Check if seat is reserved
GET seat:evt_123:L12

# Release seat (after payment)
DEL seat:evt_123:L12
```

---

### 6.5 Handling Concurrent Bookings

**Race Condition Scenario:**

```
Time: 4:00:00.000 PM
User 1: Clicks "Book" for L12, L13
User 2: Clicks "Book" for L12, L13 (0.001 seconds later)
```

**How Redis Prevents Double Booking:**

```
User 1 Request:
    │
    ▼
Booking Service: GET seat:evt_123:L12
    │
    ▼
Redis: NULL (not found)
    │
    ▼
Booking Service: SET seat:evt_123:L12 user_1 EX 600
    │
    ▼
Redis: OK (seat locked for User 1)

User 2 Request (0.001 seconds later):
    │
    ▼
Booking Service: GET seat:evt_123:L12
    │
    ▼
Redis: "user_1" (already reserved!)
    │
    ▼
Booking Service: Return error "Seat already reserved"
```

**Why This Works:**
- Redis is **single-threaded** (atomic operations)
- Even if requests arrive simultaneously, Redis processes them **sequentially**
- First request wins, second request fails
- **No race condition possible!**

---

## 7. Interview-Style Design Discussion (Q&A Format)

### **Q1: Why do we need both Cassandra and PostgreSQL? Can't we use just one database?**

**A:** No, we need both because they serve different purposes:

**Cassandra (for Event Metadata):**
- **High Availability** (AP in CAP theorem)
- Stores millions of events, movies, venues
- Frequently updated (seat availability changes)
- Read-heavy for search/browsing
- Eventual consistency is acceptable

**PostgreSQL (for Bookings & Payments):**
- **High Consistency** (CA in CAP theorem)
- ACID transactions (critical for payments)
- No double bookings (strict consistency)
- Financial data requires audit trail
- Strong consistency is mandatory

**Trade-off:**
- Using two databases adds complexity
- But it's necessary to meet both availability and consistency requirements

---

### **Q2: Why Elasticsearch? Can't we just search directly in Cassandra?**

**A:** Cassandra is **write-optimized, not read-optimized**.

**Problems with Cassandra for Search:**
- Slow full-table scans (millions of records)
- Limited query capabilities (no full-text search)
- No relevance ranking
- Cannot search by partial keywords

**Elasticsearch Advantages:**
- **Inverted index** (optimized for search)
- **Full-text search** (search "Spider" finds "Spider-Man")
- **Fast** (sub-second response for millions of records)
- **Relevance scoring** (best matches first)
- **Filters & aggregations** (by location, date, price)

**Cost:**
- Data duplication (same data in Cassandra + Elasticsearch)
- CDC pipeline complexity
- But the performance gain is worth it!

---

### **Q3: What is CDC and why do we need it?**

**A:** **CDC = Change Data Capture**

**Problem:**
- Event data is in Cassandra (source of truth)
- Search queries go to Elasticsearch
- When seat availability changes in Cassandra, Elasticsearch must be updated

**Without CDC:**
- Manual sync (error-prone)
- Batch jobs (not real-time)
- Stale data in Elasticsearch

**With CDC:**
- **Automatic sync** (no manual intervention)
- **Real-time** (changes reflected in seconds)
- **Reliable** (Kafka ensures no data loss)

**How it works:**
1. Debezium monitors Cassandra transaction logs
2. Detects INSERT/UPDATE/DELETE operations
3. Publishes change events to Kafka
4. Ingestion job consumes from Kafka
5. Updates Elasticsearch index

---

### **Q4: Why Redis for seat locking? Why not use database locks?**

**A:** Database locks have serious problems at scale:

**Problems with Database Locks:**
- **Deadlocks** (two users locking each other's seats)
- **Performance** (locks block other transactions)
- **Scalability** (locks don't work well in distributed systems)
- **Complexity** (need to handle lock timeouts, retries)

**Redis Advantages:**
- **Fast** (in-memory, < 2ms latency)
- **Atomic operations** (single-threaded, no race conditions)
- **TTL support** (automatic expiration, no cron jobs)
- **Simple** (just SET/GET/DEL operations)
- **Scalable** (can use Redis Cluster)

**Trade-off:**
- Redis is another component to manage
- But the simplicity and performance are worth it

---

### **Q5: What happens if Redis goes down?**

**A:** This is a critical failure scenario!

**Impact:**
- Cannot reserve new seats
- Existing reservations are lost
- Users might experience double bookings

**Solutions:**

**1. Redis Cluster (High Availability):**
- Run 3-5 Redis instances
- Master-slave replication
- Automatic failover
- If master dies, slave becomes master

**2. Redis Persistence:**
- Enable RDB snapshots (periodic backups)
- Enable AOF (Append-Only File) for durability
- On restart, Redis loads data from disk

**3. Fallback to Database:**
- If Redis is down, fall back to PostgreSQL locks
- Slower, but system remains functional

**4. Circuit Breaker Pattern:**
- Detect Redis failures quickly
- Return user-friendly error: "Booking temporarily unavailable"
- Prevent cascading failures

---

### **Q6: How do you handle peak traffic (e.g., Coldplay concert tickets)?**

**A:** Peak traffic requires special handling!

**Problem:**
- Coldplay concert announced
- 1 million users try to book at the same time
- Booking service gets overwhelmed
- System crashes

**Solution: Introduce a Waiting Queue (Kafka)**

```
User Request
    │
    ▼
API Gateway
    │
    ▼
Kafka Queue (Waiting Queue)
    │
    │ (Process requests one by one)
    │
    ▼
Booking Service (Controlled rate)
    │
    ▼
Redis (Seat locking)
    │
    ▼
Database
```

**How it works:**
1. All booking requests go to Kafka queue
2. Booking service consumes at a **controlled rate** (e.g., 100 req/sec)
3. Users see their position in queue: "You are #4,532 in line"
4. When their turn comes, they can book
5. Prevents system overload

**User Experience:**
- "You are in a waiting queue"
- "Estimated wait time: 5 minutes"
- "Your position: #4,532"
- Better than system crash!

---

### **Q7: How do you prevent scalpers/bots from booking all tickets?**

**A:** This is a real-world problem!

**Anti-Bot Measures:**

**1. CAPTCHA:**
- Show CAPTCHA before booking
- Prevents automated bots

**2. Rate Limiting:**
- Max 5 booking attempts per user per minute
- Prevents bulk bookings

**3. Device Fingerprinting:**
- Track device ID, IP address
- Block suspicious patterns

**4. Waiting Room:**
- Randomize queue position (not first-come-first-served)
- Prevents bots from getting advantage

**5. Phone Verification:**
- Require phone OTP before booking
- One booking per phone number

**6. Behavioral Analysis:**
- Track mouse movements, typing speed
- Bots behave differently than humans

---

### **Q8: How do you handle refunds and cancellations?**

**A:** 

**Cancellation Flow:**

```
User clicks "Cancel Booking"
    │
    ▼
Booking Service
    │
    ▼
Update PostgreSQL: status = 'cancelled'
    │
    ▼
Update Cassandra: availableSeats += 2
    │
    ▼
CDC Pipeline: Update Elasticsearch
    │
    ▼
Initiate Refund with Payment Gateway
    │
    ▼
Send confirmation email to user
```

**Refund Policy:**
- Full refund: > 24 hours before event
- 50% refund: 12-24 hours before event
- No refund: < 12 hours before event

**Database Updates:**
```sql
UPDATE bookings 
SET status = 'cancelled', 
    cancelled_at = NOW() 
WHERE booking_id = 'booking_789';

UPDATE payments 
SET status = 'refunded', 
    refund_amount = 500 
WHERE booking_id = 'booking_789';
```

---

### **Q9: How do you ensure payment security?**

**A:** 

**Security Measures:**

**1. PCI-DSS Compliance:**
- Never store credit card numbers
- Use tokenization (Payment Gateway handles card data)

**2. HTTPS:**
- All communication encrypted (TLS 1.3)

**3. JWT Tokens:**
- User authentication via JWT
- Short expiry (15 minutes)
- Refresh tokens for long sessions

**4. Payment Gateway Integration:**
- Use trusted providers (Razorpay, Stripe)
- They handle card data, we only get transaction ID

**5. Idempotency:**
- Prevent duplicate charges
- Use idempotency keys (booking_id)

**6. Audit Logs:**
- Log all payment transactions
- Track who, what, when

---

### **Q10: How do you scale the Search Service?**

**A:** 

**Scaling Strategies:**

**1. Horizontal Scaling:**
- Run 10+ instances of Search Service
- Load balancer distributes traffic

**2. Elasticsearch Cluster:**
- Run 5-10 Elasticsearch nodes
- Shard data across nodes
- Replicate for fault tolerance

**3. Caching:**
- Cache popular searches in Redis
- "Spider-Man Bangalore" cached for 5 minutes
- Reduces Elasticsearch load

**4. CDN:**
- Cache search results at edge locations
- Users get results from nearest CDN

**5. Database Indexing:**
- Index on: title, location, date
- Faster query execution

---

### **Q11: How do you monitor the system?**

**A:** 

**Monitoring Stack:**

**1. Metrics (Prometheus + Grafana):**
- Request latency (p50, p95, p99)
- Error rate (4xx, 5xx)
- Database query time
- Redis hit/miss ratio
- Kafka lag

**2. Logging (ELK Stack):**
- Centralized logs (Elasticsearch, Logstash, Kibana)
- Search logs by user_id, booking_id
- Debug production issues

**3. Alerting (PagerDuty):**
- Alert if latency > 2 seconds
- Alert if error rate > 1%
- Alert if Redis is down
- Alert if Kafka lag > 1000 messages

**4. Distributed Tracing (Jaeger):**
- Trace request across microservices
- Identify bottlenecks

**5. Health Checks:**
- API Gateway pings services every 30 seconds
- Remove unhealthy instances from load balancer

---

### **Q12: How do you handle database failures?**

**A:** 

**Cassandra Failure:**
- **Replication Factor = 3** (data copied to 3 nodes)
- If 1 node dies, other 2 serve requests
- Auto-recovery when node comes back

**PostgreSQL Failure:**
- **Master-Slave Replication**
- If master dies, promote slave to master
- Use connection pooling (HikariCP)

**Redis Failure:**
- **Redis Cluster** (3-5 nodes)
- Master-slave replication
- Automatic failover

**Elasticsearch Failure:**
- **Cluster with 5+ nodes**
- Shard replication
- If node dies, replicas serve requests

---

### **Q13: What are the edge cases to handle?**

**A:** 

**1. User books but payment fails:**
- Redis TTL expires after 10 minutes
- Seats become available again
- User gets error message

**2. User books, pays, but confirmation fails:**
- Payment succeeded but database update failed
- **Solution:** Retry mechanism with exponential backoff
- If still fails, manual intervention (support team)

**3. Two users book same seat simultaneously:**
- Redis atomic operations prevent this
- First request wins, second fails

**4. Event gets cancelled:**
- Admin marks event as cancelled
- Trigger refunds for all bookings
- Send email notifications

**5. Venue changes seat layout:**
- Admin updates seat layout in Cassandra
- CDC updates Elasticsearch
- Existing bookings remain valid

**6. User tries to book expired event:**
- Check `event_date < current_date`
- Return error: "Event has already occurred"

---

### **Q14: How do you estimate system capacity?**

**A:** 

**Given:**
- 100M daily active users
- Assume 10% actually book tickets (10M bookings/day)
- Assume 80% browse/search (80M searches/day)

**Calculations:**

**Bookings per second:**
- 10M bookings/day = 10,000,000 / 86,400 = **116 bookings/sec**
- Peak traffic (10x): **1,160 bookings/sec**

**Searches per second:**
- 80M searches/day = 80,000,000 / 86,400 = **926 searches/sec**
- Peak traffic (10x): **9,260 searches/sec**

**Storage:**
- Each event: ~5 KB (metadata)
- 1M events: 5 GB
- Each booking: ~1 KB
- 10M bookings/day × 365 days = 3.65 GB/year
- With replication (3x): ~15 GB/year

**Servers Needed:**
- Each server handles ~1,000 req/sec
- Search Service: 10 instances (9,260 req/sec)
- Booking Service: 2 instances (1,160 req/sec)
- Event Service: 5 instances

---

### **Q15: How would you improve this system further?**

**A:** 

**1. Recommendation Engine:**
- ML model to recommend events based on user history
- "Users who booked Spider-Man also booked Avengers"

**2. Dynamic Pricing:**
- Surge pricing during high demand
- Discounts for early bookings

**3. Social Features:**
- Share bookings on social media
- Group bookings (book with friends)

**4. Notifications:**
- Push notifications for event reminders
- SMS/Email for booking confirmations

**5. Analytics Dashboard:**
- Admin dashboard for event organizers
- Track ticket sales, revenue, popular events

**6. Multi-Language Support:**
- Support 10+ languages
- Localized content

**7. Accessibility:**
- Screen reader support
- Wheelchair-accessible seat selection

**8. Offline Mode:**
- Progressive Web App (PWA)
- Cache event details for offline viewing

---

## 8. Additional Insights

### **Key Takeaways**

✅ **Use the right database for the right job**
- Cassandra for high availability (event metadata)
- PostgreSQL for high consistency (bookings, payments)

✅ **Elasticsearch is essential for fast search**
- Don't try to search in Cassandra/PostgreSQL
- CDC pipeline keeps Elasticsearch in sync

✅ **Redis TTL is perfect for temporary locks**
- No cron jobs needed
- Automatic cleanup
- Fast and scalable

✅ **Separate read and write services**
- Search/Event services (read-heavy, scale 10x)
- Booking service (write-heavy, scale 3x)

✅ **Handle peak traffic with queues**
- Kafka waiting queue for popular events
- Prevents system overload

✅ **Security is critical**
- JWT authentication
- PCI-DSS compliance
- Never store card data

---

### **Common Mistakes to Avoid**

❌ Using only one database for everything  
❌ Searching directly in Cassandra (slow!)  
❌ Using cron jobs for seat expiration (inefficient)  
❌ Not handling concurrent bookings (double bookings!)  
❌ Ignoring peak traffic scenarios  
❌ Not separating read and write services  
❌ Storing credit card data (security risk!)  
❌ Not monitoring the system  

---

### **System Design Framework (5 Steps)**

This lecture demonstrates the **5-step framework** for system design:

1. ✅ **Gather Requirements** (Functional + Non-Functional)
2. ✅ **Identify Core Entities** (User, Event, Venue, Ticket)
3. ✅ **Design APIs** (Search, View, Reserve, Confirm)
4. ✅ **High-Level Design** (Microservices, API Gateway, Databases)
5. ✅ **Low-Level Design** (CDC, Redis locks, Elasticsearch, detailed flows)

**Always follow this framework in interviews!**

---

## Summary

This Ticket Booking System design demonstrates:

- **Microservices architecture** with separation of concerns
- **Two databases** (Cassandra for availability, PostgreSQL for consistency)
- **Elasticsearch** for fast, scalable search
- **CDC pipeline** for real-time data sync
- **Redis with TTL** for seat locking (no cron jobs!)
- **Kafka queue** for handling peak traffic
- **Payment gateway integration** with security best practices

**Final Architecture Components:**
- API Gateway (Auth, Rate Limiting, Routing)
- Search Service (10 instances) → Elasticsearch
- Event Service (5 instances) → Cassandra
- Booking Service (3 instances) → Redis → PostgreSQL → Payment Gateway
- CDC Pipeline (Debezium → Kafka → Ingestion Job)

**Performance:**
- Search: ~300-500ms
- Event Details: ~200-300ms
- Booking: ~1-2 seconds (including payment)

**Scale:**
- 100M daily active users
- 10M bookings/day
- 80M searches/day
- Peak traffic: 10x normal load

All requirements met! ✅

---

**End of Lecture 2**

**Next:** Lecture 3: [Next System Design Topic]
