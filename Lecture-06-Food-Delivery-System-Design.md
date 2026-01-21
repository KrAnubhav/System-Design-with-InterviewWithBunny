# Lecture 6: Food Delivery System Design (like Zomato/Swiggy)

> **Lecture:** 06  
> **Topic:** System Design  
> **Application:** Food Delivery Platform (Zomato, Swiggy, Uber Eats)  
> **Scale:** 50M Users, 1M Restaurants  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-05-Hotel-Booking-System-Design|Lecture 5: Hotel Booking System]]

---

## 1. Brief Introduction

### What is a Food Delivery System?

A **food delivery platform** connects users with restaurants and delivery partners to order food online and get it delivered to their location.

**Examples:** Zomato, Swiggy, Uber Eats, DoorDash, Grubhub

### Problem It Solves

1. **Restaurant Discovery:** Find nearby restaurants
2. **Menu Browsing:** View available dishes
3. **Online Ordering:** Place orders with payment
4. **Order Tracking:** Real-time delivery updates
5. **Delivery Logistics:** Assign and track delivery partners

### Three Core Actors

This system involves **three distinct actors:**

1. **👤 User** - Orders food
2. **🍽️ Restaurant** - Accepts orders, prepares food
3. **🚴 Delivery Partner** - Picks up and delivers food

⚠️ **Interview Tip:**

Since covering all three actors is **time-consuming**, ask your interviewer which aspect to focus on:
- User flow (ordering)
- Restaurant flow (order management)
- Delivery partner flow (assignment & tracking)

**In this lecture:** We'll cover **all three** for comprehensive understanding!

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

#### **User Flow:**

1. **Sign Up & Login**
   - Create account
   - Login/logout

2. **Browse Restaurants**
   - View nearby restaurants (within 5-10 km)
   - See restaurant details (rating, cuisine, delivery time)

3. **Search**
   - Search by restaurant name
   - Search by dish/menu item

4. **View Menu**
   - See available dishes
   - View prices, images, descriptions

5. **Add to Cart**
   - Select multiple items
   - Modify quantities

6. **Place Order**
   - Make payment
   - Confirm order

7. **Track Order**
   - Real-time delivery partner location
   - Order status updates

#### **Restaurant Flow:**

1. **Register Restaurant**
   - Upload restaurant details
   - Add menu items

2. **Receive Orders**
   - Get notified of new orders
   - Accept/reject orders

3. **Update Order Status**
   - Mark as "preparing"
   - Mark as "ready for pickup"

#### **Delivery Partner Flow:**

1. **Register as Delivery Partner**
   - Provide details, documents

2. **Update Location**
   - Send GPS location every 5-10 seconds

3. **Receive Delivery Requests**
   - Get notified of nearby pickups
   - Accept/reject requests

4. **Update Delivery Status**
   - Mark as "picked up"
   - Mark as "delivered"

### 2.2 Non-Functional Requirements

1. **Scale**
   - **50 million users** globally
   - **1 million restaurants**
   - **High traffic** during lunch/dinner hours

2. **CAP Theorem - Split Approach**

   **High Availability (Search & Browse):**
   - ✅ Restaurant listing should always work
   - ✅ Users can browse anytime
   - ❌ Slight delay in menu updates is acceptable

   **High Consistency (Payment & Ordering):**
   - ✅ Payment must be accurate
   - ✅ No duplicate orders
   - ✅ Correct order assignment

   **Summary:** **AP for search, CP for ordering**

3. **Low Latency**
   - Restaurant search: < 500ms
   - Order placement: < 2 seconds
   - Real-time location updates: < 5 seconds

4. **Real-Time Updates**
   - Delivery partner location (every 5-10 seconds)
   - Order status notifications

---

## 3. Core Entities

