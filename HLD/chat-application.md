### 1. Core Architecture & Connections

* **Persistent Connections:** Clients maintain active WebSocket connections with stateful **WebSocket Servers** for real-time, bi-directional communication.
* **Stateless APIs:** Standard HTTP servers handle non-real-time tasks like login, profile updates, and fetching historical data.

### 2. Web Socket Servers & Chat Servers
In a highly scalable chat architecture, separating the connection layer from the business logic layer is crucial for system stability and seamless deployments. 
* **WebSocket Servers** act as stateful, memory-optimized gateways whose sole responsibility is to maintain hundreds of thousands of persistent TCP connections, manage heartbeats, and pass along raw data packets without needing to understand the payload. 
* **Chat Servers** are the stateless, CPU-optimized "brains" of the operation that handle all the heavy lifting: validating user permissions, executing database writes, and routing payloads to the correct message broker channels. This clear separation of concerns ensures that backend teams can rapidly deploy new code and restart the stateless Chat Servers dozens of times a day without ever severing the active user connections held safely by the WebSocket layer.


### 3. Message Routing & The Fan-Out Problem

To route a single message to multiple users (e.g., in a group chat), we use a **Message Broker** (Kafka, RabbitMQ, or Redis Pub/Sub) combined with a central Chat Service.

* **Server-Centric Channels:** To prevent overwhelming the message broker, we do *not* create one channel per user. Instead, each WebSocket server subscribes to exactly **one** channel for itself.
* **The Flow:** The Chat Service looks up the recipient's current WebSocket server, wraps the message in a targeted payload, and drops it into that server's specific channel. The server then pushes the message down the correct local WebSocket connection.

### 4. State Synchronization (Catching Up)

When a user reconnects after being offline, they need to fetch missed messages without overloading the database.

* **Per-Chat Watermarks (Standard):** The client sends a dictionary of its chat IDs and the last `message_id` it saw for each. The server queries the database for newer messages in those specific chats.
* **Sync Token / User Inbox (Alternative):** Every user has a personal event timeline. The client sends a single "Sync Token," and the server returns all events (messages, read receipts, name changes) that happened since that token.

### 5. The Presence Service (Online Status)

This is the "switchboard" that tracks who is online and where they are connected.

* **Redis-Backed:** Uses Redis for ultra-fast key-value lookups (Key: `user_id` -> Value: `websocket_server_id`).
* **Heartbeats & TTL:** Mobile clients send periodic "pings" to keep their status active. Redis uses a **Time-To-Live (TTL)** expiration. If a server crashes or a phone loses signal without sending a disconnect event, the TTL expires and safely marks the user as offline.

### 6. Presence Broadcasting (The $N^2$ Problem)

Notifying a user's entire friends list about status changes is highly resource-intensive. We optimized this using three techniques:

* **Lazy Loading:** When opening the app, clients fetch the status of *only* the friends currently visible on their screen.
* **Targeted Push:** When User A comes online, a background worker checks Redis to see which of User A's friends are *currently online*, and only pushes the update to those specific active connections.
* **Debouncing:** To handle flaky cellular networks, the system waits for a buffer period (e.g., 30 seconds) before broadcasting an "offline" status, preventing the UI from flashing.

### 7. Message Storage (NoSQL Schema)

We use a **Wide-Column NoSQL Database** (like Cassandra or DynamoDB) optimized for massive write loads and sequential reads.

* **Partition Key (`chat_id`):** Forces all messages for a specific group or 1-on-1 chat to live on the exact same physical server, making reads incredibly fast.
* **Clustering Key (`message_id`):** Uses a TimeUUID or Snowflake ID. This embeds the timestamp directly into the ID, ensuring the database physically stores messages in strict chronological order.

### 8. The Message Lifeycle
#### 1. The Ticks

The tick system is entirely driven by **Acknowledgements (ACKs)** sent between the client devices and the server.

* **Clock Icon / Faded Text (Pending):** The message only exists in the sender's local database (SQLite). The app is actively trying to push it over the WebSocket to the server.
* **Single Grey Tick (Sent/Persisted):** The Chat Service received the message, successfully wrote it to the Cassandra database, and fired a "Server ACK" back to the sender.
* **Double Grey Tick (Delivered):** The recipient's phone received the payload from its WebSocket server, saved it locally, and fired a "Delivery ACK" back to the server. The server routed this ACK to the sender.
* **Double Blue Tick (Read):** The message was actually rendered on the recipient's screen. The recipient's phone fires a `read_receipt` event.
* *Crucial DB Rule:* The server does **not** update the original message in Cassandra to "read" (which creates bad tombstones). Instead, it updates a separate, highly efficient **Read Cursor / Watermark Table** with the `last_read_message_id`.



#### 2. Network Optimizations (Solving "Chattiness")

Because firing a network request for every single "delivered" and "read" event would overwhelm the WebSocket servers and drain phone batteries, we apply three critical client-side optimizations:

* **Cumulative ACKs (High-Water Marks):** We don't acknowledge every message. If a phone receives 5 messages at once, it sends a single ACK for the *latest* message. The server inherently knows all previous messages were also delivered.
* **Batching & Debouncing:** When a user scrolls rapidly through a busy chat, the app doesn't fire a read receipt for every message that flashes on screen. It runs a short timer (e.g., 500ms) and only sends one read receipt for the last visible message when the user stops scrolling.
* **Piggybacking:** If a user reads a message and immediately starts typing, the app doesn't send two separate WebSocket frames. It attaches (piggybacks) the read receipt ACK directly onto the "user is typing" payload, achieving two state updates with one network request.
