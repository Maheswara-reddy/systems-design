## Book My Show
### 1. High-Level Architecture & Tech Stack

* **Client / Frontend:** React (SSR/SSG for SEO on movie listings, CSR for the dynamic seat map). Highly optimized for Core Web Vitals (LCP for images, INP for instant seat clicks using Optimistic UI updates).
* **API Gateway:** **Federated GraphQL**. This stitches together the Catalog, Booking, and User subgraphs, preventing over-fetching and allowing the frontend to pull complex relational data in one request.
* **Microservices:** Catalog Service (Read-heavy), Booking Service (Write-heavy), Payment Service, Notification Service.
* **Databases & Infrastructure:** PostgreSQL (ACID transactions), Redis (Caching & Distributed Locks), Elasticsearch (Complex catalog search), Apache Kafka (Asynchronous events).

### 2. Database Schema (The Source of Truth)

The core relational entities stored in PostgreSQL:

* `City`, `Theater`, `Screen`, `Movie`, `Show`
* `Seat`: The physical seat in the room.
* **`ShowSeat` (The Critical Table):** Maps a physical seat to a specific show (`id`, `show_id`, `seat_id`, `status: AVAILABLE/LOCKED/BOOKED`).
* `Booking`: The final transaction record.

### 3. The Write Path (Booking & Concurrency)

Handling the "Double Booking" problem is the most critical part of this design. We use a two-layered locking strategy:

* **Layer 1: Temporary Lock (Redis).** When seats are selected, we execute a **Redis Lua Script** to atomically place a 10-minute TTL lock on all requested seats simultaneously. If even one seat is taken, the whole script fails, preventing partial locks.
* **Layer 2: Permanent Lock (PostgreSQL).** When payment succeeds, we use **Pessimistic Locking** (`SELECT ... FOR UPDATE`). To prevent **deadlocks** when multiple users book multiple seats concurrently, the backend always sorts the seat IDs (`ORDER BY id`) before executing the database lock.

### 4. The Read Path (Browsing & Scaling)

Handling the "Thundering Herd" when a blockbuster opens:

* **Caching Strategy:** * *Aggressive Cache (CDN/Redis):* Movie posters, theater master data, and static physical seat layouts.
* *Micro-Cache (15-30s TTL):* High-level show availability ("Filling Fast" badges).
* *Strictly Uncacheable:* The real-time seat matrix for a specific show must never be cached with a TTL.


* **Database Protection:** We use a **Cache-Aside pattern with distributed locking**. If the cache misses, only one thread is allowed to query PostgreSQL, while the rest wait a few milliseconds for that single thread to repopulate Redis.

### 5. Real-Time UI Updates (WebSockets)

To prevent users from clicking seats that were just taken:

* Users viewing a specific seat map establish a WebSocket connection (or GraphQL Subscription).
* The connection is tracked in the **local RAM** of the WebSocket server (using an $O(1)$ Hash Map of `show_id` -> `[Connections]`).
* When a seat is locked, the Booking Service fires a message to **Redis Pub/Sub**.
* The WebSocket servers subscribed to that channel instantly broadcast the change to the connected clients, updating the UI in milliseconds without polling.

### 6. Resiliency & Edge Cases (The Saga Pattern)

Handling payment gateway timeouts (User pays, but the 10-minute Redis lock expires before our system registers it, and someone else steals the seat):

* The system heavily relies on **Apache Kafka** for event-driven consistency.
* When the delayed payment webhook arrives, it's processed idempotently and a `PaymentCompleted` event is published.
* The Booking Service consumes this. If it tries to lock the database and sees the seats are now `BOOKED` by someone else, it executes a `ROLLBACK`.
* It then publishes a `BookingFailed_InitiateRefund` event, which a separate Refund Service picks up to reverse the charge, ensuring financial consistency without tying up the core database threads.

### 7. Sharding & Security

