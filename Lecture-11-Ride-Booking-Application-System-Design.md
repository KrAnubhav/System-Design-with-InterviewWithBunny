# Lecture 11: Ride Booking Application System Design (Uber/Ola)

> **Lecture:** 11  
> **Topic:** System Design  
> **Application:** Ride Booking (Uber, Ola, Lyft, Grab)  
> **Scale:** Millions of Users and Drivers  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-10-Collaborative-Text-Editor-System-Design|Lecture 10: Collaborative Text Editor]]  
> **Prerequisite:** [[Lecture-07-Proximity-Search-Algorithms|Lecture 7: Proximity Search Algorithms]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Architecture Diagram](#7-complete-architecture-diagram)
8. [Complete Ride Flow](#8-complete-ride-flow)
9. [Interview Q&A](#9-interview-qa)
10. [Key Takeaways](#10-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Ride Booking Application?

A **ride booking application** connects riders with nearby drivers, calculates fare estimates, manages ride assignments, and tracks rides in real-time.

**Examples:** Uber, Ola, Lyft, Grab, DiDi

### Problem It Solves

1. **Fair Estimation:** Calculate ride cost before booking
2. **Driver Matching:** Find nearby available drivers
3. **Real-Time Tracking:** Track driver and rider locations
4. **Payment Processing:** Handle ride payments
5. **Rating System:** Allow mutual ratings

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Fare Estimation**
   - Calculate estimated fare based on pickup/drop location
   - Show different vehicle types (Bike, Car, SUV)

2. **Ride Booking**
   - Request ride with estimated fare
   - Match with nearby available driver

3. **Driver Acceptance**
   - Driver can accept or deny ride request
   - Notify rider of driver assignment

4. **Real-Time Tracking**
   - Track driver and rider locations
   - Update every 5-10 seconds

5. **Rating System**
   - Rider rates driver
   - Driver rates rider

6. **Payment**
   - Process payment after ride completion
   - Handle refunds if needed

### 2.2 Non-Functional Requirements

1. **Scale**
   - **Millions of users and drivers**
   - **High concurrent bookings**

2. **CAP Theorem**
   - **User Side:** Highly Available (AP) - Booking should work
   - **Driver Side:** Highly Consistent (CP) - One driver, one ride
   
   **Why?**
   - ✅ User: If booking fails, they retry
   - ✅ Driver: Cannot be assigned to 2 rides simultaneously

3. **Low Latency**
   - **Driver assignment within 1 minute**
   - **Location updates every 5-10 seconds**

---

## 3. Core Entities

1. **User/Rider** - Person booking the ride
2. **Driver** - Person providing the ride
3. **Fare** - Estimated cost of ride
4. **Ride** - The actual trip
5. **Location** - Pickup and drop coordinates

---

## 4. API Design

### 4.1 User APIs

#### **Get Fare Estimation**

```
GET /v1/rides/estimate?pickupLat={lat}&pickupLng={lng}&dropLat={lat}&dropLng={lng}
```

**Response:**
```json
{
  "requestId": "req_123",
  "estimates": [
    {
      "vehicleType": "BIKE",
      "fare": 50,
      "eta": "5 mins",
      "currency": "INR"
    },
    {
      "vehicleType": "CAR",
      "fare": 150,
      "eta": "8 mins",
      "currency": "INR"
    },
    {
      "vehicleType": "SUV",
      "fare": 250,
      "eta": "10 mins",
      "currency": "INR"
    }
  ]
}
```

---

#### **Book Ride**

```
POST /v1/rides/book
```

**Request Body:**
```json
{
  "requestId": "req_123",
  "vehicleType": "CAR"
}
```

**Response:**
```json
{
  "rideId": "ride_456",
  "status": "DRIVER_ASSIGNED",
  "driver": {
    "driverId": "driver_789",
    "name": "Rahul",
    "phone": "+919876543210",
    "vehicleNumber": "KA-01-AB-1234",
    "rating": 4.8,
    "eta": "5 mins"
  }
}
```

---

#### **Get Ride History**

```
GET /v1/rides/history?userId={userId}
```

**Response:**
```json
{
  "rides": [
    {
      "rideId": "ride_456",
      "driverName": "Rahul",
      "fare": 150,
      "status": "COMPLETED",
      "date": "2026-01-21"
    }
  ]
}
```

---

#### **Cancel Ride**

```
DELETE /v1/rides/{rideId}
```

---

#### **Rate Ride**

```
POST /v1/rides/{rideId}/rating
```

**Request Body:**
```json
{
  "rating": 5,
  "feedback": "Great ride!"
}
```

---

### 4.2 Driver APIs

#### **Update Location (WebSocket!)**

```
WS /v1/drivers/location
```

**Message:**
```json
{
  "driverId": "driver_789",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "timestamp": "2026-01-21T12:00:00Z"
}
```

⚠️ **Important:** WebSocket for real-time updates (every 5-10 seconds)

---

#### **Accept/Deny Ride**

```
POST /v1/rides/{requestId}/response
```

**Request Body:**
```json
{
  "driverId": "driver_789",
  "action": "ACCEPT"  // or "DENY"
}
```

---

#### **Start Ride**

```
POST /v1/rides/{rideId}/start
```

---

#### **End Ride**

```
POST /v1/rides/{rideId}/end
```

**Response:**
```json
{
  "rideId": "ride_456",
  "fareEstimate": 150,
  "actualFare": 165,
  "distance": "12.5 km",
  "duration": "35 mins"
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│           USERS (Riders)          DRIVERS               │
└────────────────────────┬─────────────┬──────────────────┘
                         │             │
                         ▼             ▼
                ┌─────────────────────────────┐
                │  API Gateway / Load Balancer │
                └────────────────┬─────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │  Ride    │            │  Driver  │            │  Rating  │
  │ Service  │            │ Matching │            │ Service  │
  │          │            │ Service  │            │          │
  └────┬─────┘            └────┬─────┘            └────┬─────┘
       │                       │                       │
       ▼                       ▼                       ▼
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │  Google  │            │  Driver  │            │  Rating  │
  │  Maps    │            │    DB    │            │    DB    │
  └──────────┘            └──────────┘            └──────────┘


  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │ Location │            │ Payment  │            │ Notifi-  │
  │ Update   │            │ Service  │            │ cation   │
  │ Service  │            │          │            │ Service  │
  └────┬─────┘            └────┬─────┘            └────┬─────┘
       │                       │                       │
       ▼                       ▼                       ▼
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │  Redis   │            │ Payment  │            │ FCM/APNS │
  │  Cache   │            │ Gateway  │            │          │
  └──────────┘            └──────────┘            └──────────┘
```

### 5.2 Service Responsibilities

**Ride Service:**
- Calculate fare estimate
- Create ride requests

**Driver Matching Service:**
- Find nearby drivers (proximity search)
- Assign driver to ride
- Handle driver locking (Zookeeper)

**Location Update Service:**
- Receive real-time driver locations
- Store in Redis cache

**Rating Service:**
- Process ratings from users and drivers

**Payment Service:**
- Process payments after ride completion

**Notification Service:**
- Send push notifications (FCM/APNS)

---

## 6. Low-Level Design (LLD)

### 6.1 Ride Service (Fare Calculation)

#### **Flow:**

```
User requests fare estimate
    ↓
Ride Service receives request
    ↓
Call Google Maps API:
    - Get distance (12.5 km)
    - Get ETA (35 mins)
    ↓
Calculate base fare:
    fare = distance × rate_per_km + time × rate_per_min
    ↓
Apply surge pricing (if applicable)
    ↓
Return estimates for all vehicle types
```

---

#### **Rate DB (PostgreSQL):**

```sql
CREATE TABLE rates (
    rate_id UUID PRIMARY KEY,
    vehicle_type VARCHAR(20),  -- BIKE, CAR, SUV
    rate_per_km DECIMAL(10,2),
    rate_per_min DECIMAL(10,2),
    base_fare DECIMAL(10,2),
    waiting_charge_per_min DECIMAL(10,2),
    city VARCHAR(100),
    created_at TIMESTAMP
);
```

**Sample Data:**
```sql
INSERT INTO rates VALUES
    ('1', 'BIKE', 8.0, 1.0, 20.0, 1.0, 'Bangalore'),
    ('2', 'CAR', 12.0, 2.0, 50.0, 2.0, 'Bangalore'),
    ('3', 'SUV', 18.0, 3.0, 80.0, 3.0, 'Bangalore');
```

---

#### **Fare Calculation Formula:**

```python
def calculate_fare(distance_km, duration_mins, vehicle_type, surge_multiplier=1.0):
    rates = get_rates(vehicle_type)
    
    base_fare = rates.base_fare
    distance_fare = distance_km * rates.rate_per_km
    time_fare = duration_mins * rates.rate_per_min
    
    subtotal = base_fare + distance_fare + time_fare
    
    # Apply surge pricing
    total_fare = subtotal * surge_multiplier
    
    return round(total_fare, 2)
```

---

### 6.2 Surge Pricing Service

**Problem:** During high demand, increase prices

**Solution:** Analyze ride requests in real-time

---

#### **Flow:**

```
User requests ride
    ↓
Ride Service stores request in Ride Request DB
    ↓
Surge Calculator Service reads requests
    ↓
Calculate surge multiplier:
    - Requests in last 10 mins: 500
    - Available drivers: 100
    - Ratio: 5:1 → Surge: 1.5x
    ↓
Return surge multiplier to Ride Service
    ↓
Apply surge to base fare
```

---

#### **Surge Calculation:**

```python
def calculate_surge(location, time_window_mins=10):
    # Get recent ride requests in area
    requests = db.query("""
        SELECT COUNT(*) FROM ride_requests
        WHERE location = ?
        AND created_at > NOW() - INTERVAL ? MINUTE
    """, location, time_window_mins)
    
    # Get available drivers in area
    drivers = redis.scard(f"available_drivers:{location}")
    
    # Calculate ratio
    ratio = requests / max(drivers, 1)
    
    # Surge mapping
    if ratio > 5:
        return 2.0  # 2x surge
    elif ratio > 3:
        return 1.5  # 1.5x surge
    elif ratio > 2:
        return 1.3  # 1.3x surge
    else:
        return 1.0  # No surge
```

---

#### **Ride Request DB (Analytics):**

```sql
CREATE TABLE ride_requests (
    request_id UUID PRIMARY KEY,
    user_id UUID,
    pickup_lat DECIMAL(10,8),
    pickup_lng DECIMAL(11,8),
    drop_lat DECIMAL(10,8),
    drop_lng DECIMAL(11,8),
    vehicle_type VARCHAR(20),
    estimated_fare DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE INDEX idx_location_time ON ride_requests(pickup_lat, pickup_lng, created_at);
```

---

### 6.3 Location Update Service (WebSocket)

#### **Why WebSocket?**

**HTTP Polling:**
```
1 million drivers × 1 request/5 seconds
    ↓
200,000 requests/second ❌
```

**WebSocket:**
```
1 million persistent connections
    ↓
Push updates when location changes ✅
```

---

#### **Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    DRIVERS                               │
└────────────────────────┬────────────────────────────────┘
                         │ WebSocket
                         ▼
                ┌─────────────────┐
                │ WebSocket Gateway│
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Location Update │
                │    Service      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Redis   │    │  Kafka   │    │  Trip    │
  │  Cache   │    │  Queue   │    │ Consumer │
  └──────────┘    └──────────┘    └──────────┘
```

---

#### **WebSocket Gateway Responsibilities:**

1. **Authentication:** Validate driver token
2. **Session Stickiness:** Same driver → Same server
3. **Rate Limiting:** Max updates per second

---

#### **Location Data in Redis:**

**Key:** `driver:location:{driverId}`

**Value:**
```json
{
  "driverId": "driver_789",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "geohash": "tdr1y9",
  "status": "AVAILABLE",  // AVAILABLE, ON_RIDE, OFFLINE
  "lastUpdated": "2026-01-21T12:00:00Z"
}
```

**TTL:** 30 seconds (auto-expire if no update)

---

#### **Location Table (PostgreSQL):**

```sql
CREATE TABLE driver_locations (
    driver_id UUID PRIMARY KEY,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    geohash VARCHAR(12),
    status VARCHAR(20),
    last_updated TIMESTAMP
);

CREATE INDEX idx_geohash ON driver_locations(geohash);
CREATE INDEX idx_status ON driver_locations(status);
```

---

### 6.4 Driver Matching Service (Proximity Search)

#### **Prerequisite:** Geohash (Lecture 7)

**Key Concept:** Nearby locations share geohash prefix!

```
McDonald's:  tdr1y9fu
Starbucks:   tdr1y9fv  ← Only last char differs!
```

---

#### **Flow:**

```
User books ride at location (12.9716, 77.5946)
    ↓
Convert to geohash: "tdr1y9"
    ↓
Query Redis for nearby drivers:
    SCAN driver:location:* WHERE geohash LIKE 'tdr1y%'
    ↓
Filter by status: AVAILABLE
    ↓
Sort by distance
    ↓
Get top 5 nearest drivers
    ↓
Send notification to drivers (one at a time)
    ↓
Use Zookeeper lock to prevent double assignment
```

---

#### **Finding Nearby Drivers (Redis):**

```python
def find_nearby_drivers(pickup_lat, pickup_lng, radius_km=5):
    # Convert to geohash
    pickup_geohash = geohash.encode(pickup_lat, pickup_lng)[:6]  # 6 chars ≈ 1km
    
    # Get all neighboring geohash cells (handle edge cases)
    neighbors = geohash.neighbors(pickup_geohash)
    geohashes = [pickup_geohash] + neighbors
    
    nearby_drivers = []
    
    for gh in geohashes:
        # Get drivers in this geohash cell
        pattern = f"driver:location:{gh}*"
        keys = redis.scan(pattern)
        
        for key in keys:
            driver = redis.hgetall(key)
            if driver['status'] == 'AVAILABLE':
                distance = calculate_distance(
                    pickup_lat, pickup_lng,
                    driver['latitude'], driver['longitude']
                )
                if distance <= radius_km:
                    nearby_drivers.append({
                        'driverId': driver['driverId'],
                        'distance': distance
                    })
    
    # Sort by distance and return top 5
    nearby_drivers.sort(key=lambda x: x['distance'])
    return nearby_drivers[:5]
```

---

### 6.5 Zookeeper Locking (Prevent Double Assignment)

#### **Problem:**

```
Server 1: Find drivers near location A → [D1, D2, D3]
Server 2: Find drivers near location A → [D1, D2, D3]

Both send notification to D1
D1 accepts both rides
D1 assigned to 2 rides! ❌
```

---

#### **Solution: Zookeeper Ephemeral Nodes**

**Zookeeper Node Types:**
- **Persistent:** Permanent nodes
- **Ephemeral:** Deleted when session ends (or crashes)

---

#### **Locking Flow:**

```
Server 1 wants to send notification to Driver D1
    ↓
Create ephemeral node: /driver_locks/D1
    ↓
SUCCESS → Send notification to D1
    ↓
Wait for response (10 seconds timeout)
    ↓
IF ACCEPT:
    Create ride, assign driver
    Delete node: /driver_locks/D1
    ↓
IF DENY or TIMEOUT:
    Delete node: /driver_locks/D1
    Move to next driver (D2)
```

---

#### **Concurrent Access:**

```
Server 1: Create /driver_locks/D1 → SUCCESS
Server 2: Create /driver_locks/D1 → EXCEPTION (node exists)

Server 2 skips D1, moves to D2
```

---

#### **Zookeeper Lock Implementation:**

```python
from kazoo.client import KazooClient

zk = KazooClient(hosts='127.0.0.1:2181')
zk.start()

def acquire_driver_lock(driver_id):
    lock_path = f"/driver_locks/{driver_id}"
    try:
        zk.create(lock_path, ephemeral=True)
        return True
    except NodeExistsError:
        return False  # Lock already held

def release_driver_lock(driver_id):
    lock_path = f"/driver_locks/{driver_id}"
    try:
        zk.delete(lock_path)
    except NoNodeError:
        pass  # Already deleted


def assign_driver(ride_request, nearby_drivers):
    for driver in nearby_drivers:
        driver_id = driver['driverId']
        
        # Try to acquire lock
        if acquire_driver_lock(driver_id):
            try:
                # Send notification
                send_push_notification(driver_id, ride_request)
                
                # Wait for response (10 seconds)
                response = wait_for_response(driver_id, timeout=10)
                
                if response == 'ACCEPT':
                    # Assign driver to ride
                    create_ride(ride_request, driver_id)
                    return driver_id
                
            finally:
                # Release lock
                release_driver_lock(driver_id)
    
    return None  # No driver accepted
```

---

#### **Why Zookeeper? (vs Redis Lock)**

| Feature | Zookeeper | Redis Lock |
|---------|-----------|------------|
| **Ephemeral Nodes** | ✅ Auto-delete on crash | ❌ Manual cleanup |
| **Session Monitoring** | ✅ Built-in heartbeat | ❌ Need separate logic |
| **Coordination** | ✅ Designed for this | 🟡 Possible but complex |
| **Performance** | Moderate | Fast |

**Decision:** Zookeeper for coordination, Redis for caching

---

### 6.6 Driver Assignment Flow (Complete)

```
Step 1: User books ride
    POST /v1/rides/book {requestId: "req_123"}
    ↓
Step 2: Driver Matching Service receives request
    ↓
Step 3: Find nearby drivers (Redis + Geohash)
    nearby = [D1, D2, D3, D4, D5]
    ↓
Step 4: Iterate through drivers (top 5)
    FOR driver in nearby:
        ↓
Step 5: Acquire Zookeeper lock
        zk.create("/driver_locks/D1", ephemeral=True)
        ↓
Step 6: Send push notification
        notification.send(D1, ride_request)
        ↓
Step 7: Wait for response (10 seconds)
        ↓
Step 8a: IF ACCEPT:
        - Create ride in Ride DB
        - Notify user via WebSocket
        - Release lock
        - DONE
        ↓
Step 8b: IF DENY or TIMEOUT:
        - Release lock
        - Continue to next driver (D2)
        ↓
Step 9: IF no driver accepts in 1 minute:
        - Return "No drivers available"
```

---

### 6.7 Why Top 5 Drivers?

**Calculation:**

```
Max latency: 1 minute (60 seconds)
Response timeout per driver: 10 seconds
Travel time for notifications: ~2 seconds

Available time: 60 - 2 = 58 seconds
Drivers we can try: 58 / 10 ≈ 5 drivers
```

**Decision:** Send to maximum 5 drivers (one at a time)

---

### 6.8 Ride Database (PostgreSQL)

#### **Estimated Fare Table:**

```sql
CREATE TABLE estimated_fares (
    request_id UUID PRIMARY KEY,
    user_id UUID,
    pickup_lat DECIMAL(10,8),
    pickup_lng DECIMAL(11,8),
    drop_lat DECIMAL(10,8),
    drop_lng DECIMAL(11,8),
    vehicle_type VARCHAR(20),
    estimated_fare DECIMAL(10,2),
    surge_multiplier DECIMAL(3,2),
    currency VARCHAR(10),
    created_at TIMESTAMP,
    expires_at TIMESTAMP
);
```

---

#### **Rides Table:**

```sql
CREATE TABLE rides (
    ride_id UUID PRIMARY KEY,
    request_id UUID REFERENCES estimated_fares(request_id),
    user_id UUID,
    driver_id UUID,
    pickup_lat DECIMAL(10,8),
    pickup_lng DECIMAL(11,8),
    drop_lat DECIMAL(10,8),
    drop_lng DECIMAL(11,8),
    status VARCHAR(20),  -- ASSIGNED, DRIVER_EN_ROUTE, STARTED, COMPLETED, CANCELLED
    actual_fare DECIMAL(10,2),
    distance_km DECIMAL(10,2),
    duration_mins INT,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    created_at TIMESTAMP
);

CREATE INDEX idx_user_id ON rides(user_id);
CREATE INDEX idx_driver_id ON rides(driver_id);
CREATE INDEX idx_status ON rides(status);
```

---

### 6.9 Driver Database (PostgreSQL)

```sql
CREATE TABLE drivers (
    driver_id UUID PRIMARY KEY,
    name VARCHAR(255),
    phone VARCHAR(20),
    email VARCHAR(255),
    vehicle_type VARCHAR(20),
    vehicle_number VARCHAR(20),
    vehicle_model VARCHAR(100),
    rating DECIMAL(2,1),  -- 4.8
    total_rides INT,
    status VARCHAR(20),  -- ONLINE, OFFLINE, ON_RIDE
    created_at TIMESTAMP
);

CREATE INDEX idx_status ON drivers(status);
```

---

### 6.10 Rating Service

#### **Flow:**

```
Ride completed
    ↓
User submits rating (5 stars)
    ↓
Rating Service stores in Rating DB
    ↓
Aggregator Service calculates average
    ↓
Update driver's rating in Driver DB
```

---

#### **Rating DB (PostgreSQL):**

```sql
CREATE TABLE ratings (
    rating_id UUID PRIMARY KEY,
    ride_id UUID REFERENCES rides(ride_id),
    sender_id UUID,  -- user_id or driver_id
    receiver_id UUID,  -- driver_id or user_id
    sender_type VARCHAR(10),  -- USER, DRIVER
    rating INT,  -- 1-5
    feedback TEXT,
    created_at TIMESTAMP
);

CREATE INDEX idx_receiver ON ratings(receiver_id);
```

---

#### **Aggregator Service:**

```python
def update_driver_rating(driver_id):
    # Calculate average rating
    avg_rating = db.query("""
        SELECT AVG(rating) FROM ratings
        WHERE receiver_id = ?
        AND sender_type = 'USER'
    """, driver_id)
    
    # Update driver record
    db.execute("""
        UPDATE drivers
        SET rating = ?
        WHERE driver_id = ?
    """, avg_rating, driver_id)
```

---

### 6.11 Trip Tracking (Route History)

#### **Why Track Route?**

1. **Fare Reconciliation:** Verify actual distance traveled
2. **Refund Calculation:** Compare estimated vs actual fare
3. **Safety:** Track unusual routes

---

#### **Flow:**

```
Driver starts ride
    ↓
Location updates continue via WebSocket
    ↓
Location Update Service pushes to Kafka
    ↓
Trip Consumer Service stores in Trip DB
```

---

#### **Trip Route Table:**

```sql
CREATE TABLE trip_routes (
    id SERIAL PRIMARY KEY,
    ride_id UUID REFERENCES rides(ride_id),
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    timestamp TIMESTAMP
);

CREATE INDEX idx_ride_id ON trip_routes(ride_id);
```

---

#### **Trip Consumer Service:**

```python
def consume_location_updates():
    while True:
        update = kafka.consume("driver_locations")
        
        # Only store if driver is ON_RIDE
        if update['status'] == 'ON_RIDE':
            db.execute("""
                INSERT INTO trip_routes (ride_id, latitude, longitude, timestamp)
                VALUES (?, ?, ?, ?)
            """, update['rideId'], update['latitude'], update['longitude'], update['timestamp'])
```

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│           USERS (Riders)          DRIVERS               │
└────────────────────────┬─────────────┬──────────────────┘
                         │             │
        ┌────────────────┴─────────────┴────────────────┐
        │                                                │
        ▼                                                ▼
  ┌──────────┐                                    ┌──────────┐
  │   API    │                                    │WebSocket │
  │ Gateway  │                                    │ Gateway  │
  └────┬─────┘                                    └────┬─────┘
       │                                               │
       │         ┌─────────────────────────────────────┘
       │         │
       ▼         ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    SERVICES                              │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │  Ride    │  │ Driver   │  │ Location │  │  Surge   │ │
  │  │ Service  │  │ Matching │  │  Update  │  │Calculator│ │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
  │       │             │             │             │        │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │  Rating  │  │ Payment  │  │ Notifi-  │  │   Trip   │ │
  │  │ Service  │  │ Service  │  │ cation   │  │ Consumer │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  └─────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    DATA STORES                           │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │  Redis   │  │Zookeeper │  │  Kafka   │  │  Google  │ │
  │  │  Cache   │  │  Locks   │  │  Queue   │  │   Maps   │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │ Rate DB  │  │ Ride DB  │  │ Driver DB│  │ Rating DB│ │
  │  │(Postgres)│  │(Postgres)│  │(Postgres)│  │(Postgres)│ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  └─────────────────────────────────────────────────────────┘
```

---

## 8. Complete Ride Flow

### 8.1 Fare Estimation

```
Step 1: User enters pickup and drop location
    ↓
Step 2: GET /v1/rides/estimate
    ↓
Step 3: Ride Service calls Google Maps API
    - Distance: 12.5 km
    - ETA: 35 mins
    ↓
Step 4: Ride Service calls Surge Calculator
    - Surge multiplier: 1.3x
    ↓
Step 5: Calculate fare for each vehicle type:
    BIKE: (12.5 × 8) + (35 × 1) + 20 = 155 × 1.3 = 201.5
    CAR:  (12.5 × 12) + (35 × 2) + 50 = 270 × 1.3 = 351
    SUV:  (12.5 × 18) + (35 × 3) + 80 = 410 × 1.3 = 533
    ↓
Step 6: Store in Estimated Fares DB (with request ID)
    ↓
Step 7: Return estimates to user
```

---

### 8.2 Driver Assignment

```
Step 1: User selects CAR, books ride
    POST /v1/rides/book {requestId: "req_123"}
    ↓
Step 2: Driver Matching Service receives request
    ↓
Step 3: Find nearby drivers (Redis + Geohash)
    - Pickup geohash: "tdr1y9"
    - Nearby drivers: [D1, D2, D3, D4, D5]
    ↓
Step 4: Try D1:
    a. Acquire Zookeeper lock: /driver_locks/D1 ✅
    b. Send push notification (FCM/APNS)
    c. Wait 10 seconds...
    d. D1 responds: DENY
    e. Release lock
    ↓
Step 5: Try D2:
    a. Acquire Zookeeper lock: /driver_locks/D2 ✅
    b. Send push notification
    c. Wait 10 seconds...
    d. D2 responds: ACCEPT ✅
    e. Create ride in Ride DB
    f. Release lock
    ↓
Step 6: Notify user via WebSocket:
    "Driver Rahul is on the way! ETA: 5 mins"
    ↓
Step 7: Update driver status: AVAILABLE → ON_RIDE
```

---

### 8.3 During Ride

```
Step 1: Driver navigates to pickup location
    - Location updates every 5 seconds
    - User sees driver on map (via WebSocket)
    ↓
Step 2: Driver arrives, clicks "Start Ride"
    POST /v1/rides/{rideId}/start
    ↓
Step 3: Ride status: DRIVER_EN_ROUTE → STARTED
    ↓
Step 4: Driver navigates to drop location
    - Location updates continue
    - Stored in Trip Routes DB (via Kafka + Trip Consumer)
    ↓
Step 5: Driver arrives, clicks "End Ride"
    POST /v1/rides/{rideId}/end
    ↓
Step 6: Calculate actual fare:
    - Actual distance: 13.2 km (from trip route)
    - Actual duration: 38 mins
    - Fare: 375 INR
    ↓
Step 7: Update ride status: STARTED → COMPLETED
```

---

### 8.4 Payment and Rating

```
Step 1: Payment Service processes payment
    - Payment gateway (Razorpay, Stripe)
    - Deduct from user's wallet/card
    ↓
Step 2: User submits rating (5 stars)
    POST /v1/rides/{rideId}/rating
    ↓
Step 3: Rating stored in Rating DB
    ↓
Step 4: Aggregator updates driver's average rating
    ↓
Step 5: Driver submits rating for user
```

---

## 9. Interview Q&A

### Q1: Why Zookeeper for locking? (vs Redis Lock)

**A:**

**Zookeeper Advantages:**
- ✅ **Ephemeral nodes:** Auto-delete on crash
- ✅ **Session monitoring:** Built-in heartbeat
- ✅ **Designed for coordination**

**Redis Lock:**
- ❌ Need manual cleanup on crash
- ❌ Need separate session logic

### Q2: Why WebSocket for location updates?

**A:**

**HTTP Polling:**
```
1M drivers × 1 req/5s = 200K req/s ❌
```

**WebSocket:**
```
1M persistent connections ✅
Push updates when needed
```

### Q3: How to handle driver timeout?

**A:**

**Flow:**
```
Send notification to D1
    ↓
Wait 10 seconds
    ↓
No response → TIMEOUT
    ↓
Release Zookeeper lock
    ↓
Move to next driver (D2)
```

### Q4: Why geohash for proximity search?

**A:**

**Benefits:**
- ✅ **Nearby locations share prefix:** "tdr1y9fu" and "tdr1y9fv" are close
- ✅ **Efficient lookup:** `LIKE 'tdr1y9%'`
- ✅ **Scalable:** Partition by geohash

### Q5: Why store estimated fare?

**A:**

**Reasons:**
1. **Refund calculation:** Compare estimated vs actual
2. **Price guarantee:** User locked price
3. **Surge visibility:** Show user what they agreed to

### Q6: How to handle surge pricing?

**A:**

**Formula:**
```
surge = requests / available_drivers

if surge > 5: return 2.0x
if surge > 3: return 1.5x
if surge > 2: return 1.3x
else: return 1.0x
```

### Q7: Why top 5 drivers?

**A:**

**Calculation:**
```
Max latency: 60 seconds
Timeout per driver: 10 seconds
Max drivers: 60 / 10 = 6

Buffer: 1 driver
Result: 5 drivers
```

### Q8: How to handle fare difference?

**A:**

**Scenario:**
```
Estimated: 350 INR
Actual: 375 INR (longer route due to traffic)
```

**Solution:**
- Store estimated fare in DB
- Compare with actual fare
- If difference > 20%, flag for review
- Process refund if needed

### Q9: How to scale driver matching?

**A:**

**Strategies:**
1. **Partition by geohash:** Each server handles specific regions
2. **Redis cluster:** Shard driver locations
3. **Multiple Zookeepers:** Regional clusters

### Q10: Security considerations?

**A:**

1. **Driver verification:** Background checks, document verification
2. **User verification:** OTP for phone number
3. **Ride tracking:** Store complete route
4. **Emergency button:** Notify authorities
5. **Rating system:** Flag problematic users/drivers

---

## 10. Key Takeaways

✅ **Geohash:** Find nearby drivers efficiently  
✅ **Zookeeper:** Distributed lock to prevent double assignment  
✅ **WebSocket:** Real-time location updates  
✅ **Top 5 Drivers:** Match 1-minute latency requirement  
✅ **Surge Pricing:** Demand/supply ratio  
✅ **Two-Phase:** Estimation → Booking  
✅ **Push Notifications:** FCM (Android), APNS (iOS)  
✅ **Trip Route Tracking:** Fare reconciliation  
✅ **Redis Cache:** Driver locations (TTL 30s)  
✅ **PostgreSQL:** Transactional data (rides, payments)  

---

## Summary

**Architecture Highlights:**
- 8+ microservices
- 5+ databases (PostgreSQL)
- Redis (driver locations)
- Zookeeper (distributed locking)
- Kafka (location updates)
- WebSocket (real-time tracking)
- Google Maps API (distance/ETA)

**Driver Matching Flow:**
```
User books → Find nearby (geohash) → Lock (Zookeeper) → Notify → Wait → Assign/Retry
```

**Key Algorithms:**
- **Geohash:** Proximity search
- **Surge Pricing:** Demand/supply ratio
- **Top-K Selection:** Limit drivers to match latency

**Performance:**
- Driver assignment: < 1 minute
- Location updates: Every 5-10 seconds
- Millions of concurrent users

**End of Lecture 11**