1. **User** - Customer ordering food
2. **Restaurant** - Food provider
3. **Menu Item** - Dishes available
4. **Cart** - User's selected items
5. **Order** - Placed order
6. **Payment** - Transaction details
7. **Delivery Partner** - Driver
8. **Delivery** - Delivery assignment

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
  "phone": "+1234567890",
  "address": "123 Main St, Bangalore"
}
```

**Additional APIs** (mention verbally):
- `POST /v1/users/login`
- `POST /v1/users/logout`
- `PUT /v1/users/profile`

---

### 4.2 Get Nearby Restaurants

```
GET /v1/restaurants/nearby?lat={latitude}&lon={longitude}&radius={km}
```

**Query Parameters:**
- `lat` (required): User's latitude
- `lon` (required): User's longitude
- `radius` (optional): Search radius in km (default: 5)

**Response:**
```json
{
  "restaurants": [
    {
      "restaurantId": "rest_123",
      "name": "Pizza Palace",
      "cuisine": "Italian",
      "rating": 4.5,
      "deliveryTime": "30 mins",
      "thumbnail": "https://cdn.example.com/rest_123.jpg"
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 5
  }
}
```

⚠️ **Important:** Use pagination!

---

### 4.3 Search Restaurants

```
GET /v1/restaurants/search?query={keyword}&lat={latitude}&lon={longitude}
```

**Query Parameters:**
- `query` (required): Restaurant name or dish name
- `lat` (required): User's latitude
- `lon` (required): User's longitude

**Response:**
```json
{
  "restaurants": [
    {
      "restaurantId": "rest_456",
      "name": "Biryani House",
      "matchedDish": "Chicken Biryani",
      "rating": 4.7
    }
  ]
}
```

---

### 4.4 Get Restaurant Details

```
GET /v1/restaurants/{restaurantId}
```

**Response:**
```json
{
  "restaurantId": "rest_123",
  "name": "Pizza Palace",
  "address": "456 MG Road, Bangalore",
  "cuisine": "Italian",
  "rating": 4.5,
  "deliveryTime": "30 mins",
  "images": ["url1", "url2"]
}
```

---

### 4.5 Get Restaurant Menu

```
GET /v1/restaurants/{restaurantId}/menu
```

**Response:**
```json
{
  "menu": [
    {
      "itemId": "item_789",
      "name": "Margherita Pizza",
      "description": "Classic cheese pizza",
      "price": 299,
      "currency": "INR",
      "category": "Pizza",
      "image": "https://cdn.example.com/item_789.jpg"
    }
  ]
}
```

---

### 4.6 Add to Cart

```
POST /v1/cart
```

**Request Body:**
```json
{
  "restaurantId": "rest_123",
  "itemId": "item_789",
  "quantity": 2
}
```

**Response:**
```json
{
  "cartId": "cart_101"
}
```

⚠️ **Note:** User ID sent in JWT header (not in body)

**Additional APIs:**
- `PUT /v1/cart/{cartId}` - Update cart
- `DELETE /v1/cart/{cartId}` - Clear cart

---

### 4.7 Place Order

```
POST /v1/orders
```

**Request Body:**
```json
{
  "cartId": "cart_101"
}
```

**Response:**
```json
{
  "orderId": "order_202",
  "status": "pending"
}
```

⚠️ **Note:** This triggers payment flow internally

---

### 4.8 Payment (Internal)

```
POST /v1/payments
```

**Request Body:**
```json
{
  "orderId": "order_202",
  "amount": 598,
  "currency": "INR",
  "paymentMethod": "card"
}
```

**Response:**
```json
{
  "paymentId": "pay_303",
  "status": "success"
}
```

---

### 4.9 Track Order

```
GET /v1/orders/{orderId}/track
```

**Response:**
```json
{
  "orderId": "order_202",
  "status": "out_for_delivery",
  "deliveryPartner": {
    "name": "Ravi Kumar",
    "phone": "+919876543210",
    "location": {
      "lat": 12.9716,
      "lon": 77.5946
    }
  },
  "estimatedTime": "10 mins"
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture (All Three Actors)

```
┌─────────────────────────────────────────────────────────┐
│                    50M Users                             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                │  Load Balancer  │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┬──────────┐
        │                │                │          │
        ▼                ▼                ▼          ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Search  │    │  Cart    │ │  Order   │
  │ Service  │    │ Service  │    │ Service  │ │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘ └────┬─────┘
       │               │                │            │
       ▼               ▼                ▼            ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │ User DB  │    │Elastic   │    │ Cart DB  │ │ Order DB │
  │ (MySQL)  │    │ Search   │    │(Postgres)│ │ (MySQL)  │
  └──────────┘    └────▲─────┘    └──────────┘ └──────────┘
                       │
                       │ CDC
                       │
                  ┌────┴─────┐
                  │Restaurant│
                  │   DB     │
                  │(Postgres)│
                  └────▲─────┘
                       │
                       │
┌──────────────────────┼────────────────────────────────────┐
│         Restaurants  │                                    │
└──────────────────────┼────────────────────────────────────┘
                       │
                       ▼
                ┌─────────────────┐
                │ Restaurant       │
                │   Service        │
                └─────────────────┘


┌─────────────────────────────────────────────────────────┐
│              Delivery Partners                           │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Kafka Gateway   │
                └────────┬────────┘
                         │
                         ▼
                    Kafka Broker
                         │
                         ▼
                ┌─────────────────┐
                │   Location      │
                │Update Service   │
                └────────┬────────┘
                         │
                         ▼
                    Redis Cache
                         │
                         ▼
                  ┌──────────────┐
                  │  Driver DB   │
                  │ (PostgreSQL) │
                  └──────────────┘
```

### 5.2 Service Responsibilities

**User Services:**
- User Service: Registration, login
- Search Service: Find restaurants
- Cart Service: Manage cart
- Order Service: Place orders
- Payment Service: Process payments

**Restaurant Services:**
- Restaurant Service: Manage restaurant data
- Order Service: Accept/reject orders

**Delivery Services:**
- Location Update Service: Track driver locations
- Delivery Matching Service: Assign drivers
- Delivery Acceptance Service: Get driver confirmation

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
    address TEXT,
    created_at TIMESTAMP
);
```

**Flow:**
1. User registers → Store in MySQL
2. User logs in → Validate → Return JWT token
3. JWT used in all API calls

---

### 6.2 Restaurant Database (PostgreSQL)

**Why PostgreSQL?**
- **Relational data:** Restaurant → Menu items
- **ACID compliance:** Critical for order accuracy

#### **Table 1: Restaurants**

```sql
CREATE TABLE restaurants (
    restaurant_id UUID PRIMARY KEY,
    name VARCHAR(255),
    address TEXT,
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    cuisine VARCHAR(100),
    status VARCHAR(20),  -- open, closed, maintenance
    rating DECIMAL(3, 2),
    delivery_time INT,  -- in minutes
    images TEXT[],  -- S3 URLs
    created_at TIMESTAMP
);
```

#### **Table 2: Menu Items**

```sql
CREATE TABLE menu_items (
    item_id UUID PRIMARY KEY,
    restaurant_id UUID REFERENCES restaurants(restaurant_id),
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10, 2),
    currency VARCHAR(3),
    category VARCHAR(100),  -- Pizza, Burger, Dessert
    image_url TEXT,  -- S3 URL
    is_available BOOLEAN,
    created_at TIMESTAMP
);
```

---

### 6.3 Image Storage (S3 Bucket)

**What to store:**
- Restaurant images
- Menu item images

**Why S3?**
- Scalable
- CDN integration
- Cost-effective

---

### 6.4 Search Service (Elasticsearch)

**Why Elasticsearch?**
- **Text search:** Restaurant name, dish name
- **Geo search:** Nearby restaurants
- **Fast:** Sub-second queries

**Index Structure:**
```json
{
  "restaurantId": "rest_123",
  "name": "Pizza Palace",
  "cuisine": "Italian",
  "location": {
    "lat": 12.9716,
    "lon": 77.5946
  },
  "menuItems": ["Margherita Pizza", "Pepperoni Pizza"],
  "rating": 4.5
}
```

**Query 1: Nearby Restaurants**
```json
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": {
        "lat": 12.9716,
        "lon": 77.5946
      }
    }
  }
}
```

**Query 2: Search by Name/Dish**
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Pizza"
          }
        },
        {
          "geo_distance": {
            "distance": "10km",
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

**CDC Pipeline:**
```
PostgreSQL (Restaurants + Menu)
    │
    │ Debezium
    ▼
