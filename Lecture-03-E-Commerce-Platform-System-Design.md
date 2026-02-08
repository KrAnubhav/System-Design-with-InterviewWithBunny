# Lecture 3: E-Commerce Platform System Design (like Amazon)

> **Lecture:** 03  
> **Topic:** System Design  
> **Application:** E-Commerce Platform (Amazon, Flipkart)  
> **Scale:** 10M Monthly Active Users, 10 Orders/Second  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-02-Ticket-Booking-System-Design|Lecture 2: Ticket Booking System]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Low-Level Architecture](#7-complete-low-level-architecture)
8. [Interview-Style Q&A](#8-interview-style-qa)
9. [Key Takeaways](#9-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is an E-Commerce Platform?

An **e-commerce platform** is an online service that allows users to:
- Browse and search for products
- View product details and reviews
- Add items to shopping cart
- Make secure payments
- Track order status

**Examples:** Amazon, Flipkart, eBay, Walmart

### Problem It Solves

1. **Convenient Shopping:** Purchase from anywhere, anytime
2. **Wide Selection:** Access to millions of products
3. **Price Comparison:** Easy to compare prices
4. **Order Tracking:** Real-time order status updates
5. **Secure Payments:** Safe transaction processing
6. **Inventory Management:** Prevent overselling limited stock

### Scope Clarification

⚠️ **Important:** E-commerce is HUGE! Clarify with interviewer:
- **User flow** (search, cart, checkout) ← We focus on this
- **Seller flow** (product listing, inventory)
- **Warehouse management** (logistics, shipping)
- **Analytics** (recommendations, trending products)

---

## 2. Functional and Non-Functional Requirements
<img width="706" height="278" alt="image" src="https://github.com/user-attachments/assets/0c8fdecc-b138-4a07-be16-d46ad2736c7e" />


### 2.1 Functional Requirements

**User Flow (Primary Focus):**

1. **Search Products**
   - Search by title, keywords, category
   - Filter by price, rating, availability

2. **View Product Details**
   - Description, specifications, images
   - Reviews, ratings, manufacturer info
   - Available quantity, pricing

3. **Add to Cart**
   - Select quantity
   - Update or remove items
   - View cart total

4. **Checkout & Payment**
   - Make secure payment
   - Apply discount codes (optional)
   - Select delivery address

5. **Track Order Status**
   - View order history
   - Real-time status updates
   - Delivery tracking

6. **Handle Limited Stock (Critical!)**
   - Prevent overselling during flash sales
   - Handle concurrent purchases efficiently
   - Example: 1000 users trying to buy 5 iPhones

### 2.2 Non-Functional Requirements

1. **Scale**
   - **10 million monthly active users**
   - **10 orders per second**
   - Must handle peak traffic (flash sales, holiday season)

2. **CAP Theorem - Mixed Approach**

   **High Availability:**
   - ✅ **Search service** (users must always be able to browse)
   - ✅ **Product viewing** (details should load quickly)
   
   **High Consistency:**
   - ✅ **Payment & Order Management** (no duplicate charges)
   - ✅ **Inventory Management** (no overselling)

   **Why this split?**
   - Browsing = 80% of traffic (read-heavy, eventual consistency okay)
   - Ordering = 20% of traffic (write-heavy, strict consistency required)

3. **Low Latency**
   - Search results: **< 200ms**
   - Product details: **< 300ms**
   - Checkout: **< 2 seconds**

4. **Reliability**
   - No double orders
   - Payment failures should release cart
   - Inventory must be accurate

---

## 3. Core Entities

1. **User** - Customer account
2. **Product** - Items for sale
3. **Cart** - Shopping basket
4. **Order** - Completed purchase
5. **Checkout** - Payment process

---

## 4. API Design

### 4.1 Search Products

```
GET /v1/products/search?term={keyword}
```

**Response:**
```json
{
  "products": [
    {
      "productId": "prod_123",
      "title": "iPhone 15 Pro",
      "price": 999.99,
      "currency": "USD",
      "thumbnail": "https://cdn.example.com/iphone.jpg",
      "rating": 4.5,
      "availableQty": 10
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 50
  }
}
```

⚠️ **Important:** Use pagination! Don't return all results.

---

### 4.2 View Product Details

```
GET /v1/products/{productId}
```

**Response:**
```json
{
  "productId": "prod_123",
  "title": "iPhone 15 Pro",
  "description": "Latest iPhone with A17 chip...",
  "price": 999.99,
  "currency": "USD",
  "images": ["url1", "url2"],
  "specifications": {...},
  "reviews": [...],
  "availableQty": 10
}
```

---

### 4.3 Add to Cart

```
POST /v1/cart/add
```

**Headers:**
```
Authorization: Bearer {JWT_TOKEN}
```

**Request Body:**
```json
{
  "productId": "prod_123",
  "quantity": 2
}
```

**Response:**
```json
{
  "cartId": "cart_456"
}
```

**Additional APIs** (mention verbally to save time):
- `PUT /v1/cart/update` - Update cart item
- `DELETE /v1/cart/remove` - Remove from cart

---

### 4.4 Checkout (Two-Step Process)

**Step 1: Checkout**

```
POST /v1/checkout
```

**Request Body:**
```json
{
  "cartId": "cart_456",
  "products": [
    {
      "productId": "prod_123",
      "quantity": 2,
      "price": 999.99
    }
  ]
}
```

**Response:**
```json
{
  "orderId": "order_789"
}
```

**Step 2: Payment**

```
POST /v1/payment
```

**Request Body:**
```json
{
  "orderId": "order_789",
  "paymentMethod": "card",
  "cardDetails": {...},
  "amount": 1999.98
}
```

**Response:**
```json
{
  "status": "success",
  "transactionId": "txn_abc"
}
```

---

### 4.5 Track Order

```
GET /v1/orders/{orderId}/status
```

**Response:**
```json
{
  "orderId": "order_789",
  "status": "shipped",
  "trackingNumber": "TRACK123",
  "estimatedDelivery": "2026-01-25"
}
```

---

## 5. High-Level Design (HLD)
<img width="655" height="315" alt="image" src="https://github.com/user-attachments/assets/08669c2b-b484-4493-8357-f9a4fa4ff266" />


### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 10M Monthly Active Users                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                │                 │
                │ - Routing       │
                │ - Auth/AuthZ    │
                │ - Rate Limiting │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┬───────────┐
        │                │                │           │
        ▼                ▼                ▼           ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Search  │    │ Product  │ │   Cart   │
  │ Service  │    │ Service  │    │ Service  │ │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘ └────┬─────┘
       │               │               │            │
       ▼               ▼               ▼            ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │Elastic   │    │ Product  │ │   Cart   │
  │   DB     │    │ Search   │    │   DB     │ │   DB     │
  │ (MySQL)  │    │          │    │(MongoDB) │ │(Postgres)│
  └──────────┘    └──────────┘    └──────────┘ └──────────┘


        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │Checkout  │    │ Payment  │    │  Order   │
        │ Service  │───>│ Service  │    │ Status   │
        └────┬─────┘    └────┬─────┘    │ Service  │
             │               │           └────┬─────┘
             │               ▼                │
             │          ┌──────────┐          │
             │          │ Payment  │          │
             │          │ Gateway  │          │
             │          └──────────┘          │
             │                                │
             └────────────┬───────────────────┘
                          ▼
                    ┌──────────┐
                    │  Order   │
                    │   DB     │
                    │ (MySQL)  │
                    └──────────┘
```

### 5.2 Service Responsibilities

**1. User Service:**
- Login/Signup
- User profile management
- JWT token generation

**2. Search Service:**
- Query Elasticsearch
- Return paginated results

**3. Product Service:**
- Manage product catalog
- Update product metadata

**4. Cart Service:**
- Add/Update/Remove items
- Calculate cart total

**5. Checkout Service:**
- Validate inventory
- Initiate payment

**6. Payment Service:**
- Process payments via gateway
- Handle success/failure

**7. Order Status Service:**
- Track order history
- Provide status updates

---

## 6. Low-Level Design (LLD)
<img width="736" height="390" alt="image" src="https://github.com/user-attachments/assets/2c99c8d7-c0a8-4193-9934-d85a4fcabe5c" />


### 6.1 User Service

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    addresses JSON,  -- Array of delivery addresses
    created_at TIMESTAMP
);
```

**Flow:**
1. User signs up → Store in MySQL
2. User logs in → Validate credentials
3. Return JWT token (used in header for all API calls)
4. API Gateway verifies JWT for authentication

---

### 6.2 Search Service + Product Service

#### **Problem: Slow Search in Database**

❌ **Naive Approach:**
- Store products in MongoDB
- Search directly in MongoDB
- **Problem:** Slow for millions of products!

✅ **Optimized Approach: Elasticsearch**

**Architecture:**

```
Product DB (MongoDB)
    │
    │ CDC Pipeline
    ▼
Elasticsearch
    │
    │ Search Queries
    ▼
Search Service
```

**Why Elasticsearch?**
- **Fast:** Sub-second search on millions of records
- **Full-text search:** "iPhon" finds "iPhone"
- **Filters:** By price, rating, category
- **Relevance ranking:** Best matches first

**Product Database (MongoDB):**

**Why MongoDB?**
- **Flexible schema:** Different products have different attributes
- **Document storage:** JSON format for product details
- **Scalable:** Handles millions of products

**Schema:**
```json
{
  "productId": "prod_123",
  "title": "iPhone 15 Pro",
  "description": "Latest iPhone...",
  "category": "Electronics",
  "quantity": 10,  // ⚠️ Will optimize later!
  "price": 999.99,
  "currency": "USD",
  "images": ["blob_url_1", "blob_url_2"],  // S3 URLs
  "specifications": {
    "brand": "Apple",
    "model": "A17 Pro",
    "ram": "8GB"
  }
}
```

**Image Storage:**

❌ **Don't store images in MongoDB** (too large!)

✅ **Use S3 + CDN:**

```
Images → S3 Bucket → CDN (CloudFront)
                      │
                      ▼
              Frontend fetches from CDN
```

**Benefits:**
- **Fast:** CDN serves from edge locations
- **Scalable:** S3 handles millions of images
- **Cost-effective:** CDN caching reduces S3 calls

**CDC Pipeline (Change Data Capture):**

```
MongoDB
    │
    │ Detect changes
    ▼
Debezium
    │
    │ Publish events
    ▼
Kafka
    │
    │ Consumer
    ▼
Elasticsearch
```

**When to sync?**
- Product added/updated
- Quantity changes (important for availability!)

---

### 6.3 Cart Service

**Database:** PostgreSQL

**Why PostgreSQL?**
- Need consistency (cart must be accurate)
- Relational data (user ↔ products)

**Schema:**
```sql
CREATE TABLE cart (
    cart_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    products JSONB,  -- [{productId, quantity}, ...]
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Important:** Don't store price in cart!

**Why?**
- Price changes over time
- User adds item today for $100
- Comes back tomorrow, price is $90
- Cart should show updated price

**Flow:**
1. User adds item to cart
2. Store: `{cart_id, user_id, products: [{productId, qty}]}`
3. When user views cart:
   - Fetch product prices from Product DB
   - Calculate total dynamically

---

### 6.4 Checkout Service + Inventory Management

This is the **MOST CRITICAL** part!

#### **Challenge: Inventory Consistency**

**Problem:**
- User added 3 iPhones to cart yesterday
- Today, only 1 iPhone left in stock
- User tries to checkout → Should fail!

**Solution:** Inventory Service

**Architecture:**

```
Checkout Service
    │
    │ Check stock
    ▼
Inventory Service
    │
    ▼
Inventory DB (PostgreSQL)
```

**Inventory Database:**

**Why PostgreSQL?**
- **ACID transactions** (critical!)
- **Consistency** (source of truth for stock)
- **Indexed** on product_id (fast lookups)

**Schema:**
```sql
CREATE TABLE inventory (
    product_id UUID PRIMARY KEY,
    available_qty INT NOT NULL,
    reserved_qty INT DEFAULT 0,
    updated_at TIMESTAMP
);

CREATE INDEX idx_product_id ON inventory(product_id);
```

⚠️ **This DB is the BACKBONE of the system!**

**Checkout Flow (Before Event-Driven Architecture):**

```
1. User clicks "Checkout"
2. Checkout Service → Inventory Service: Check stock
3. If stock available → Payment Service
4. Payment successful → Update:
   - Payment DB
   - Inventory DB (reduce qty)
   - Order DB (create order)
```

**Problem with this approach:**
- 3 API calls after payment
- What if one fails?
- **Not atomic!** (consistency issue)

#### **Optimized Solution: Event-Driven Architecture with Kafka**

```
Payment Service (After payment)
    │
    │ Publish event
    ▼
Kafka Broker
    │
    ├─> Order Consumer → Order DB
    │
    ├─> Inventory Consumer → Inventory DB
    │
    └─> Notification Consumer → Send email/SMS
```

**Flow:**

1. **Payment successful**
2. **Payment Service:**
   - Update Payment DB
   - Publish event to Kafka: `{orderId, userId, products, status: "success"}`

3. **Kafka Consumers:**
   - **Order Consumer:** Create order in Order DB
   - **Inventory Consumer:** Reduce inventory quantity
   - **Notification Consumer:** Send confirmation email

**Benefits:**
- ✅ **Decoupled:** Services don't directly call each other
- ✅ **Reliable:** Kafka guarantees message delivery
- ✅ **Asynchronous:** No blocking calls
- ✅ **Retryable:** If consumer fails, retry from Kafka

**Order Database Schema:**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    products JSONB,  -- [{productId, qty, price}]
    total_amount DECIMAL(10, 2),
    currency VARCHAR(3),
    payment_id UUID,
    status VARCHAR(50),  -- pending, confirmed, shipped, delivered
    created_at TIMESTAMP
);
```

---

### 6.5 Handling Limited Stock (Race Condition)

#### **Problem: Flash Sale Scenario**

```
Sale: 1 iPhone, 1000 users trying to buy!

Checkout Service (10 instances)
    │
    ├─ Instance 1: Checks inventory → 1 available → Proceed
    ├─ Instance 2: Checks inventory → 1 available → Proceed
    ├─ Instance 3: Checks inventory → 1 available → Proceed
    └─ ... (10 users proceed to payment!)
```

**Result:** 10 orders for 1 iPhone! ❌

#### **Solution: Redis Distributed Lock**

```
Checkout Flow:
    │
    ▼
Check Inventory Service → 1 iPhone available
    │
    ▼
Try to acquire Redis lock:
    SET product:prod_123:lock user_1 EX 60 NX
    │
    ├─ Success → Proceed to payment
    │
    └─ Failure → "Out of stock" error
```

**Redis Lock:**
```redis
Key: product:{productId}:lock
Value: {userId}
TTL: 60 seconds
```

**Flow:**

1. User 1 tries to buy iPhone
2. Check inventory: 1 available ✅
3. Try Redis lock: `SET product:prod_123:lock user_1 EX 60 NX`
4. Success! → Proceed to payment
5. User 2 tries to buy (0.001 seconds later)
6. Check inventory: 1 available ✅
7. Try Redis lock: **FAILS** (already locked!)
8. Return error: "This item is being purchased by another user"

**After Payment:**
- Payment success → Delete Redis lock
- Payment failure/timeout → Lock expires after 60s (TTL)

---

### 6.6 Optimizing Inventory Service (Caching)

**Problem:**
- 10 orders/second
- Each checks Inventory DB
- **High load on database!**

**Solution:** Redis Cache in front of database

```
Inventory Service
    │
    ▼
Redis Cache (Check first)
    │
    ├─ Cache Hit → Return quantity
    │
    └─ Cache Miss → Query DB → Update cache
```

**New Problem: Cache Consistency**

```
Scenario:
1. Inventory DB: iPhone qty = 1
2. Redis Cache: iPhone qty = 1
3. User buys iPhone
4. Inventory Consumer updates DB: qty = 0
5. Redis still shows: qty = 1 ❌
```

**Solution 1: Write-Through Cache**
- Inventory Consumer updates **both** DB and Redis

**Solution 2: CDC Pipeline**
- DB update → CDC → Kafka → Update Redis
- **Near real-time** sync

**Recommendation:** Write-Through (simpler, faster)

---

### 6.7 Updating Product Quantity

**Problem:**
- Product DB has `quantity` field
- But it's not updated when orders placed!
- Users see wrong availability

**Solution:** CDC Pipeline from Inventory DB to Product DB

```
Inventory DB (qty changes)
    │
    │ CDC
    ▼
Kafka
    │
    ▼
Product DB Consumer
    │
    ▼
Product DB (update quantity)
    │
    │ CDC
    ▼
Elasticsearch (sync)
```

---

## 7. Complete Low-Level Architecture

```
Users
  │
  ▼
API Gateway (Auth, Rate Limiting)
  │
  ├─> User Service → MySQL
  │
  ├─> Search Service → Elasticsearch ← CDC ← Product DB
  │
  ├─> Product Service → MongoDB → S3/CDN (images)
  │
  ├─> Cart Service → PostgreSQL
  │
  └─> Checkout Service
        │
        ├─> Inventory Service → Redis Cache → PostgreSQL
        │                                         │
        │                                         │
        └─> Redis Lock (for limited stock)       │
                │                                 │
                ▼                                 │
           Payment Service                        │
                │                                 │
                ▼                                 │
           Payment Gateway                        │
                │                                 │
                ▼                                 │
           Kafka Broker                           │
                │                                 │
                ├─> Order Consumer → Order DB ────┘
                │
                ├─> Inventory Consumer → Inventory DB
                │
                └─> Notification Consumer → Email/SMS
```

---

## 8. Interview-Style Q&A

### Q1: Why Elasticsearch instead of MongoDB for search?

**A:** MongoDB is **write-optimized**, not **read-optimized**.

- Searching millions of products in MongoDB = slow full-table scan
- Elasticsearch uses **inverted index** (optimized for search)
- Elasticsearch: < 200ms, MongoDB: > 2 seconds

### Q2: Why separate Product DB and Inventory DB?

**A:** Different consistency requirements!

**Product DB:**
- Metadata (title, description, images)
- Changes rarely
- Eventual consistency okay
- MongoDB (flexible schema)

**Inventory DB:**
- Stock quantities
- Changes frequently (every order!)
- **Strict consistency required**
- PostgreSQL (ACID transactions)

### Q3: What if Kafka goes down?

**A:** 
- **During outage:** Payment succeeds, but orders not created ❌
- **Solution:** Kafka cluster with replication (3-5 brokers)
- **Fallback:** Payment Service writes to DB, background job processes

### Q4: How to handle payment failures?

**A:**
1. Payment fails → Don't publish to Kafka
2. Release Redis lock (TTL expires)
3. Inventory remains unchanged
4. User gets error message

### Q5: Why use JWT tokens?

**A:**
- **Stateless:** No session storage needed
- **Scalable:** Works across multiple servers
- **Secure:** Contains user_id, encrypted
- **Fast:** No database lookup for auth

### Q6: How to scale the system?

**A:**

**Horizontal Scaling:**
- Checkout Service: 10 instances (handles 10 orders/sec)
- Search Service: 20 instances (80% of traffic)
- All stateless (easy to scale)

**Database Scaling:**
- Read replicas for Product DB
- Sharding for Inventory DB (by product_id)
- Redis Cluster for distributed cache

### Q7: What are the database choices summary?

**A:**

| Service | Database | Why? |
|---------|----------|------|
| User | MySQL | Consistent, relational |
| Product | MongoDB | Flexible schema, document storage |
| Cart | PostgreSQL | Consistent, relational |
| Inventory | PostgreSQL | **ACID critical!** |
| Order | MySQL | Consistent, transactional |
| Payment | MySQL | Audit trail, consistency |

### Q8: Security considerations?

**A:**
- JWT tokens in headers (not body)
- HTTPS for all communication
- PCI-DSS compliance for payments
- Rate limiting (prevent bot attacks)
- Input validation (prevent SQL injection)

---

## 9. Key Takeaways

✅ **Elasticsearch is essential** for fast product search  
✅ **Kafka** for event-driven, decoupled architecture  
✅ **Redis locks** prevent race conditions in flash sales  
✅ **Separate DBs** for different consistency needs  
✅ **CDC pipelines** keep data in sync across systems  
✅ **Caching** reduces database load (but watch consistency!)  
✅ **S3 + CDN** for efficient image delivery  

---

## Summary

**Architecture Highlights:**
- 7 microservices (User, Search, Product, Cart, Checkout, Payment, Order Status)
- 6 databases (MySQL x3, PostgreSQL x2, MongoDB x1)
- Elasticsearch for search
- Kafka for event-driven architecture
- Redis for caching + distributed locks
- S3 + CDN for image storage

**Performance:**
- Search: < 200ms
- Product details: < 300ms
- Checkout: < 2 seconds

**Scale:**
- 10M monthly active users
- 10 orders/second
- Handles flash sales with Redis locks

**End of Lecture 3**