* **Database Sharding:** When PostgreSQL gets too large, we shard by **`show_id`** (e.g., `hash(show_id) % N`). This ensures all seats for a specific show live on the same physical server, keeping our pessimistic locks fast and local. We use an asynchronous secondary index (NoSQL) to serve queries like "My Booking History" by `user_id`.
* **Rate Limiting:** A Token Bucket algorithm at the API Gateway blocks bot scripts from locking up an entire theater's seats.

---
## Some important Considerations and Concepts

### 1. Frontend Rendering & SEO (The ISR Strategy)

To handle a massive, ever-growing catalog of movies without crashing build times or overloading Node.js servers, we use a hybrid rendering approach.

* **Build Time (The "Hot" Catalog):** We pre-build static HTML pages (SSG) only for currently running and highly anticipated movies, pushing them to a CDN for instant, zero-compute loading.
* **Request Time (The Fallback):** If a user requests an older/obscure movie, the CDN passes the request to the server. The server dynamically renders the page (SSR), serves it to the user, and **caches a copy on the CDN** for future users.
* **Stale-While-Revalidate:** We assign a TTL to cached pages. When data (like reviews) changes, the CDN serves the slightly stale HTML immediately, while pinging the backend to seamlessly regenerate and swap in the fresh HTML in the background.

### 2. High-Level Read vs. Write Paths

Separating these paths is crucial because they have entirely different scaling requirements and failure domains.

* **The Read Path (Browsing):** * Focuses on high availability and low latency via a federated GraphQL API.
* Master data (movie details) is aggressively cached in Redis/CDNs.
* To view real-time seat availability, we use a **Cache-Aside pattern with distributed locks** to prevent a "thundering herd" of database queries.
* Real-time seat updates (someone else selecting a seat) are pushed to the UI via **WebSockets (Redis Pub/Sub)**.


* **The Write Path (Booking):** * Focuses on strict ACID consistency.
* **Layer 1 (Redis):** Uses atomic Lua scripts to place a temporary 10-minute hold on multiple seats simultaneously (preventing partial locks).
* **Layer 2 (PostgreSQL):** Uses pessimistic locking (`SELECT ... FOR UPDATE`) sorted by seat ID to permanently commit the booking and prevent deadlocks during concurrent transactions.



### 3. Payment Flows & Webhook Communication

Because handling PCI-compliant data on our servers is a massive liability, we decouple the payment execution from our backend.

* **The Execution:** The React frontend loads the Payment Gateway's SDK. The user's credit card data goes *directly* from the browser to the Payment Gateway (PG), bypassing our backend entirely.
* **The Webhook (Source of Truth):** Once the PG processes the charge, it securely notifies our backend via a server-to-server HTTP POST webhook.
* **Security & Reliability:**
* **Signatures:** We verify the webhook's HMAC signature to ensure it genuinely came from the PG and wasn't spoofed.
* **Idempotency:** We store the webhook's unique `event_id` in the database to ensure we don't process the same payment twice if the PG retries the request.
* **Decoupling:** The webhook handler returns a `200 OK` instantly and drops the event into **Apache Kafka**. The Booking Service processes it asynchronously to ensure the PG doesn't timeout waiting for our database.



### 4. UI Handling for Payments (The "Anxious User")

When a user's money is deducted, the UI must manage their psychology gracefully while waiting for the asynchronous webhook to hit our backend.

* **Seconds 0–10 (Short Polling/WebSockets):** The UI displays a standard "Processing your payment..." spinner while checking the backend for a `CONFIRMED` status.
* **Seconds 10–30 (Exponential Backoff):** Polling slows down to protect the backend. The UI updates to reassure the user: *"Waiting for bank confirmation... Please do not refresh."*
* **30+ Seconds (Timeout):** We stop polling and transition to a "Payment Pending" screen. We assure the user their money is safe and give them an exit route to their dashboard.
* **The Safety Net:** Even if the user closes the tab in frustration, our decoupled Kafka-based backend will eventually process the delayed webhook, lock the seats, and email them the tickets without requiring the frontend to be active.