Kafka
    │
    ▼
Consumer Service (Merge tables)
    │
    ▼
Elasticsearch
```

---

### 6.5 Cart Service

**Database:** PostgreSQL

**Schema:**
```sql
CREATE TABLE carts (
    cart_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    restaurant_id UUID REFERENCES restaurants(restaurant_id),
    items JSONB,  -- [{itemId, quantity}, ...]
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Why restaurant_id?**
- **Constraint:** User can only order from **one restaurant at a time**
- Switching restaurants → Clear cart

**Example:**
```json
{
  "cartId": "cart_101",
  "userId": "user_789",
  "restaurantId": "rest_123",
  "items": [
    {
      "itemId": "item_456",
      "quantity": 2
    },
    {
      "itemId": "item_789",
      "quantity": 1
    }
  ]
}
```

---

### 6.6 Order Service (Event-Driven Architecture!)

This is the **MOST COMPLEX** part of the system!

#### **Architecture:**

```
User clicks "Place Order"
    │
    ▼
Order Service
    │
    ▼
Payment Service
    │
    ▼
Payment Gateway (Razorpay, Stripe)
    │
    ▼
Payment Success/Failure
    │
    ▼
Publish event to Kafka
    │
    ├─> Order Consumer → Update Order DB
    │
    ├─> Notification Consumer → Notify user
    │
    └─> Restaurant Acceptance Service → Notify restaurant
```

---

#### **Step 1: Payment Service**

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    order_id UUID,
    amount DECIMAL(10, 2),
    currency VARCHAR(3),
    status VARCHAR(20),  -- success, failed, pending
    transaction_id VARCHAR(255),
    created_at TIMESTAMP
);
```

**Flow:**
1. User enters card details
2. Payment Service calls Payment Gateway
3. Gateway returns success/failure
4. Update `payments` table

---

#### **Step 2: Kafka Event (Order Placed)**

**Event:**
```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "order_202",
  "userId": "user_789",
  "restaurantId": "rest_123",
  "items": [...],
  "amount": 598,
  "timestamp": "2026-01-21T12:00:00Z"
}
```

**Consumers:**

**1. Order Consumer:**
- Insert into `orders` table

**2. Notification Consumer:**
- Send email/SMS to user: "Order placed!"

**3. Restaurant Acceptance Service:**
- Send notification to restaurant
- Wait for acceptance

---

#### **Step 3: Restaurant Accepts Order**

**Flow:**
```
Restaurant clicks "Accept"
    │
    ▼
