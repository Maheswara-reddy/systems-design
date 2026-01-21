# 🚀 Senior System Design (HLD) Master List
*A comprehensive guide for SDE 2/3 and Senior Member of Technical Staff (IC3) interviews.*

---

## 1. Foundational Systems
Essential for testing core concepts like Load Balancing, Sharding, and Hashing.

### [1] URL Shortener (TinyURL)
- **Key Concepts:** Base62 encoding, unique ID generation (Snowflake), database sharding.
- **Resources:**
  - [Alex Xu (Vol 1) - Chapter 8](https://bytebytego.com/courses/system-design-interview/design-a-url-shortener)
  - [Coding Horror: URL Shortening Hashes](https://blog.codinghorror.com/url-shortening-hashes-in-practice/)

### [2] Distributed Cache (Redis)
- **Key Concepts:** Consistent hashing, eviction policies (LRU/LFU), Cache Stampede prevention.
- **Resources:**
  - [Hello Interview: Design a Distributed Cache](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-cache)
  - [Redis.io: Leaderboards & Caching](https://redis.io/solutions/leaderboards/)

### [3] API Rate Limiter
- **Key Concepts:** Token Bucket vs. Fixed Window, distributed locking with Redis/Lua.
- **Resources:**
  - [Stripe Engineering: How we built rate limiters](https://stripe.com/blog/rate-limiters)
  - [Gaurav Sen: Rate Limiting Algorithms (Video)](https://www.youtube.com/watch?v=mY2X30oN8Ew)

### [4] Web Crawler
- **Key Concepts:** Distributed queues, robots.txt politeness, URL frontier management.
- **Resources:**
  - [System Design Primer: Web Crawler (GitHub)](https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/web_crawler/README.md)
  - [Alex Xu (Vol 1) - Chapter 9](https://bytebytego.com/courses/system-design-interview/design-a-web-crawler)

---

## 2. Messaging & Real-Time Systems
Crucial for Node.js/Full-Stack roles focusing on concurrency and low latency.

### [5] WhatsApp / Messenger
- **Key Concepts:** WebSocket connection management, push notifications, message sequencing.
- **Resources:**
  - [Arpit Bhayani: Discord's 12M+ Connections (Video)](https://www.youtube.com/watch?v=6w6E6p_m0_8)
  - [WhatsApp Engineering: Scaling to Billions](https://blog.whatsapp.com/)

### [6] Notification System
- **Key Concepts:** Priority queues, idempotency (duplicate prevention), 3rd-party integration.
- **Resources:**
  - [Uber Engineering: Real-time Notifications](https://www.uber.com/en-IN/blog/migrating-uber-notifications-to-push-platform/)
  - [Alex Xu (Vol 1) - Chapter 10](https://bytebytego.com/courses/system-design-interview/design-a-notification-system)

### [7] Real-time Leaderboard
- **Key Concepts:** Redis Sorted Sets (ZSETs), handling massive frequency of updates.
- **Resources:**
  - [Redis: Building Real-time Leaderboards](https://redis.io/solutions/leaderboards/)
  - [Roadmap.sh: Leaderboard Project](https://roadmap.sh/projects/realtime-leaderboard-system)

---

## 3. High-Traffic Marketplaces
Tests your ability to handle strict data consistency and "Thundering Herd" problems.

### [8] Ride-Sharing (Uber/Lyft)
- **Key Concepts:** Geospatial indexing (H3, Quadtrees, S2), matching algorithms.
- **Resources:**
  - [Uber Engineering: H3 Spatial Index](https://www.uber.com/en-IN/blog/h3-ubers-hexagonal-hierarchical-spatial-index/)
  - [Hello Interview: Design Uber](https://www.hellointerview.com/learn/system-design/problem-breakdowns/uber)

### [9] Ticket Booking (BookMyShow)
- **Key Concepts:** Distributed locking (Redlock), transactional integrity, flash sale handling.
- **Resources:**
  - [InterviewReady: Designing a Ticketing System](https://interviewready.io/blog/ticket-booking-system-design)
  - [GeeksforGeeks: Design BookMyShow](https://www.geeksforgeeks.org/system-design-bookmyshow/)

### [10] E-commerce Checkout (Amazon)
- **Key Concepts:** Saga Pattern (distributed transactions), inventory reservation, eventual consistency.
- **Resources:**
  - [Microsoft: Saga Design Pattern (Azure Architecture)](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga)
  - [ScholarHat: Saga in Microservices](https://www.scholarhat.com/tutorial/designpatterns/saga-design-pattern-microservices-guide)

---

## 4. Media & Social Analytics
Focuses on data ingestion pipelines and content delivery.

### [11] Instagram / Twitter News Feed
- **Key Concepts:** Fan-out on Write vs. Fan-out on Load, hybrid models for celebrities.
- **Resources:**
  - [Twitter Engineering: Timeline Infrastructure](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-timelines)
  - [DDIA Book - Chapter 1: Twitter Case Study](https://dataintensive.net/)

### [12] YouTube / Netflix
- **Key Concepts:** CDN strategy, video transcoding (DAGs), Adaptive Bitrate Streaming (HLS).
- **Resources:**
  - [Netflix Tech Blog: High Quality Encoding](https://netflixtechblog.com/high-quality-video-encoding-at-scale-d1b32f913e11)
  - [ByteByteGo: Design YouTube](https://bytebytego.com/courses/system-design-interview/design-youtube)

### [13] Ad-Click Aggregator
- **Key Concepts:** Stream processing (Flink/Kafka Streams), Exactly-once semantics, watermarking.
- **Resources:**
  - [Meta/Facebook: Ad Click Aggregation Design](https://medium.com/@bugfreeai/designing-an-ad-click-aggregation-system-meta-senior-engineer-system-design-interview-guide-18db8a974c3b)
  - [Grokking System Design: Click Aggregator](https://grokkingthesystemdesign.com/guides/ad-click-aggregator-system-design/)

---

## 5. Infrastructure & Storage
Deep-dives into how the cloud actually works.

### [14] Log Aggregation (ELK Stack)
- **Key Concepts:** Backpressure (Kafka), Indexing vs. Search, storage tiering (Hot/Cold).
- **Resources:**
  - [DesignGurus: Log Aggregation from Millions of Servers](https://www.designgurus.io/answers/detail/how-would-you-design-a-log-aggregation-system-that-collects-and-indexes-logs-from-millions-of-servers)
  - [Dev.to: Scalable ELK Infrastructure](https://dev.to/chaira/building-a-scalable-secure-elk-stack-infrastructure-a-practical-guide-37hb)
  - [Designing a Scalable Centralized Logging System](https://medium.com/@rahulgargblog/designing-a-scalable-centralized-logging-system-f99172c2e89b)

### [15] Distributed Task Scheduler (Airflow)
- **Key Concepts:** High Availability of scheduler, avoiding double execution, retry mechanisms.
- **Resources:**
  - [System Design Handbook: Job Scheduler](https://www.systemdesignhandbook.com/guides/design-a-distributed-job-scheduler/)
  - [Apache Airflow: How the Scheduler Works](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/scheduler.html)

### [16] Key-Value Store (Dynamo Style)
- **Key Concepts:** Quorum (N, W, R), Vector Clocks, Merkle Trees for anti-entropy.
- **Resources:**
  - [ByteByteGo: Design a KV Store](https://bytebytego.com/courses/system-design-interview/design-a-key-value-store)
  - [DDIA Book - Chapter 3: Storage and Retrieval](https://dataintensive.net/)

---

## 💡 The "Azure" Mapping for Senior Interviews
*Use these services when describing your design to show platform expertise.*

| Generic Component | Azure Service |
| :--- | :--- |
| **Message Broker** | Event Hubs / Service Bus |
| **Global Database** | Cosmos DB |
| **Compute / Worker** | Azure Functions / AKS |
| **Storage** | Blob Storage |
| **Caching** | Azure Cache for Redis |
