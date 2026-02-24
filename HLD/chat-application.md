### 1. Core Architecture & Connections

* **Persistent Connections:** Clients maintain active WebSocket connections with stateful **WebSocket Servers** for real-time, bi-directional communication.
* **Stateless APIs:** Standard HTTP servers handle non-real-time tasks like login, profile updates, and fetching historical data.

### 2. Web Socket Servers & Chat Servers
In a highly scalable chat architecture, separating the connection layer from the business logic layer is crucial for system stability and seamless deployments. 
* **WebSocket Servers** act as stateful, memory-optimized gateways whose sole responsibility is to maintain hundreds of thousands of persistent TCP connections, manage heartbeats, and pass along raw data packets without needing to understand the payload. 
* **Chat Servers** are the stateless, CPU-optimized "brains" of the operation that handle all the heavy lifting: validating user permissions, executing database writes, and routing payloads to the correct message broker channels. This clear separation of concerns ensures that backend teams can rapidly deploy new code and restart the stateless Chat Servers dozens of times a day without ever severing the active user connections held safely by the WebSocket layer.


### 2. Message Routing & The Fan-Out Problem

To route a single message to multiple users (e.g., in a group chat), we use a **Message Broker** (Kafka, RabbitMQ, or Redis Pub/Sub) combined with a central Chat Service.

* **Server-Centric Channels:** To prevent overwhelming the message broker, we do *not* create one channel per user. Instead, each WebSocket server subscribes to exactly **one** channel for itself.
* **The Flow:** The Chat Service looks up the recipient's current WebSocket server, wraps the message in a targeted payload, and drops it into that server's specific channel. The server then pushes the message down the correct local WebSocket connection.

### 3. State Synchronization (Catching Up)

When a user reconnects after being offline, they need to fetch missed messages without overloading the database.

* **Per-Chat Watermarks (Standard):** The client sends a dictionary of its chat IDs and the last `message_id` it saw for each. The server queries the database for newer messages in those specific chats.
* **Sync Token / User Inbox (Alternative):** Every user has a personal event timeline. The client sends a single "Sync Token," and the server returns all events (messages, read receipts, name changes) that happened since that token.

### 4. The Presence Service (Online Status)

This is the "switchboard" that tracks who is online and where they are connected.

* **Redis-Backed:** Uses Redis for ultra-fast key-value lookups (Key: `user_id` -> Value: `websocket_server_id`).
* **Heartbeats & TTL:** Mobile clients send periodic "pings" to keep their status active. Redis uses a **Time-To-Live (TTL)** expiration. If a server crashes or a phone loses signal without sending a disconnect event, the TTL expires and safely marks the user as offline.

### 5. Presence Broadcasting (The $N^2$ Problem)

Notifying a user's entire friends list about status changes is highly resource-intensive. We optimized this using three techniques:

* **Lazy Loading:** When opening the app, clients fetch the status of *only* the friends currently visible on their screen.
* **Targeted Push:** When User A comes online, a background worker checks Redis to see which of User A's friends are *currently online*, and only pushes the update to those specific active connections.
* **Debouncing:** To handle flaky cellular networks, the system waits for a buffer period (e.g., 30 seconds) before broadcasting an "offline" status, preventing the UI from flashing.

### 6. Message Storage (NoSQL Schema)

We use a **Wide-Column NoSQL Database** (like Cassandra or DynamoDB) optimized for massive write loads and sequential reads.

* **Partition Key (`chat_id`):** Forces all messages for a specific group or 1-on-1 chat to live on the exact same physical server, making reads incredibly fast.
* **Clustering Key (`message_id`):** Uses a TimeUUID or Snowflake ID. This embeds the timestamp directly into the ID, ensuring the database physically stores messages in strict chronological order.