Order Service (Restaurant)
    │
    ▼
Publish event to Kafka
    │
    ├─> Order Consumer → Update status to "accepted"
    │
    ├─> Notification Consumer → Notify user
    │
    └─> Delivery Matching Service → Assign driver
```

**Event:**
```json
{
  "eventType": "ORDER_ACCEPTED",
  "orderId": "order_202",
  "restaurantId": "rest_123",
  "timestamp": "2026-01-21T12:05:00Z"
}
```

---

### 6.7 Order Database

**Schema:**
```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    restaurant_id UUID REFERENCES restaurants(restaurant_id),
    items JSONB,
    total_amount DECIMAL(10, 2),
    currency VARCHAR(3),
    status VARCHAR(50),  -- placed, accepted, preparing, ready, picked_up, delivered
    driver_id UUID,  -- Assigned later
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Status Flow:**
```
placed → accepted → preparing → ready → picked_up → delivered
```

---

### 6.8 Delivery Matching Service (Proximity Search!)

#### **Challenge: Find Nearest Available Driver**

**Requirements:**
1. Driver must be **near the restaurant** (< 5 km)
2. Driver must be **idle** (not on another delivery)
3. **Real-time** driver locations

---

#### **Solution: Geohash + Redis Cache**

**Why Geohash?**
- **Fast proximity search**
- **Efficient encoding** of lat/lon
- **Better than Quad Tree** for this use case

**Geohash Explained:**

```
Geohash: "tdr1y"
    ↓
Lat: 12.9716, Lon: 77.5946
    ↓
Precision: ~5 km radius
```

**How it works:**
1. Encode driver location as geohash
2. Store in Redis: `geohash:tdr1y → [driver1, driver2, driver3]`
3. Search: Get all drivers with same geohash prefix

**Example:**
```
Restaurant location: 12.9716, 77.5946 → Geohash: "tdr1y"
Driver 1 location: 12.9720, 77.5950 → Geohash: "tdr1y" ✅ Match!
Driver 2 location: 12.9800, 77.6000 → Geohash: "tdr2x" ❌ No match
```

---

#### **Driver Location Updates (High Frequency!)**

