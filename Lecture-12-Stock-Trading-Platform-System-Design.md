# Lecture 12: Stock Trading Platform System Design (Zerodha/Groww)

> **Lecture:** 12  
> **Topic:** System Design  
> **Application:** Stock Trading Platform (Zerodha, Groww, Upstox)  
> **Scale:** Millions of Users, High-Frequency Trading  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-11-Ride-Booking-Application-System-Design|Lecture 11: Ride Booking Application]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Architecture Diagram](#7-complete-architecture-diagram)
8. [Complete Order Flow](#8-complete-order-flow)
9. [Interview Q&A](#9-interview-qa)
10. [Kafka Topics Summary](#10-kafka-topics-summary)
11. [Key Takeaways](#11-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Stock Trading Platform?

A **stock trading platform** (broker) enables users to buy and sell stocks on exchanges (NSE/BSE) with real-time price tracking, order placement, and portfolio management.

**Important Distinction:**
- **Stock Broker (This Video):** Zerodha, Groww, Upstox - Interface for users
- **Stock Exchange (Out of Scope):** NSE, BSE - Where actual trading happens

**Examples:** Zerodha, Groww, Upstox, Robinhood, E*Trade

### Problem It Solves

1. **Real-Time Price Tracking:** See stock prices in < 50ms
2. **Order Placement:** Buy/sell stocks via exchange
3. **Portfolio Management:** Track P&L, holdings
4. **Watchlist:** Save favorite stocks
5. **Historical Data:** View past price trends

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **User Onboarding**
   - Register, login, KYC verification

2. **Real-Time Stock Prices**
   - View current stock prices (< 50ms latency)
   - View historical price data (charts)

3. **Order Placement**
   - Buy/sell stocks (market order, limit order)
   - Cancel pending orders

4. **Watchlist**
   - Create custom watchlists
   - Add/remove stocks

5. **Portfolio & P&L**
   - View holdings, positions
   - Track profit/loss

### 2.2 Non-Functional Requirements

1. **Scale**
   - **Millions of users**
   - **Millions of transactions/day**
   - **8,000-10,000 stocks** (NSE + BSE)

2. **CAP Theorem**
   - **Highly Consistent (CP):** No inconsistency in money/stocks
   
   **Why?**
   - ✅ Payment deducted but stock not booked = Disaster
   - ✅ Downtime acceptable, inconsistency is NOT

3. **Low Latency**
   - **Real-time price:** < 50ms
   - **Order placement:** < 100ms

4. **Regional Deployment**
   - All servers in same geographical region (India for Indian stocks)
   - Reduces latency significantly

---
<img width="1314" height="252" alt="image" src="https://github.com/user-attachments/assets/d1089758-220e-4200-afe5-900cbc3ebd49" />


## 3. Core Entities

1. **User** - Trader
2. **Stock** - Symbol (RELIANCE, TCS, INFY)
3. **Order** - Buy/sell request
4. **Trade** - Confirmed transaction
5. **Portfolio** - User's holdings
6. **Watchlist** - Saved stocks

---

## 4. API Design
<img width="859" height="292" alt="image" src="https://github.com/user-attachments/assets/e16d8870-9f26-4750-8536-00983a4c883f" />

<img width="797" height="211" alt="image" src="https://github.com/user-attachments/assets/ac080180-4a0e-449b-b20e-b51cc78099e5" />


### 4.1 User Onboarding APIs

```
POST /v1/users/signup
POST /v1/users/login
POST /v1/users/kyc
```

---

### 4.2 Market Data APIs (WebSocket!)

#### **Get All Stocks (Paginated)**

```
GET /v1/stocks?page=1&limit=100
```

**Response:**
```json
{
  "stocks": [
    {"symbol": "RELIANCE", "name": "Reliance Industries"},
    {"symbol": "TCS", "name": "Tata Consultancy Services"}
  ],
  "totalPages": 80
}
```

---

#### **Real-Time Stock Price (WebSocket!)**

```
WS /v1/stocks/live
```

⚠️ **Important:** This is a **WebSocket** connection, not HTTP!

**Subscribe Message:**
```json
{
  "action": "subscribe",
  "symbols": ["RELIANCE", "TCS", "INFY"]
}
```

**Price Update (Server → Client):**
```json
{
  "symbol": "RELIANCE",
  "ltp": 2450.75,
  "change": +15.25,
  "changePercent": +0.62,
  "timestamp": "2026-01-21T10:30:00.123Z"
}
```

---

#### **Historical Price Data**

```
GET /v1/stocks/{symbol}/history?from={date}&to={date}
```

**Response:**
```json
{
  "symbol": "RELIANCE",
  "data": [
    {"timestamp": "2026-01-20T09:15:00Z", "open": 2435, "high": 2455, "low": 2430, "close": 2445},
    {"timestamp": "2026-01-20T09:16:00Z", "open": 2445, "high": 2450, "low": 2440, "close": 2448}
  ]
}
```

---

### 4.3 Fund Management APIs

```
POST /v1/funds/deposit
POST /v1/funds/withdraw
GET /v1/funds/history
GET /v1/funds/balance
```

---

### 4.4 Trading APIs

#### **Place Order**

```
POST /v1/orders
```

**Request Body:**
```json
{
  "symbol": "RELIANCE",
  "orderType": "MARKET",  // MARKET, LIMIT
  "side": "BUY",  // BUY, SELL
  "quantity": 10,
  "price": 2450.00  // Only for LIMIT orders
}
```

**Response:**
```json
{
  "orderId": "ord_123",
  "status": "PENDING",
  "message": "Order placed successfully"
}
```

---

#### **Get Orders**

```
GET /v1/orders
GET /v1/orders/{orderId}
```

---

#### **Cancel Order**

```
DELETE /v1/orders/{orderId}
```

---

### 4.5 Portfolio APIs

```
GET /v1/portfolio
GET /v1/portfolio/holdings
GET /v1/portfolio/positions
GET /v1/portfolio/performance
```

---

### 4.6 Watchlist APIs

```
POST /v1/watchlists
GET /v1/watchlists
GET /v1/watchlists/{watchlistId}
PUT /v1/watchlists/{watchlistId}
DELETE /v1/watchlists/{watchlistId}
```

---

## 5. High-Level Design (HLD)
<img width="714" height="343" alt="image" src="https://github.com/user-attachments/assets/d39c25ce-1009-477c-9dd4-6c1449b34cea" />


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
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       │               │
       ▼               ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    SERVICES                              │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │   User   │  │  Price   │  │  Order   │  │Portfolio │ │
  │  │ Service  │  │ Tracker  │  │ Service  │  │ Service  │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │Watchlist │  │ Payment  │  │Validator │  │  Order   │ │
  │  │ Service  │  │ Service  │  │ Service  │  │ Tracker  │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  └─────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │                EXCHANGE GATEWAY                          │
  └─────────────────────────────┬───────────────────────────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────┐
  │                EXCHANGE (NSE/BSE)                        │
  │                   [BLACK BOX]                            │
  └─────────────────────────────────────────────────────────┘
```

### 5.2 Service Responsibilities

**User Service:**
- Registration, login, KYC

**Price Tracker Service:**
- Real-time stock prices via WebSocket
- Historical data from InfluxDB

**Order Service:**
- Place, cancel orders
- Send to exchange via gateway

**Validator Service:**
- Validate KYC, funds, market hours

**Order Tracker Service:**
- Track order status from exchange
- Update status in DB

**Portfolio Service:**
- Calculate P&L
- Show holdings, positions

**Watchlist Service:**
- CRUD watchlists

**Payment Service:**
- Deposit/withdraw funds

**Exchange Gateway:**
- Single point of contact with exchange
- WebSocket connection to NSE/BSE

---

## 6. Low-Level Design (LLD)

### 6.1 User Service (Simple)

**User DB (PostgreSQL):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20) UNIQUE,
    name VARCHAR(255),
    is_kyc_verified BOOLEAN DEFAULT FALSE,
    kyc_provider_id VARCHAR(100),  -- Third-party KYC reference
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**KYC Flow:**
```
User submits documents
    ↓
Third-party KYC service (DigiLocker, etc.)
    ↓
Returns verification status
    ↓
Update is_kyc_verified = TRUE
```

⚠️ **Security:** Sensitive data (PAN, Aadhaar) stored encrypted or at KYC provider

---

### 6.2 Payment Service (Simple)

**Payment DB (PostgreSQL):**

```sql
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    amount DECIMAL(15,2),
    type VARCHAR(20),  -- DEPOSIT, WITHDRAW
    status VARCHAR(20),  -- PENDING, SUCCESS, FAILED
    payment_gateway_ref VARCHAR(100),
    currency VARCHAR(10) DEFAULT 'INR',
    created_at TIMESTAMP
);

CREATE TABLE wallets (
    user_id UUID PRIMARY KEY REFERENCES users(user_id),
    balance DECIMAL(15,2) DEFAULT 0.00,
    updated_at TIMESTAMP
);
```

---

### 6.3 Price Tracker Service (Core!)

#### **Why WebSocket? (Not SSE or HTTP Polling)**

**HTTP Polling:**
```
8,000 stocks × 1 request/second = 8,000 req/s ❌
```

**SSE (Server-Sent Events):**
```
Open connection → Server pushes data
    ↓
Problem: To add new stock subscription,
         need to CLOSE and REOPEN connection ❌
```

**WebSocket:**
```
Bidirectional connection
    ↓
Client: "Subscribe to RELIANCE"
Server: Pushes RELIANCE price updates
    ↓
Client: "Add TCS subscription"
Server: Pushes TCS price updates too
    ↓
No need to close connection! ✅
```

---

#### **Price Tracker Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│              EXCHANGE (NSE/BSE)                          │
└─────────────────────────────┬───────────────────────────┘
                              │ WebSocket
                              ▼
                    ┌─────────────────┐
                    │ Exchange Gateway│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │     Kafka       │
                    │  (stock_price)  │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │  Price   │      │  Redis   │      │ InfluxDB │
    │ Ingestor │ ──▶  │  PubSub  │      │(History) │
    │ Service  │      └────┬─────┘      └──────────┘
    └──────────┘           │
                           │ Publish
                           ▼
                    ┌──────────┐
                    │  Price   │
                    │ Tracker  │
                    │ Service  │
                    └────┬─────┘
                         │ WebSocket
                         ▼
                    ┌──────────┐
                    │  USERS   │
                    └──────────┘
```

---

#### **Component Responsibilities:**

**Exchange Gateway:**
- Single WebSocket connection to exchange
- 1 connection supports ~300-500 stocks
- For 8,000 stocks → ~20 WebSocket connections

**Kafka (stock_price topic):**
- Buffer for high-volume price updates
- Decouple exchange from internal services

**Price Ingestor Service:**
- Consumes from Kafka
- Writes to InfluxDB (historical)
- Publishes to Redis PubSub (real-time)

**Redis PubSub:**
- Ultra-low latency (< 10ms)
- Broadcast to all subscribers

**InfluxDB (Time-Series DB):**
- Store historical OHLC data
- Query: "Give me RELIANCE prices from 9:15 to 12:00"

**Price Tracker Service (WebSocket Server):**
- Subscribes to Redis PubSub
- Pushes updates to clients via WebSocket

---

#### **Data Flow (Real-Time Price):**

```
Step 1: Exchange sends price update
    {symbol: "RELIANCE", ltp: 2450.75}
    ↓
Step 2: Exchange Gateway receives via WebSocket
    ↓
Step 3: Publishes to Kafka (stock_price topic)
    ↓
Step 4: Price Ingestor Service consumes
    ↓
Step 5: Writes to InfluxDB (historical storage)
    ↓
Step 6: Publishes to Redis PubSub (LTP channel)
    PUBLISH stock:RELIANCE {ltp: 2450.75}
    ↓
Step 7: Price Tracker Service receives (subscriber)
    ↓
Step 8: Pushes to User via WebSocket (< 50ms total)
```

---

#### **Data Flow (Historical Data):**

```
User opens chart for RELIANCE
    ↓
GET /v1/stocks/RELIANCE/history?from=9:15&to=12:00
    ↓
Price Tracker Service queries InfluxDB
    ↓
SELECT * FROM stock_prices
WHERE symbol = 'RELIANCE'
AND time BETWEEN '9:15' AND '12:00'
    ↓
Return OHLC data for chart
```

---

#### **InfluxDB Schema:**

```sql
-- Time-series database (not SQL, but similar)
-- Measurement: stock_prices

stock_prices {
    symbol: TAG,      -- RELIANCE, TCS
    ltp: FIELD,       -- 2450.75
    open: FIELD,      -- 2435.00
    high: FIELD,      -- 2455.00
    low: FIELD,       -- 2430.00
    volume: FIELD,    -- 1000000
    timestamp: TIME   -- 2026-01-21T10:30:00.123Z
}

-- Query Example
SELECT ltp, open, high, low, close
FROM stock_prices
WHERE symbol = 'RELIANCE'
AND time > now() - 1h
```

---

#### **Why Subscribe to ALL Stocks at Market Open?**

**Problem:**
```
User A wants RELIANCE at 10:00
But we never subscribed to RELIANCE
    ↓
No historical data from 9:15 to 10:00 ❌
```

**Solution:**
```
Market opens at 9:15
    ↓
Exchange Gateway subscribes to ALL 8,000 stocks
    ↓
All prices flow to InfluxDB
    ↓
When User A opens RELIANCE at 10:00:
    - Historical: InfluxDB (9:15 → 10:00)
    - Real-time: Redis PubSub (10:00+)
```

---

### 6.4 Order Service (Order Placement)

#### **Order Flow Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
│               POST /v1/orders                            │
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
                │   (raw_orders)  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Validator     │
                │    Service      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Kafka    │    │ Kafka    │    │ Kafka    │
  │(verified)│    │(rejected)│    │(verified)│
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │  Order   │    │  Order   │
  │ Service  │    │ Service  │
  │(Execute) │    │ (Notify) │
  └────┬─────┘    └──────────┘
       │
       ▼
  ┌──────────┐
  │ Exchange │
  │ Gateway  │
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │ EXCHANGE │
  └──────────┘
```

---

#### **Validation Checks:**

```python
def validate_order(order):
    # 1. KYC Verification
    if not is_kyc_verified(order.user_id):
        return REJECTED, "KYC not verified"
    
    # 2. Account Status
    if not is_account_active(order.user_id):
        return REJECTED, "Account inactive"
    
    # 3. Fund Validation
    required_amount = order.quantity * order.price
    if get_balance(order.user_id) < required_amount:
        return REJECTED, "Insufficient funds"
    
    # 4. Market Hours
    if not is_market_open():
        return REJECTED, "Market closed"
    
    # 5. Stock Validation
    if not is_valid_stock(order.symbol):
        return REJECTED, "Invalid stock symbol"
    
    return VERIFIED, "Order validated"
```

---

#### **Order DB (PostgreSQL):**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    symbol VARCHAR(20),
    order_type VARCHAR(20),  -- MARKET, LIMIT
    side VARCHAR(10),  -- BUY, SELL
    quantity INT,
    price DECIMAL(15,2),
    status VARCHAR(20),  -- PENDING, EXECUTED, REJECTED, CANCELLED
    trade_id VARCHAR(100),  -- From exchange (null until confirmed)
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_symbol ON orders(symbol);
```

---

#### **Order Types:**

**Market Order:**
```
Execute immediately at current market price
    ↓
Exchange confirms quickly (seconds)
```

**Limit Order:**
```
Execute only at specified price
    ↓
May take minutes/hours if price not reached
    ↓
Need callback mechanism (Order Tracker)
```

---

### 6.5 Order Tracker Service (Callbacks)

#### **Problem:**

```
User places LIMIT order at 2,500
Current price: 2,450
    ↓
Order goes to exchange
    ↓
Price reaches 2,500 after 2 hours
    ↓
Exchange executes order
    ↓
How does our system know? 🤔
```

---

#### **Solution: WebSocket Callback from Exchange**

```
┌──────────────────────────────────────────────────────────┐
│                    EXCHANGE                               │
│              (Order Status Updates)                       │
└───────────────────────┬──────────────────────────────────┘
                        │ WebSocket
                        ▼
              ┌─────────────────┐
              │ Exchange Gateway│
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │     Kafka       │
              │ (order_status)  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Order Tracker  │
              │    Service      │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Order DB │  │ Trade DB │  │ Notifi-  │
  │ (Update) │  │ (Insert) │  │ cation   │
  └──────────┘  └──────────┘  └──────────┘
```

---

#### **Order Tracker Flow:**

```python
def process_order_status(status_update):
    order_id = status_update['order_id']
    new_status = status_update['status']
    trade_id = status_update.get('trade_id')
    
    # 1. Update Order DB
    db.execute("""
        UPDATE orders
        SET status = ?,
            trade_id = ?,
            updated_at = NOW()
        WHERE order_id = ?
    """, new_status, trade_id, order_id)
    
    # 2. Insert into Trade DB (if executed)
    if new_status == 'EXECUTED':
        order = get_order(order_id)
        db.execute("""
            INSERT INTO trades
            (trade_id, order_id, user_id, symbol, side, quantity, price, executed_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, NOW())
        """, trade_id, order_id, order.user_id, order.symbol, 
             order.side, order.quantity, order.price)
    
    # 3. Send Notification
    send_push_notification(order.user_id, f"Order {new_status}")
```

---

#### **Trade DB (PostgreSQL):**

```sql
CREATE TABLE trades (
    trade_id VARCHAR(100) PRIMARY KEY,  -- From exchange
    order_id UUID REFERENCES orders(order_id),
    user_id UUID REFERENCES users(user_id),
    symbol VARCHAR(20),
    side VARCHAR(10),  -- BUY, SELL
    quantity INT,
    price DECIMAL(15,2),
    executed_at TIMESTAMP
);

CREATE INDEX idx_user_id ON trades(user_id);
CREATE INDEX idx_symbol ON trades(symbol);
```

---

#### **Why Separate Order DB and Trade DB?**

| Order DB | Trade DB |
|----------|----------|
| All orders (pending, rejected, executed) | Only executed trades |
| Includes rejected by validator | Only confirmed by exchange |
| For user order history | For P&L calculation |
| | For reconciliation with exchange |

---

### 6.6 Portfolio Service (P&L Calculation)

#### **Flow:**

```
User opens Portfolio page
    ↓
Portfolio Service queries Trade DB
    - Get all executed trades for user
    ↓
Calculate holding quantity per stock
    ↓
Portfolio Service calls Price Tracker
    - Get current price for each stock
    ↓
Calculate P&L:
    P&L = (Current Price - Avg Buy Price) × Quantity
    ↓
Return holdings with P&L
```

---

#### **P&L Calculation:**

```python
def calculate_portfolio(user_id):
    # 1. Get all trades
    trades = db.query("""
        SELECT symbol, side, quantity, price
        FROM trades
        WHERE user_id = ?
        ORDER BY executed_at
    """, user_id)
    
    # 2. Calculate holdings (quantity per stock)
    holdings = {}
    for trade in trades:
        if trade.symbol not in holdings:
            holdings[trade.symbol] = {'quantity': 0, 'avg_price': 0}
        
        if trade.side == 'BUY':
            # Update average price
            total_qty = holdings[trade.symbol]['quantity'] + trade.quantity
            total_value = (holdings[trade.symbol]['quantity'] * holdings[trade.symbol]['avg_price']) + (trade.quantity * trade.price)
            holdings[trade.symbol]['avg_price'] = total_value / total_qty
            holdings[trade.symbol]['quantity'] = total_qty
        else:  # SELL
            holdings[trade.symbol]['quantity'] -= trade.quantity
    
    # 3. Get current prices
    current_prices = price_tracker.get_prices(holdings.keys())
    
    # 4. Calculate P&L
    portfolio = []
    for symbol, holding in holdings.items():
        if holding['quantity'] > 0:
            current_price = current_prices[symbol]
            pnl = (current_price - holding['avg_price']) * holding['quantity']
            portfolio.append({
                'symbol': symbol,
                'quantity': holding['quantity'],
                'avgPrice': holding['avg_price'],
                'currentPrice': current_price,
                'pnl': pnl,
                'pnlPercent': (pnl / (holding['avg_price'] * holding['quantity'])) * 100
            })
    
    return portfolio
```

---

### 6.7 Watchlist Service (Simple)

#### **Watchlist DB (PostgreSQL):**

```sql
CREATE TABLE watchlists (
    watchlist_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    name VARCHAR(100),  -- "Finance Stocks", "IT Stocks"
    created_at TIMESTAMP
);

CREATE TABLE watchlist_stocks (
    watchlist_id UUID REFERENCES watchlists(watchlist_id),
    symbol VARCHAR(20),
    added_at TIMESTAMP,
    PRIMARY KEY (watchlist_id, symbol)
);

CREATE INDEX idx_user_id ON watchlists(user_id);
```

---

#### **Watchlist with Real-Time Prices:**

```
User opens watchlist
    ↓
Watchlist Service gets stocks
    SELECT symbol FROM watchlist_stocks
    WHERE watchlist_id = ?
    ↓
For each stock:
    Subscribe via WebSocket to Price Tracker
    ↓
User sees real-time prices for all watchlist stocks
```

---

## 7. Complete Architecture Diagram
<img width="934" height="536" alt="image" src="https://github.com/user-attachments/assets/eae02ab3-6870-43ad-9361-f34fe6403efe" />


```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   API    │    │WebSocket │    │   KYC    │
  │ Gateway  │    │ Gateway  │    │ Provider │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       │               │
       ▼               ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    SERVICES                              │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │   User   │  │  Price   │  │  Order   │  │Portfolio │ │
  │  │ Service  │  │ Tracker  │  │ Service  │  │ Service  │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │Watchlist │  │ Payment  │  │Validator │  │  Order   │ │
  │  │ Service  │  │ Service  │  │ Service  │  │ Tracker  │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  │                                                          │
  │  ┌──────────┐                                            │
  │  │  Price   │                                            │
  │  │ Ingestor │                                            │
  │  └──────────┘                                            │
  └─────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    DATA STORES                           │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │  Kafka   │  │  Redis   │  │ InfluxDB │  │PostgreSQL│ │
  │  │ (Events) │  │ (PubSub) │  │ (Prices) │  │  (Data)  │ │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
  └─────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │                EXCHANGE GATEWAY                          │
  │           (WebSocket to NSE/BSE)                         │
  └─────────────────────────────┬───────────────────────────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────┐
  │                EXCHANGE (NSE/BSE)                        │
  │                   [BLACK BOX]                            │
  └─────────────────────────────────────────────────────────┘
```

---

## 8. Complete Order Flow

### 8.1 Place Order

```
Step 1: User clicks "Buy 10 RELIANCE @ Market"
    POST /v1/orders {symbol: "RELIANCE", side: "BUY", quantity: 10, orderType: "MARKET"}
    ↓
Step 2: API Gateway → Kafka (raw_orders topic)
    ↓
Step 3: Validator Service consumes
    - Check KYC: ✅
    - Check balance: ✅ (2,500 × 10 = 25,000 available)
    - Check market hours: ✅
    ↓
Step 4: Publish to Kafka (verified_orders topic)
    ↓
Step 5: Order Service consumes
    - Insert into Order DB (status: PENDING, trade_id: NULL)
    - Send to Exchange Gateway
    ↓
Step 6: Exchange Gateway → Exchange (NSE/BSE)
    ↓
Step 7: Exchange confirms (trade_id: "TRD123")
    ↓
Step 8: Exchange Gateway → Kafka (order_status topic)
    ↓
Step 9: Order Tracker Service consumes
    - Update Order DB (status: EXECUTED, trade_id: "TRD123")
    - Insert into Trade DB
    - Send notification: "Order executed!"
```

---

### 8.2 View Real-Time Prices

```
Step 1: User opens app at 10:00 AM
    ↓
Step 2: WebSocket connection established
    ↓
Step 3: User subscribes to RELIANCE
    {"action": "subscribe", "symbols": ["RELIANCE"]}
    ↓
Step 4: Price Tracker receives subscription
    - Fetch history from InfluxDB (9:15 → 10:00)
    - Subscribe to Redis PubSub (stock:RELIANCE)
    ↓
Step 5: Send historical data to user (chart)
    ↓
Step 6: Push real-time updates as they arrive
    - Every price change → User sees in < 50ms
```

---

### 8.3 Calculate P&L

```
Step 1: User opens Portfolio
    GET /v1/portfolio
    ↓
Step 2: Portfolio Service queries Trade DB
    - User has: 10 RELIANCE @ avg 2,400
    ↓
Step 3: Get current price (via Price Tracker)
    - RELIANCE current: 2,500
    ↓
Step 4: Calculate P&L
    - (2,500 - 2,400) × 10 = 1,000 profit
    - Percent: (1,000 / 24,000) × 100 = 4.17%
    ↓
Step 5: Return portfolio with P&L
```

---

## 9. Interview Q&A

### Q1: Why WebSocket instead of SSE for price updates?

**A:**

**SSE Problem:**
```
User watching RELIANCE
    ↓
User wants to add TCS
    ↓
Must CLOSE SSE connection
    ↓
Open NEW SSE with [RELIANCE, TCS]
    ↓
Every new stock = new connection ❌
```

**WebSocket Solution:**
```
WebSocket open
    ↓
Send: {"subscribe": ["RELIANCE"]}
    ↓
Send: {"subscribe": ["TCS"]}  (same connection!)
    ↓
No reconnection needed ✅
```

### Q2: Why Kafka between Exchange Gateway and services?

**A:**

**Problem:**
```
Exchange sends 8,000 price updates/second
    ↓
Price Ingestor can process 5,000/second
    ↓
3,000 updates LOST ❌
```

**Kafka Solution:**
```
Exchange → Kafka (buffer) → Price Ingestor
    ↓
Kafka handles backpressure
    ↓
No data loss ✅
```

### Q3: Why InfluxDB for historical prices?

**A:**

**Requirements:**
- Store OHLC data every second
- Query: "RELIANCE prices from 9:15 to 12:00"
- Billions of data points

**Why InfluxDB?**
- ✅ Optimized for time-series queries
- ✅ Built-in aggregation (1-min, 5-min candles)
- ✅ Automatic data retention policies

### Q4: Why separate Order DB and Trade DB?

**A:**

| Aspect | Order DB | Trade DB |
|--------|----------|----------|
| Contains | All orders | Only executed trades |
| Status | PENDING, REJECTED, EXECUTED | Only EXECUTED |
| Use | Order history | P&L calculation |
| Reconciliation | No | Yes (with exchange) |

### Q5: How to handle exchange downtime?

**A:**

**Strategies:**

1. **Queue Orders:**
   - Store in Kafka
   - Retry when exchange back online

2. **Fallback Exchange:**
   - If NSE down, route to BSE

3. **Notify Users:**
   - "Exchange connectivity issue"

### Q6: Why validate before sending to exchange?

**A:**

**Reasons:**

1. **Cost:** Exchange API calls are expensive (₹ crores/month)
2. **Performance:** Exchange is slower than local validation
3. **Rate Limiting:** Exchange has request limits
4. **User Experience:** Reject invalid orders instantly

### Q7: How many WebSocket connections to exchange?

**A:**

```
Each connection: ~300-500 stocks
Total stocks: 8,000
Connections needed: 8,000 / 400 ≈ 20 connections
```

### Q8: How to scale Price Tracker for millions of users?

**A:**

**Strategies:**

1. **Redis PubSub:**
   - Single source of truth
   - Multiple Price Tracker instances subscribe

2. **Horizontal Scaling:**
   - Add more WebSocket servers
   - Load balancer distributes users

3. **Connection Pooling:**
   - Reuse connections

### Q9: Security considerations?

**A:**

1. **KYC:** Mandatory before trading
2. **HTTPS:** Encrypt all traffic
3. **JWT:** Authenticate WebSocket connections
4. **Rate Limiting:** Prevent abuse
5. **Encryption:** Sensitive data (PAN, Aadhaar)

### Q10: How is payment reconciliation done?

**A:**

**Daily Reconciliation:**
```
Market closes at 3:30 PM
    ↓
Exchange sends settlement file
    ↓
Compare with Trade DB
    ↓
Calculate net amount (buy - sell)
    ↓
Transfer funds via clearing house
```

---

## 10. Kafka Topics Summary

| Topic | Producer | Consumer | Purpose |
|-------|----------|----------|---------|
| `stock_price` | Exchange Gateway | Price Ingestor | Real-time prices |
| `order_status` | Exchange Gateway | Order Tracker | Trade confirmations |
| `raw_orders` | API Gateway | Validator | Incoming orders |
| `verified_orders` | Validator | Order Service | Validated orders |
| `rejected_orders` | Validator | Order Service | Rejected orders |

---

## 11. Key Takeaways

✅ **WebSocket (not SSE):** Bidirectional, add subscriptions without reconnect  
✅ **Exchange Gateway:** Single point of contact with exchange  
✅ **Kafka:** Buffer for high-volume price updates and orders  
✅ **InfluxDB:** Time-series DB for historical OHLC data  
✅ **Redis PubSub:** Ultra-low latency real-time broadcast  
✅ **Validator Service:** Filter orders before expensive exchange calls  
✅ **Order Tracker:** Handle async confirmations from exchange  
✅ **Trade DB:** Only confirmed trades (for reconciliation)  
✅ **Consistency > Availability:** Money/stock integrity critical  
✅ **Subscribe to ALL stocks:** Ensure historical data available  

---

## Summary

**Architecture Highlights:**
- 10+ microservices
- 5 databases (PostgreSQL, InfluxDB, Redis)
- Kafka for event streaming
- WebSocket for real-time (users + exchange)
- Exchange as black box

**Price Flow:**
```
Exchange → WebSocket → Gateway → Kafka → Ingestor → InfluxDB + Redis → Price Tracker → User
```

**Order Flow:**
```
User → Kafka → Validator → Kafka → Order Service → Gateway → Exchange → Kafka → Order Tracker → DB + Notification
```

**Key Optimizations:**
- Validate locally (save exchange API costs)
- Kafka buffer (handle backpressure)
- InfluxDB (optimized for time-series)
- Redis PubSub (< 10ms latency)

**Performance:**
- Real-time price: < 50ms
- Order placement: < 100ms
- Millions of concurrent users

**End of Lecture 12**
