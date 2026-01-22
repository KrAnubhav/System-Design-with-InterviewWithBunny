# Lecture 17: Payment Gateway System Design

> **Lecture:** 17  
> **Topic:** System Design  
> **Application:** Payment Gateway (Stripe, Razorpay, PayU)  
> **Scale:** 10,000 TPS (Transactions Per Second)  
> **Difficulty:** Very Hard (Domain-Specific)  
> **Previous:** [[Lecture-16-Distributed-Logging-System-Design|Lecture 16: Distributed Logging System]]

---

## ⚠️ Important Note

**This is a domain-centric problem!**

Unless you have payment industry experience, this question is **unlikely** in interviews. However, if asked, this guide covers everything needed.

**Key Distinction:** We're designing a **Payment Gateway**, NOT a Payment Processor.

---

## 📋 Table of Contents

1. [Payment Gateway vs Payment Processor](#1-payment-gateway-vs-payment-processor)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [Payment Flow (3 Steps)](#4-payment-flow-3-steps)
5. [API Design](#5-api-design)
6. [High-Level Design (HLD)](#6-high-level-design-hld)
7. [Low-Level Design (LLD)](#7-low-level-design-lld)
8. [Complete Architecture Diagram](#8-complete-architecture-diagram)
9. [Interview Q&A](#9-interview-qa)
10. [Key Takeaways](#10-key-takeaways)

---

<br>

## 1. Payment Gateway vs Payment Processor

### 1.1 Payment Gateway

**Definition:** Platform that collects payment details, secures/tokenizes them, and orchestrates transaction flow with processors.

**Analogy:** Traffic controller for payments

**Examples:** Stripe, Razorpay, PayU, Square

**Responsibilities:**
- Collect card details securely
- Tokenize sensitive data (PCI-DSS compliance)
- Route to appropriate processor
- Manage sessions
- Handle callbacks

---

### 1.2 Payment Processor

**Definition:** Financial network entity that talks to banks and card networks to authorize, capture, and settle money.

**Analogy:** System that actually moves money

**Examples:** Visa, Mastercard, Paytm Payments Bank

**Responsibilities:**
- Talk to banks
- Authorize transactions
- Settle funds
- Handle chargebacks

---

### 1.3 Visual Comparison

```
USER CLICKS "PAY NOW"
    ↓
PAYMENT GATEWAY (Stripe, Razorpay)
    - Collects card details
    - Tokenizes (PCI-DSS)
    - Validates
    - Orchestrates
    ↓
PAYMENT PROCESSOR (Visa, Mastercard)
    - Talks to bank
    - Authorizes transaction
    - Debits money
    - Settles funds
    ↓
BANK (ICICI, HDFC)
```

**In this lecture:** We design the **Gateway**, not the Processor!

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Payment Intent**
   - Client creates payment intent (amount, currency)

2. **Session Creation**
   - Generate secure checkout page
   - Session timeout (10 minutes)

3. **PCI-DSS Compliance**
   - Securely handle card data
   - Tokenize sensitive information
   - Use HSM (Hardware Security Module)

4. **Transaction Status**
   - Track payment lifecycle
   - Provide status to clients

---

### 2.2 Out of Scope

❌ Partial payments  
❌ Refunds/Returns  
❌ Chargebacks  
❌ Recurring payments (subscriptions)

---

### 2.3 Non-Functional Requirements

1. **Scale**
   - **10,000 TPS** (Transactions Per Second)

2. **CAP Theorem**
   - **Highly Consistent (CP):** Money transactions require accuracy
   
   **Why?**
   - ✅ Double-charging is unacceptable
   - ✅ Lost transactions = Lost revenue
   - ⚠️ Some downtime acceptable (maintenance windows)

3. **Latency**
   - **< 200ms** for gateway operations (tokenization, validation)
   - Processor time NOT included (bank communication is external)

4. **Security**
   - **PCI-DSS Level 1** compliance
   - End-to-end encryption (TLS 1.3)
   - Tokenization with HSM

---

## 3. Core Entities

1. **Merchant/Client** - Businesses using gateway (Amazon, Flipkart)
2. **Transaction** - Payment record
3. **Payment Method** - Visa, Mastercard, UPI
4. **User/Customer** - End user making payment
5. **Webhook** - Callback for status updates
6. **Payment Session** - Temporary checkout context

---

## 4. Payment Flow (3 Steps)

### Step 1: Payment Intent

```
User clicks "BUY NOW" on Amazon
    ↓
Amazon → POST /v1/payment/intent
    ↓
Gateway creates intent, stores metadata
    ↓
Response: {paymentIntentId: "pi_12345"}
```

---

### Step 2: Session Creation

```
Amazon → POST /v1/payment/session (with paymentIntentId)
    ↓
Gateway creates session (10-min timeout)
    ↓
Response: {
  sessionId: "sess_67890",
  redirectUrl: "https://gateway.com/checkout?session=sess_67890"
}
    ↓
Amazon redirects user to checkout page
    ↓
User enters card details on Gateway's secure page
```

---

### Step 3: Payment Execution

```
User clicks "PAY NOW"
    ↓
Gateway validates session
    ↓
Tokenizes card data (HSM)
    ↓
Routes to processor (Razorpay, PayU)
    ↓
Processor talks to bank
    ↓
Callback to gateway with status
    ↓
Gateway updates transaction
    ↓
Redirect user to success/failure page
```

---

## 5. API Design

### 5.1 Create Payment Intent

```
POST /v1/payment/intent
```

**Request Body:**
```json
{
  "merchantId": "merchant_amazon",
  "amount": 5000,  // Rs 50.00 (in paisa)
  "currency": "INR",
  "orderId": "order_789",
  "customerId": "user_456",
  "metadata": {
    "items": ["iPhone 15", "AirPods"]
  }
}
```

**Response:**
```json
{
  "paymentIntentId": "pi_abc123",
  "status": "PENDING",
  "createdAt": "2026-01-21T10:00:00Z"
}
```

---

### 5.2 Create Checkout Session

```
POST /v1/payment/session
```

**Request Body:**
```json
{
  "paymentIntentId": "pi_abc123",
  "merchantId": "merchant_amazon",
  "successUrl": "https://amazon.com/order/success",
  "cancelUrl": "https://amazon.com/order/cancel"
}
```

**Response:**
```json
{
  "sessionId": "sess_xyz789",
  "redirectUrl": "https://gateway.com/checkout?session=sess_xyz789",
  "expiresAt": "2026-01-21T10:10:00Z"  // 10 minutes
}
```

---

### 5.3 Process Payment (Internal)

**User clicks "PAY NOW" on checkout page**

```
POST /internal/payment/process
```

**Request Body:**
```json
{
  "sessionId": "sess_xyz789",
  "cardNumber": "4111111111111111",
  "expiryMonth": "12",
  "expiryYear": "2028",
  "cvv": "123",
  "cardholderName": "John Doe"
}
```

**Response:**
```json
{
  "transactionId": "txn_999",
  "status": "PROCESSING"
}
```

---

### 5.4 Get Transaction Status

```
GET /v1/payment/transaction/{transactionId}
```

**Response:**
```json
{
  "transactionId": "txn_999",
  "paymentIntentId": "pi_abc123",
  "status": "SUCCESS",  // PENDING, PROCESSING, SUCCESS, FAILED
  "amount": 5000,
  "currency": "INR",
  "completedAt": "2026-01-21T10:05:32Z"
}
```

---

## 6. High-Level Design (HLD)

### 6.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│          MERCHANT (Amazon, Flipkart)                     │
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
│ Payment  │      │ Session  │          │Checkout  │
│ Intent   │      │ Service  │          │ Backend  │
│ Service  │      └────┬─────┘          └────┬─────┘
└────┬─────┘           │                     │
     │                 ▼                     ▼
     ▼           ┌──────────┐          ┌──────────┐
┌──────────┐    │  Redis   │          │Tokenizer │
│Intent DB │    │ (Session)│          │(PCI Zone)│
└──────────┘    └──────────┘          └────┬─────┘
                                           │
                                           ▼
                                     ┌──────────┐
                                     │Orchestrator
                                     └────┬─────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │  Processor   │
                                   │  (External)  │
                                   └──────────────┘
```

---

## 7. Low-Level Design (LLD)

### 7.1 Payment Intent Service

**Purpose:** Create intent, store payment metadata

**Payment Intent DB (PostgreSQL):**

```sql
CREATE TABLE payment_intents (
    payment_intent_id UUID PRIMARY KEY,
    merchant_id VARCHAR(100),
    customer_id VARCHAR(100),
    order_id VARCHAR(100),
    amount BIGINT,  -- In smallest currency unit (paisa)
    currency VARCHAR(3),  -- INR, USD
    payment_method_type VARCHAR(50),  -- CREDIT_CARD, DEBIT_CARD, UPI
    status VARCHAR(20),  -- PENDING, PROCESSING, SUCCESS, FAILED
    metadata JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_merchant ON payment_intents(merchant_id);
CREATE INDEX idx_order ON payment_intents(order_id);
```

**Why PostgreSQL?**
- ✅ ACID compliance (money = consistency!)
- ✅ Strong consistency
- ✅ Relational integrity

---

### 7.2 Session Service

**Purpose:** Create secure checkout session

**Session Storage (Redis):**

```
Key: session:{sessionId}
Value: {
  "paymentIntentId": "pi_abc123",
  "merchantId": "merchant_amazon",
  "transactionId": "txn_999",
  "orderId": "order_789",
  "customerId": "user_456",
  "createdAt": "2026-01-21T10:00:00Z"
}
TTL: 600 seconds (10 minutes)
```

**Why Redis?**
- ✅ Fast access (< 200ms requirement)
- ✅ Built-in TTL (session expiration)
- ✅ In-memory (low latency)

---

**Session Flow:**

```
1. Merchant → POST /v1/payment/session
    ↓
2. Session Service:
    - Generate sessionId (UUID)
    - Store in Redis (TTL: 10 mins)
    ↓
3. Return redirectUrl:
    https://gateway.com/checkout?session={sessionId}
```

---

### 7.3 Checkout Frontend Service

**Purpose:** Render secure checkout page

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│          USER REDIRECTED TO CHECKOUT                     │
│   https://gateway.com/checkout?session=sess_xyz789       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Load Balancer  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Checkout       │
                │  Frontend       │
                │  Service (HTML) │
                └─────────────────┘
```

**HTML Page Example:**

```html
<form action="/internal/payment/process" method="POST">
  <input type="hidden" name="sessionId" value="sess_xyz789">
  
  <label>Card Number</label>
  <input type="text" name="cardNumber" maxlength="16">
  
  <label>Expiry Date</label>
  <input type="text" name="expiryMonth" placeholder="MM">
  <input type="text" name="expiryYear" placeholder="YYYY">
  
  <label>CVV</label>
  <input type="text" name="cvv" maxlength="3">
  
  <label>Cardholder Name</label>
  <input type="text" name="cardholderName">
  
  <button type="submit">PAY NOW</button>
</form>

<script>
  // Session timer
  let timeLeft = 600; // 10 minutes
  setInterval(() => {
    timeLeft--;
    if (timeLeft <= 0) {
      window.location.href = '/session-expired';
    }
    document.getElementById('timer').innerText = formatTime(timeLeft);
  }, 1000);
</script>
```

**Key Point:** Card details NEVER touch merchant's servers!

---

### 7.4 Checkout Backend Service (Validation)

**Purpose:** Validate session and route to tokenizer

**Validation Flow:**

```java
void processPayment(PaymentRequest request) {
    String sessionId = request.getSessionId();
    
    // 1. Check session exists
    SessionData session = redis.get("session:" + sessionId);
    if (session == null) {
        throw new SessionExpiredException();
    }
    
    // 2. Validate payment intent
    PaymentIntent intent = db.findById(session.getPaymentIntentId());
    if (intent == null || intent.getStatus() != "PENDING") {
        throw new InvalidPaymentException();
    }
    
    // 3. Validate amount hasn't changed
    if (intent.getAmount() != request.getAmount()) {
        throw new AmountMismatchException();
    }
    
    // 4. Call tokenizer (TLS connection)
    TokenResponse token = tokenizerService.tokenize(request.getCardDetails());
    
    // 5. Route to orchestrator
    orchestrator.processPayment(intent, token);
}
```

---

### 7.5 Tokenization Service (PCI Zone) ⭐

**Purpose:** Securely tokenize card data

**PCI-DSS Compliance:** All card data handling MUST be in PCI Zone

---

#### **7.5.1 Card Number Structure**

```
Card: 4111 1111 1111 1234

First 6 digits: BIN (Bank Identification Number)
    4111 11 → Visa, ICICI Bank
    
Last 4 digits: Unique identifier
    1234
    
Middle: Account number (encrypted)
```

---

#### **7.5.2 Tokenization Steps**

**Step 1: Validate Card**

```java
boolean validateCard(String cardNumber) {
    // Luhn algorithm (checksum)
    int sum = 0;
    boolean alternate = false;
    for (int i = cardNumber.length() - 1; i >= 0; i--) {
        int digit = cardNumber.charAt(i) - '0';
        if (alternate) {
            digit *= 2;
            if (digit > 9) digit -= 9;
        }
        sum += digit;
        alternate = !alternate;
    }
    return sum % 10 == 0;
}
```

---

**Step 2: Generate Fingerprint**

```java
String generateFingerprint(CardDetails card) {
    String data = 
        card.getBin() +           // First 6 digits
        card.getLast4() +         // Last 4 digits
        card.getExpiryMonth() +
        card.getExpiryYear() +
        card.getCardholderName();
    
    return SHA256(data);
}
```

**Example:**
```
Input:
  BIN: 411111
  Last4: 1234
  Expiry: 12/2028
  Name: John Doe

Fingerprint: a3f8d719e4b2c1d0... (SHA-256 hash)
```

---

**Step 3: Encrypt with HSM**

**What is HSM (Hardware Security Module)?**

A **physical device** that generates and stores encryption keys securely.

**Why HSM?**
- ✅ Keys never leave hardware
- ✅ FIPS 140-2 Level 3 certified
- ✅ Tamper-resistant

**Encryption:**

```
Card PAN (Primary Account Number): 4111111111111234
    ↓
Send to HSM
    ↓
HSM uses internal key to encrypt
    ↓
Encrypted Token: enc_a8f3d9c2b1e4...
```

**Key Point:** Encryption key is stored **inside HSM hardware**, NOT in software!

---

**Complete Tokenization Flow:**

```
┌─────────────────────────────────────────────────────────┐
│          CHECKOUT BACKEND SERVICE                        │
└────────────────────────┬────────────────────────────────┘
                         │ (TLS 1.3)
                         ▼
                ┌─────────────────┐
                │  Tokenization   │
                │    Service      │
                │   (PCI Zone)    │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Validate │    │Generate  │    │  HSM     │
  │  Card    │    │Fingerprint    │ Encrypt  │
  │ (Luhn)   │    │ (SHA-256)│    │          │
  └──────────┘    └──────────┘    └──────────┘
                         │
                         ▼
                   Encrypted Token
```

**Response:**
```json
{
  "token": "tok_a8f3d9c2b1e4...",
  "fingerprint": "fp_411111****1234",
  "cardBrand": "VISA",
  "expiryMonth": "12",
  "expiryYear": "2028"
}
```

---

### 7.6 Orchestrator Service

**Purpose:** Route payment to correct processor

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│          ORCHESTRATOR SERVICE                            │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Merchant │    │Transaction│   │ Processor│
  │Preference│    │    DB     │    │Connector │
  │   DB     │    │ (Insert)  │    │(Adapter) │
  └──────────┘    └──────────┘    └────┬─────┘
                                       │
                        ┌──────────────┼──────────────┐
                        │              │              │
                        ▼              ▼              ▼
                   ┌──────────┐  ┌──────────┐  ┌──────────┐
                   │Razorpay  │  │  PayU    │  │  Stripe  │
                   │Connector │  │Connector │  │Connector │
                   └────┬─────┘  └────┬─────┘  └────┬─────┘
                        │             │             │
                        ▼             ▼             ▼
                   ┌─────────────────────────────────────┐
                   │      PROCESSOR GATEWAY (External)   │
                   └─────────────────────────────────────┘
```

---

**Merchant Preference DB (PostgreSQL):**

```sql
CREATE TABLE merchant_preferences (
    merchant_id VARCHAR(100) PRIMARY KEY,
    preferred_processor VARCHAR(50),  -- RAZORPAY, PAYU, STRIPE
    fallback_processor VARCHAR(50),
    transaction_fee_percent DECIMAL(5, 2),
    config JSONB
);
```

**Example:**
```sql
INSERT INTO merchant_preferences VALUES (
    'merchant_amazon',
    'RAZORPAY',
    'PAYU',
    1.5,  -- 1.5% fee
    '{"api_key": "rzp_live_xxx", "webhook_secret": "xxx"}'
);
```

---

**Transaction DB (PostgreSQL):**

```sql
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY,
    payment_intent_id UUID,
    merchant_id VARCHAR(100),
    customer_id VARCHAR(100),
    amount BIGINT,
    currency VARCHAR(3),
    status VARCHAR(20),  -- SENT, PROCESSING, SUCCESS, FAILED
    processor VARCHAR(50),  -- RAZORPAY, PAYU
    processor_txn_id VARCHAR(100),
    token VARCHAR(500),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_intent ON transactions(payment_intent_id);
CREATE INDEX idx_status ON transactions(status);
```

---

**Orchestrator Flow:**

```java
void processPayment(PaymentIntent intent, TokenResponse token) {
    // 1. Get merchant's processor preference
    MerchantPreference pref = db.getMerchantPreference(intent.getMerchantId());
    
    // 2. Create transaction record (status: SENT)
    Transaction txn = new Transaction();
    txn.setTransactionId(UUID.randomUUID());
    txn.setPaymentIntentId(intent.getPaymentIntentId());
    txn.setStatus("SENT");
    txn.setProcessor(pref.getPreferredProcessor());
    txn.setToken(token.getToken());
    db.save(txn);
    
    // 3. Route to processor connector
    ProcessorConnector connector = getConnector(pref.getPreferredProcessor());
    
    // 4. Call processor
    ProcessorResponse response = connector.charge(
        txn.getTransactionId(),
        token.getToken(),
        intent.getAmount(),
        intent.getCurrency()
    );
    
    // 5. Update transaction (status: PROCESSING)
    txn.setProcessorTxnId(response.getProcessorTxnId());
    txn.setStatus("PROCESSING");
    db.update(txn);
}
```

---

**Adapter Pattern (Processor Connectors from):**

```java
interface ProcessorConnector {
    ProcessorResponse charge(UUID txnId, String token, long amount, String currency);
}

class RazorpayConnector implements ProcessorConnector {
    ProcessorResponse charge(...) {
        // Build Razorpay-specific request
        RazorpayRequest req = new RazorpayRequest();
        req.setAmount(amount);
        req.setCardToken(token);
        
        // Call Razorpay API
        return razorpayClient.processPayment(req);
    }
}

class PayUConnector implements ProcessorConnector {
    ProcessorResponse charge(...) {
        // Build PayU-specific request
        PayURequest req = new PayURequest();
        req.setTxnAmount(amount);
        req.setToken(token);
        
        // Call PayU API
        return payuClient.charge(req);
    }
}
```

---

### 7.7 Processor Gateway (External)

**Out of Scope:** This is the external processor (Razorpay, PayU, Stripe)

**What it does:**
- Talks to card networks (Visa, Mastercard)
- Authorizes with bank
- Debits money
- Settles funds

**Callback:** Sends status back to gateway

---

### 7.8 Callback Service (Async Status Updates)

**Purpose:** Receive async status from processor

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│          PROCESSOR (Razorpay, PayU)                      │
└────────────────────────┬────────────────────────────────┘
                         │ (Webhook)
                         ▼
                ┌─────────────────┐
                │   Callback      │
                │   Service       │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │ (payment_status)│
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Orchestrator   │Transaction│   │ Webhook  │
  │(Update DB)│    │   DB     │    │ Service  │
  └──────────┘    └──────────┘    └────┬─────┘
                                       │
                                       ▼
                               ┌──────────────┐
                               │   MERCHANT   │
                               │  (Callback)  │
                               └──────────────┘
```

---

**Kafka Topics:**

```
1. payment_callback_status (Immediate)
   - Processor sends initial acknowledgment
   - "Order placed with bank"
   
2. payment_processor_status (Delayed - 1 day)
   - Final settlement confirmation
   - "Money actually debited"
```

---

**Callback Flow:**

```
Step 1: Processor calls webhook
    POST https://gateway.com/webhooks/razorpay
    {
      "txnId": "txn_999",
      "processorTxnId": "rzp_tx_abc",
      "status": "SUCCESS",
      "timestamp": "2026-01-21T10:05:32Z"
    }
    ↓
Step 2: Callback Service validates signature
    - HMAC verification with shared secret
    ↓
Step 3: Publish to Kafka (payment_callback_status)
    ↓
Step 4: Orchestrator consumes from Kafka
    ↓
Step 5: Update Transaction DB (status: SUCCESS)
    ↓
Step 6: Webhook Service notifies merchant
    POST https://amazon.com/webhooks/payment
    {
      "transactionId": "txn_999",
      "paymentIntentId": "pi_abc123",
      "status": "SUCCESS"
    }
```

---

### 7.9 Reconciliation Service ⭐

**Purpose:** Final settlement confirmation (Day T+1)

**Problem:**

```
Initial Response (Immediate):
  "Order placed with bank" ✅
  
Actual Settlement (Next Day):
  - Insufficient funds → FAILED ❌
  - Bank declined → FAILED ❌
  - Success → SUCCESS ✅
```

**Example Scenario:**

```
Day 1, 10:00 AM:
  User pays Rs 5,000
  Processor says: "Order placed"
  Gateway shows: "Payment Successful"
  Amazon ships product

Day 2, 2:00 AM (Settlement):
  Bank declines (insufficient funds)
  Processor sends: "FAILED"
  
Reconciliation:
  - Update transaction: FAILED
  - Notify Amazon
  - Amazon cancels order/requests re-payment
```

---

**Reconciliation Flow:**

```
┌─────────────────────────────────────────────────────────┐
│          PROCESSOR (Daily Settlement)                    │
└────────────────────────┬────────────────────────────────┘
                         │ (File or API)
                         ▼
                ┌─────────────────┐
                │ Reconciliation  │
                │    Service      │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Transaction│   │ Ledger   │    │ Webhook  │
  │   DB     │    │   DB     │    │ Service  │
  │(Expected)│    │(Final)   │    │(Notify)  │
  └──────────┘    └──────────┘    └──────────┘
```

---

**Ledger DB (Final Record):**

```sql
CREATE TABLE payment_ledger (
    ledger_id UUID PRIMARY KEY,
    transaction_id UUID,
    payment_intent_id UUID,
    merchant_id VARCHAR(100),
    customer_id VARCHAR(100),
    amount BIGINT,
    currency VARCHAR(3),
    final_status VARCHAR(20),  -- SUCCESS, FAILED
    settled_at TIMESTAMP,
    reconciled_at TIMESTAMP
);
```

---

**Reconciliation Logic:**

```java
void reconcile() {
    // 1. Fetch settlement file from processor
    List<Settlement> settlements = processor.getSettlements(LocalDate.now().minusDays(1));
    
    // 2. Compare with our transactions
    for (Settlement settlement : settlements) {
        Transaction txn = db.findByProcessorTxnId(settlement.getProcessorTxnId());
        
        // 3. Check mismatch
        if (txn.getStatus() == "SUCCESS" && settlement.getStatus() == "FAILED") {
            // Mismatch! Update and notify
            txn.setStatus("FAILED");
            db.update(txn);
            
            // Insert into ledger
            ledgerDb.insert(new LedgerEntry(txn, "FAILED"));
            
            // Notify merchant
            webhookService.notify(txn.getMerchantId(), txn, "FAILED");
        } else {
            // Match! Confirm in ledger
            ledgerDb.insert(new LedgerEntry(txn, settlement.getStatus()));
        }
    }
}
```

---

## 8. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│              MERCHANT (Amazon)                           │
│                    ↓                                     │
│              CUSTOMER (User)                             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│ Payment  │      │ Session  │          │Checkout  │
│ Intent   │      │ Service  │          │ Frontend │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     ▼                 ▼                     │
┌──────────┐      ┌──────────┐              │
│Intent DB │      │  Redis   │              │
└──────────┘      └──────────┘              │
                                            ▼
                                     ┌──────────┐
                                     │Checkout  │
                                     │ Backend  │
                                     └────┬─────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │ Tokenization │
                                   │   Service    │
                                   │  (PCI Zone)  │
                                   └────┬─────────┘
                                        │
                                        ▼
                                   ┌──────────┐
                                   │   HSM    │
                                   └────┬─────┘
                                        │
                                        ▼
                                   ┌──────────┐
                                   │Orchestrator
                                   └────┬─────┘
                                        │
                ┌───────────────────────┼───────────────┐
                │                       │               │
                ▼                       ▼               ▼
          ┌──────────┐          ┌──────────┐    ┌──────────┐
          │Merchant  │          │Transaction│   │Razorpay  │
          │Pref DB   │          │    DB     │    │Connector │
          └──────────┘          └──────────┘    └────┬─────┘
                                                     │
                                                     ▼
                                              ┌──────────────┐
                                              │  Processor   │
                                              │  Gateway     │
                                              └────┬─────────┘
                                                   │
                                                   ▼
                                            ┌──────────────┐
                                            │   Callback   │
                                            │   Service    │
                                            └────┬─────────┘
                                                 │
                                                 ▼
                                            ┌──────────┐
                                            │  Kafka   │
                                            └────┬─────┘
                                                 │
                        ┌────────────────────────┼────────────┐
                        │                        │            │
                        ▼                        ▼            ▼
                   ┌──────────┐          ┌──────────┐  ┌──────────┐
                   │Orchestrator│         │Webhook   │  │Reconciliation
                   │(Update)  │          │ Service  │  │  Service │
                   └──────────┘          └──────────┘  └────┬─────┘
                                                            │
                                                            ▼
                                                      ┌──────────┐
                                                      │ Ledger   │
                                                      │   DB     │
                                                      └──────────┘
```

---

## 9. Interview Q&A

### Q1: Why use Redis for sessions instead of database?

**A:**

**Latency Requirement:** < 200ms

| Storage | Read Latency | Write Latency |
|---------|--------------|---------------|
| PostgreSQL | 10-50ms | 20-100ms |
| Redis | < 1ms | < 1ms |

**Session Characteristics:**
- ✅ Short-lived (10 mins)
- ✅ High read frequency
- ✅ No persistence needed (TTL auto-expires)

### Q2: What is PCI-DSS? Why is it important?

**A:**

**PCI-DSS:** Payment Card Industry Data Security Standard

**Levels:**
- Level 1: > 6M transactions/year (most strict)
- Level 2: 1M - 6M transactions/year
- Level 3: 20K - 1M transactions/year
- Level 4: < 20K transactions/year

**Requirements:**
1. Never store CVV
2. Encrypt card data at rest
3. Use HSM for key management
4. Network segmentation (PCI Zone)
5. Regular security audits

**Violation Penalty:** $5,000 - $500,000 per incident!

### Q3: Why use HSM instead of software encryption?

**A:**

**Software Encryption:**
```
Encryption key stored in:
  - Config file ❌
  - Environment variable ❌
  - Database ❌
  
All can be compromised!
```

**HSM (Hardware Security Module):**
```
Key stored in tamper-proof hardware
    ↓
Physical attack → Key self-destructs
    ↓
FIPS 140-2 Level 3 certified
```

**Cost:** $10,000 - $100,000 per HSM
**Benefit:** PCI-DSS compliance + Security

### Q4: Why immediate callback + reconciliation?

**A:**

**Two-Phase Confirmation:**

**Phase 1: Immediate (< 1 sec)**
```
User pays → Processor acknowledges
    ↓
Status: "Order placed with bank"
    ↓
Show success to user (UX)
```

**Phase 2: Settlement (T+1)**
```
Bank actually debits money
    ↓
Processor confirms final status
    ↓
Reconcile with our records
```

**Why?**
- ✅ Immediate UX (don't make user wait 24 hours!)
- ✅ Final accuracy (catch failures)

### Q5: How to handle duplicate payments?

**A:**

**Idempotency Key:**

```java
@PostMapping("/v1/payment/process")
void processPayment(@Header("Idempotency-Key") String key, ...) {
    // Check if already processed
    if (redis.exists("idempotency:" + key)) {
        Transaction existing = redis.get("idempotency:" + key);
        return existing;  // Return cached result
    }
    
    // Process payment
    Transaction txn = orchestrator.processPayment(...);
    
    // Cache result (24h TTL)
    redis.setex("idempotency:" + key, 86400, txn);
    
    return txn;
}
```

**Client Side:**
```
POST /v1/payment/process
Headers:
  Idempotency-Key: unique_key_per_request
```

### Q6: How to handle processor failures?

**A:**

**Fallback Strategy:**

```java
void processPayment(PaymentIntent intent, TokenResponse token) {
    MerchantPreference pref = db.getMerchantPreference(intent.getMerchantId());
    
    try {
        // Try primary processor
        return processorFactory
            .getConnector(pref.getPreferredProcessor())
            .charge(...);
    } catch (ProcessorDownException e) {
        // Fallback to secondary
        log.warn("Primary processor down, falling back");
        return processorFactory
            .getConnector(pref.getFallbackProcessor())
            .charge(...);
    }
}
```

### Q7: How to scale to 10,000 TPS?

**A:**

**Bottlenecks & Solutions:**

| Component | Bottleneck | Solution |
|-----------|------------|----------|
| **Session Service** | Redis single-threaded | Redis Cluster (10+ nodes) |
| **Tokenizer** | HSM throughput | Multiple HSMs (load balanced) |
| **Orchestrator** | DB writes | Connection pooling, async writes |
| **Callback** | Kafka lag | 100 partitions, 100 consumers |

**Calculation:**
```
10,000 TPS
    ↓
Each transaction: 3 DB writes
    ↓
30,000 writes/sec required
    ↓
PostgreSQL: ~10,000 writes/sec per instance
    ↓
Need 3 write replicas (sharding by merchant_id)
```

### Q8: Security considerations?

**A:**

1. **TLS 1.3:** All inter-service communication
2. **No Logging:** Never log card numbers/CVV
3. **PCI Zone Isolation:** Network firewall
4. **Rate Limiting:** Prevent brute-force (card testing)
5. **3DS (3D Secure):** Two-factor authentication for cards
6. **Webhook Validation:** HMAC signature verification

### Q9: What if session expires during payment?

**A:**

**Scenario:**
```
User fills card details (9 mins)
    ↓
Clicks "PAY NOW" (10 mins 1 sec)
    ↓
Session expired!
```

**Solution:**

```java
void processPayment(String sessionId, ...) {
    SessionData session = redis.get("session:" + sessionId);
    
    if (session == null) {
        // Extend grace period
        session = redis.get("session:expired:" + sessionId);
        
        if (session != null && isWithinGracePeriod(session, 60)) {
            // Allow payment within 1-minute grace period
            return processPaymentInternal(session, ...);
        }
        
        throw new SessionExpiredException();
    }
}
```

### Q10: How to test without real money?

**A:**

**Test Mode:**

```java
if (environment.equals("TEST")) {
    // Use test processor
    return new TestProcessorConnector();  // Always returns SUCCESS
}
```

**Test Cards (Razorpay):**
```
4111 1111 1111 1111 → SUCCESS
4000 0000 0000 0002 → FAILED (Declined)
4000 0000 0000 0119 → FAILED (Insufficient funds)
```

---

## 10. Key Takeaways

✅ **Gateway ≠ Processor:** Gateway tokenizes, Processor moves money  
✅ **3-Step Flow:** Intent → Session → Payment  
✅ **PCI-DSS:** HSM for encryption, TLS for transport  
✅ **Tokenization:** BIN + Last4 + Expiry + Name → SHA-256  
✅ **HSM:** Physical device, keys never leave hardware  
✅ **Orchestrator:** Adapter pattern for multiple processors  
✅ **Async Callback:** Kafka for status updates  
✅ **Reconciliation:** T+1 settlement confirmation  
✅ **Idempotency:** Prevent duplicate charges  
✅ **Session:** Redis with TTL (10 mins)  

---

## Summary

**Payment Gateway Flow:**
```
Step 1: Intent
  Merchant → POST /v1/payment/intent
  Gateway → Returns paymentIntentId

Step 2: Session
  Merchant → POST /v1/payment/session
  Gateway → Returns redirectUrl
  User → Enters card details on gateway's page

Step 3: Payment
  User → Clicks "PAY NOW"
  Gateway → Validates session
          → Tokenizes (HSM)
          → Routes to processor
  Processor → Talks to bank
           → Sends callback
  Gateway → Updates transaction
         → Notifies merchant

Day T+1: Reconciliation
  Processor → Sends settlement file
  Gateway → Compares with ledger
         → Fixes mismatches
```

**Security Layers:**
1. TLS 1.3 (transport)
2. Tokenization (storage)
3. HSM (key management)
4. PCI Zone (network isolation)

**Architecture Highlights:**
- 8 microservices
- 5 databases (PostgreSQL for consistency)
- Redis for sessions (TTL)
- Kafka for async callbacks
- HSM for encryption
- Adapter pattern for processors

**Performance:**
- 10,000 TPS
- < 200ms latency (gateway only)
- Redis Cluster for scaling
- PostgreSQL sharding by merchant_id

**End of Lecture 17**