**Problem:**
- 10,000 drivers × 1 update/10 seconds = **1,000 updates/second**
- Too much load for database!

**Solution:** **Kafka + Redis**

```
Driver sends location update
    │
    ▼
Kafka Gateway
    │
    ▼
Kafka Broker (Location topic)
    │
    ▼
Location Update Service (Consumer)
    │
    ▼
Redis Cache (with TTL)
```

**Why Kafka?**
- **High throughput:** Handle 1,000+ updates/second
- **Decoupled:** Drivers don't wait for DB write

**Why Redis?**
- **Fast writes:** < 1ms
- **TTL:** Auto-remove stale locations (if driver offline)

---

#### **Redis Schema:**

**Key:** `driver:{driverId}`

**Value:**
```json
{
  "driverId": "driver_456",
  "location": {
    "lat": 12.9716,
    "lon": 77.5946
  },
  "geohash": "tdr1y",
  "status": "idle",  // idle, busy
  "lastUpdated": "2026-01-21T12:10:00Z"
}
```

**TTL:** 30 seconds (if no update, assume driver offline)

---

#### **Geohash Index in Redis:**

**Key:** `geohash:{geohash}`

**Value:** Set of driver IDs

**Example:**
```
geohash:tdr1y → {driver_456, driver_789, driver_101}
```

**Search Algorithm:**
```python
def find_nearby_drivers(restaurant_lat, restaurant_lon, radius_km):
    # 1. Encode restaurant location
    geohash = encode_geohash(restaurant_lat, restaurant_lon, precision=5)
    
    # 2. Get all drivers with same geohash
    driver_ids = redis.smembers(f"geohash:{geohash}")
    
    # 3. Filter by status
    idle_drivers = []
    for driver_id in driver_ids:
        driver = redis.get(f"driver:{driver_id}")
        if driver['status'] == 'idle':
            idle_drivers.append(driver)
    
    # 4. Sort by distance
    idle_drivers.sort(key=lambda d: distance(restaurant_lat, restaurant_lon, d['lat'], d['lon']))
    
    return idle_drivers[:10]  # Top 10 nearest
```

---

### 6.9 Delivery Acceptance Service

**Problem:**
- Delivery Matching Service finds 10 nearby drivers
- **Which one accepts first?**

**Solution:** **Broadcast to all, first-come-first-served**

```
Delivery Matching Service
    │
    ▼
Delivery Acceptance Service
    │
    ├─> Send notification to Driver 1
    ├─> Send notification to Driver 2
    ├─> Send notification to Driver 3
    └─> ...
    
First driver to accept → Assigned!
```

**Flow:**
```
Driver clicks "Accept"
    │
    ▼
API Gateway
    │
    ▼
Delivery Acceptance Service
    │
    ▼
Publish event to Kafka
    │
    ├─> Order Consumer → Update order.driver_id
    │
    ├─> Notification Consumer → Notify user
    │
    └─> Cancel notifications to other drivers
```

---

### 6.10 Driver Database

**Schema:**
```sql
CREATE TABLE drivers (
    driver_id UUID PRIMARY KEY,
    name VARCHAR(255),
    phone VARCHAR(20),
    vehicle_type VARCHAR(50),  -- bike, car
    license_number VARCHAR(100),
    status VARCHAR(20),  -- idle, busy, offline
    current_location_lat DECIMAL(10, 8),
    current_location_lon DECIMAL(11, 8),
    created_at TIMESTAMP
);
```

⚠️ **Note:** `current_location` is updated from Redis (batch updates every 1 minute)

---

### 6.11 Real-Time Tracking (WebSocket!)

**Challenge:** User needs **real-time** delivery partner location

**Solution:** **WebSocket Connection**

```
Driver updates location (every 5 seconds)
    │
    ▼
Kafka Gateway
    │
    ▼
Kafka Broker
    │
    ▼
WebSocket Manager
    │
    ▼
WebSocket Gateway
    │
    ▼
User's Browser (Live map update)
```

**WebSocket Manager:**
- Manages connections efficiently
- Opens connection when order status = "picked_up"
- Closes connection when order status = "delivered"

**Why WebSocket?**
- **Real-time:** No polling needed
- **Efficient:** Single connection, multiple updates

---

### 6.12 Order Confirmation Service

