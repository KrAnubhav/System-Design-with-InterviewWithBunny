# System Design with InterviewWithBunny 🐰

Welcome to the **System Design Masterclass**! This repository contains comprehensive, revision-friendly notes for **System Design (HLD + LLD)** based on the popular lecture series by *InterviewWithBunny*.

These notes are efficiently structured in **Markdown**, featuring **ASCII diagrams**, **database schemas**, **API designs**, and **real-world trade-offs**, making them perfect for last-minute revisions before your tech interviews.

---

## 📚 Curriculum (Lecture Notes)

| Lecture | Topic / System | Key Concepts Covered | Status |
| :---: | :--- | :--- | :---: |
| **01** | **URL Shortener** (TinyURL/Bitly) | Hashing (Base62), KGS, Caching, Redirects (301 vs 302) | ✅ |
| **02** | **Rate Limiter** | Token Bucket, Leaky Bucket, Sliding Window, Redis Lua, Throttling | ✅ |
| **03** | **E-Commerce Platform** (Amazon/Flipkart) | Microservices, Inventory Locking (Optimistic vs Pessimistic), Cart Mgmt | ✅ |
| **04** | **Consistent Hashing** | Replication, Virtual Nodes, Partitioning, Load Balancing | ✅ |
| **05** | **Design Key-Value Store** (DynamoDB/Cassandra) | Quorum (R+W>N), Gossip Protocol, Merkle Trees, SSTables, WAL | ✅ |
| **06** | **Unique ID Generator** (Snowflake) | Twitter Snowflake, Sequence Generation, Concurrency, NTP Issues | ✅ |
| **07** | **Design Cache** (Redis/Memcached) | Eviction Policies (LRU/LFU), Write-Through, Write-Back, Consistency | ✅ |
| **08** | **Chat Application** (WhatsApp/Telegram) | WebSockets, Long Polling, Presence Service, Message Queues (Kafka) | ✅ |
| **09** | **Distributed Job Scheduler** | Quartz, Dead Letter Queues, Partitioning, Leader Election | ✅ |
| **10** | **Collaborative Text Editor** (Google Docs) | Operational Transformation (OT), CRDTs, WebSocket, Conflict Resolution | ✅ |
| **11** | **Ride Booking System** (Uber/Ola) | QuadTrees, Geohashing, Location Tracking, Matching Algorithms | ✅ |
| **12** | **Stock Trading Platform** (Zerodha) | High Frequency Trading, Matching Engine, Order Book, ACID Compliance | ✅ |
| **13** | **Social Media Algorithm** (Facebook Feed) | Fan-out (Push vs Pull), EdgeRank, Caching Strategies (Redis) | ✅ |
| **14** | **Leaderboard System** (Gaming) | Redis Sorted Sets (Skip List), Real-time Ranking, sparse updates | ✅ |
| **15** | **Notification System** | Fan-out, Priority Queues, Idempotency, 3rd Party Integration (FCM/SES) | ✅ |
| **16** | **Distributed Logging** (ELK/Splunk) | Log Aggregation, Kafka, ElasticSearch, Sidecar Agents (Fluentd) | ✅ |
| **17** | **Payment Gateway** (Stripe/Razorpay) | PCI-DSS, Idempotency, Double-Entry Ledger, Reconciliation, 2PC | ✅ |
| **18** | **Distributed Cloud Storage** (Google Drive/Dropbox) | Chunking, De-duplication, Synchronization, Block Storage (S3), Metadata | ✅ |
| **19** | *Coming Soon...* | | ⏳ |

---

## 🛠 Tech Stack & Patterns Covered

This repository isn't just about specific apps; it's about mastering the **patterns** used to build them.

*   **Communication:** REST, gRPC, GraphQL, WebSockets, Long Polling, SSE.
*   **Databases:**
    *   **SQL (ACID):** PostgreSQL, MySQL (Sharding, Replication).
    *   **NoSQL (BASE):** Cassandra (Write-heavy), MongoDB (Document), DynamoDB.
    *   **Time Series:** InfluxDB, Prometheus.
*   **Caching:** Redis (Distributed Lock, Pub/Sub, Sorted Sets), Memcached, CDN.
*   **Message Queues:** Kafka (Partitions, Consumer Groups), RabbitMQ.
*   **Algorithms:** Consistent Hashing, QuadTrees, GeoHashing, Token Bucket, Leaky Bucket, Operational Transformation (OT).
*   **Concepts:** CAP Theorem, PACELC, Strong vs Eventual Consistency, Quorum, Idempotency, Rate Limiting.

---

## 📖 How to Use These Notes

1.  **Understand the Flow:** Each note follows a strict structure:
    *   **Requirements:** Functional & Non-Functional.
    *   **Capacity Estimation:** Math behind the scale.
    *   **API Design:** RESTful endpoints.
    *   **Database Design:** Schema & choice of DB.
    *   **High-Level Design (HLD):** Block diagrams.
    *   **Deep Dive:** Handling concurrency, failures, and scaling.
2.  **Visual Learning:** Pay attention to the **ASCII diagrams**; they are designed to be reproducible on a whiteboard during interviews.
3.  **Interview Q&A:** The end of each file contains "Bunny's Tricky Questions" – typical curveballs interviewers throw.

---

## 🤝 Contribution

Found a bug? Want to add a better diagrams?
Feel free to open a **Pull Request**! Let's make this the ultimate System Design resource.

---

*Made with ❤️ by [Anubhav] & InterviewWithBunny Community*
