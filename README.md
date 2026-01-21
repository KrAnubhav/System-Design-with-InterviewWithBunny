# System Design with Interview With Bunny 🐰

Comprehensive system design notes based on the "System Design with Interview With Bunny" series. These notes cover functional/non-functional requirements, high-level and low-level designs, database schemas, and interview-style Q&A.

## 📚 Curriculum

| # | Topic | Key Concepts |
|---|---|---|
| 01 | [URL Shortener](./Lecture-01-URL-Shortener-System-Design.md) | Scalable Hashing, Counter-based approach, Zookeeper/Snowflake |
| 02 | [Ticket Booking (BookMyShow)](./Lecture-02-Ticket-Booking-System-Design.md) | Concurrency, Distributed Locking (Redis), Isolation Levels |
| 03 | [E-Commerce Platform (Amazon)](./Lecture-03-E-Commerce-Platform-System-Design.md) | Inventory Management, Search, Consistent Hashing |
| 04 | [OTT Platform (Netflix)](./Lecture-04-OTT-Platform-System-Design.md) | Video Encoding, CDN, Distributed Caching, Microservices |
| 05 | [Hotel Booking (Airbnb)](./Lecture-05-Hotel-Booking-System-Design.md) | Availability Tracking, Booking Flow, Consistency Trade-offs |
| 06 | [Food Delivery (Zomato/Swiggy)](./Lecture-06-Food-Delivery-System-Design.md) | Real-time Tracking, Delivery Matching, Order Lifecycle |
| 07 | [Proximity Search Algorithms](./Lecture-07-Proximity-Search-Algorithms.md) | Geohash, Quadtree, S2 Geometry, Spatial Indexing |
| 08 | [Chat Application (WhatsApp)](./Lecture-08-Chat-Application-System-Design.md) | WebSockets, Message Routing (Redis Stream), Persistence (Cassandra) |
| 09 | [Distributed Job Scheduler](./Lecture-09-Distributed-Job-Scheduler-System-Design.md) | Kafka, Heartbeats, Dead-letter Queues, Retry Mechanisms |
| 10 | [Collaborative Text Editor](./Lecture-10-Collaborative-Text-Editor-System-Design.md) | OT (Operational Transformation), CRDT, Delta Sync, Redis Canonical Copy |
| 11 | [Ride Booking (Uber/Ola)](./Lecture-11-Ride-Booking-Application-System-Design.md) | Driver Matching, Proximity Search, Surge Pricing, Zookeeper Locks |
| 12 | [Stock Trading (Zerodha/Groww)](./Lecture-12-Stock-Trading-Platform-System-Design.md) | Real-time Market Data, WebSockets, Kafka, Time-series DB (InfluxDB) |

---

## 🚀 How to use these notes
- **Revision Ready:** Each lecture follows a strict structure for quick revision.
- **Visuals Included:** ASCII diagrams represent High-Level and Low-Level designs.
- **Interview Focus:** Every file ends with a "Top 10 Interview Q&A" section.
- **Obsidain Friendly:** Optimized for Obsidian markdown formatting with internal links.

---

## 🛠️ Tech Stack Patterns Covered
- **Communication:** WebSockets, gRPC, HTTP Polling, SSE.
- **Storage:** PostgreSQL (ACID), Cassandra (AP), Redis (In-memory), InfluxDB (TSDB), S3.
- **Queuing:** Kafka, Redis Streams.
- **Coordination:** Zookeeper, Redis Locks.
- **Algorithms:** OT, CRDT, Geohash, Consistent Hashing.

---
*Happy System Designing!* 🐰💻