**Purpose:** Notify driver when food is ready

**Flow:**
```
Restaurant clicks "Ready for Pickup"
    │
    ▼
Order Service (Restaurant)
    │
    ▼
Publish event to Kafka
    │
    ▼
Order Confirmation Service
    │
    ▼
Send notification to assigned driver
```

**Event:**
```json
{
  "eventType": "ORDER_READY",
  "orderId": "order_202",
  "restaurantId": "rest_123",
  "driverId": "driver_456",
  "timestamp": "2026-01-21T12:30:00Z"
}
```

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USER FLOW                             │
└─────────────────────────────────────────────────────────┘

Users → API Gateway → Load Balancer
    │
    ├─> User Service → User DB (MySQL)
    │
    ├─> Search Service → Elasticsearch ← CDC ← Restaurant DB
    │
    ├─> Cart Service → Cart DB (PostgreSQL)
    │
    └─> Order Service → Payment Service → Payment Gateway
            │
            ▼
        Kafka Broker (ORDER_PLACED)
            │
            ├─> Order Consumer → Order DB
            │
            ├─> Notification Consumer → Email/SMS
            │
            └─> Restaurant Acceptance Service
                    │
                    ▼
                Kafka Broker (ORDER_ACCEPTED)
                    │
                    ├─> Order Consumer → Update status
                    │
                    └─> Delivery Matching Service
                            │
                            ▼
                        Redis Cache (Geohash)
                            │
                            ▼
                        Delivery Acceptance Service
                            │
                            ▼
                        Kafka Broker (DRIVER_ASSIGNED)
                            │
                            └─> Order Consumer → Update driver_id


┌─────────────────────────────────────────────────────────┐
│                 RESTAURANT FLOW                          │
└─────────────────────────────────────────────────────────┘

Restaurants → API Gateway
    │
    ├─> Restaurant Service → Restaurant DB (PostgreSQL)
    │
    └─> Order Service → Kafka (ORDER_ACCEPTED, ORDER_READY)


┌─────────────────────────────────────────────────────────┐
│              DELIVERY PARTNER FLOW                       │
└─────────────────────────────────────────────────────────┘

Drivers → Kafka Gateway
    │
    ▼
Kafka Broker (LOCATION_UPDATE)
    │
    ▼
Location Update Service
    │
    ├─> Redis Cache (with TTL)
    │
    └─> Driver DB (batch updates)

Drivers → API Gateway
    │
    └─> Delivery Acceptance Service → Kafka (DRIVER_ASSIGNED)


┌─────────────────────────────────────────────────────────┐
│              REAL-TIME TRACKING                          │
└─────────────────────────────────────────────────────────┘

Driver location updates → Kafka → WebSocket Manager → User
```

---

## 8. Interview-Style Q&A

### Q1: Why three separate flows (User, Restaurant, Driver)?

**A:**
- **Different actors** with different needs
- **Separation of concerns:** Easier to scale independently
- **Security:** Restaurant can't access user data directly

### Q2: Why Elasticsearch for search?

**A:**
- **Text search:** Restaurant name, dish name
- **Geo search:** Nearby restaurants
- **Fast:** Sub-second queries
- **Alternative:** Quad Tree (no text search), PostgreSQL GIS (slower)

### Q3: Why Geohash over Quad Tree for driver matching?

**A:**

| Feature | Geohash | Quad Tree |
|---------|---------|-----------|
| Memory | Low (Redis) | High (entire tree in memory) |
| Updates | Fast (just update hash) | Slow (rebuild tree) |
| Frequency | 10,000 updates/sec | Not suitable |
| Real-time | ✅ Yes | ❌ No |

**Decision:** Geohash (high-frequency updates)

### Q4: Why Kafka for driver location updates?

**A:**
- **High throughput:** 1,000+ updates/second
- **Decoupled:** Drivers don't wait for DB write
- **Reliable:** No data loss
- **Asynchronous:** Non-blocking

**Without Kafka:**
- Direct DB writes → Slow (100ms/write)
- 1,000 updates/sec → Database overload ❌

### Q5: Why Redis cache for driver locations?

**A:**
- **Fast writes:** < 1ms (vs 100ms for DB)
- **TTL:** Auto-remove offline drivers
- **In-memory:** No disk I/O
- **Geohash indexing:** Fast proximity search

**Flow:**
1. Driver sends location → Kafka
2. Consumer writes to Redis (fast!)
3. Batch update to DB every 1 minute

### Q6: How to handle multiple drivers accepting same order?

**A:**
**Problem:** 10 drivers notified, 3 click "Accept" simultaneously

**Solution:** **Atomic operation in Kafka consumer**

```python
def assign_driver(order_id, driver_id):
    # Check if order already assigned
    order = db.get_order(order_id)
    
    if order.driver_id is not None:
        return False  # Already assigned
    
    # Atomic update (use DB transaction)
    success = db.update_order(
        order_id=order_id,
        driver_id=driver_id,
        condition="driver_id IS NULL"  # Only if not assigned
    )
    
    if success:
        # Cancel notifications to other drivers
        cancel_notifications(order_id, exclude=driver_id)
        return True
    else:
        return False  # Another driver was faster
```

### Q7: Why WebSocket for real-time tracking?

**A:**

**Alternatives:**

**1. Polling (HTTP):**
- User requests location every 5 seconds
- ❌ Inefficient (many unnecessary requests)
- ❌ High latency

**2. WebSocket:**
- ✅ Single connection
- ✅ Server pushes updates
- ✅ Low latency (< 1 second)

**When to open WebSocket:**
- Order status = "picked_up"

**When to close:**
- Order status = "delivered"

### Q8: Why event-driven architecture (Kafka)?

**A:**

**Without Kafka (Direct DB updates):**
```
Order Service
    ├─> Update Order DB
    ├─> Send notification
    ├─> Notify restaurant
    └─> Assign driver

If any step fails → Inconsistency!
```

**With Kafka (Event-driven):**
```
Order Service → Publish event → Kafka
    ↓
Multiple consumers process independently
    ├─> Order Consumer
    ├─> Notification Consumer
    └─> Restaurant Consumer

Each can retry on failure → Eventual consistency!
```

### Q9: How to handle peak traffic (lunch/dinner)?

**A:**

**Strategies:**

1. **Horizontal Scaling:**
   - Add more instances of Order Service
   - Load balancer distributes traffic

2. **Kafka Partitioning:**
   - Partition by `restaurantId`
   - Parallel processing

3. **Redis Cluster:**
   - Shard driver locations by geohash
   - Distribute load

4. **Database Sharding:**
   - Shard Order DB by `userId`
   - Reduce load per shard

### Q10: Security considerations?

**A:**
1. **JWT Tokens:** Authenticate all API calls
2. **HTTPS:** Encrypt communication
3. **PCI Compliance:** For payment data
4. **Rate Limiting:** Prevent abuse
5. **Driver Verification:** Background checks
6. **Order Validation:** Prevent fake orders

---

## 9. Key Takeaways

✅ **Three actors:** User, Restaurant, Delivery Partner  
✅ **Elasticsearch:** Text + geo search  
✅ **Geohash + Redis:** Fast driver matching  
✅ **Kafka:** Event-driven architecture  
✅ **WebSocket:** Real-time tracking  
✅ **High-frequency updates:** Kafka → Redis (not DB)  
✅ **Proximity search:** Geohash (better than Quad Tree for real-time)  
✅ **Split CAP:** AP for search, CP for ordering  

---

## Summary

**Architecture Highlights:**
- 10+ microservices
- 6 databases (MySQL x2, PostgreSQL x3, Elasticsearch x1)
- Redis for driver locations (with TTL)
- Kafka for event streaming
- S3 for images
- WebSocket for real-time tracking

**Proximity Search:**
- Elasticsearch for restaurants (text + geo)
- Geohash for drivers (real-time updates)

**Event-Driven Flow:**
- Order placed → Kafka → Multiple consumers
- Restaurant accepts → Kafka → Assign driver
- Driver assigned → Kafka → Notify user

**Real-Time Updates:**
- Driver location: Kafka → Redis → WebSocket → User
- Update frequency: Every 5-10 seconds

**Performance:**
- Restaurant search: < 500ms
- Order placement: < 2 seconds
- Location updates: 1,000+ updates/second

**Scale:**
- 50M users
- 1M restaurants
- 10,000+ active drivers

**End of Lecture 6**
