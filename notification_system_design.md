Stage 1

1. Core Actions Supported by the Platform
To provide a complete user notification experience upon logging in, the platform supports the following operations:
**Fetch Notifications:** Retrieves unread and read notifications for the authenticated user.
 **Mark as Read/Archive:** Allows the client to clear notifications once viewed.
**Preferences Management:** Allows users to opt-in or opt-out of specific channels (Email, Push, SMS).
 **Real-time Stream:** Sustains an open channel to push live notifications to the UI instantly without polling.

---

2. REST API Design Contract 

 A. Fetch User Notifications
* **Endpoint:** `GET /v1/notifications`
* **Description:** Retrieves a paginated list of notifications for the logged-in user.
* **Headers:**
    ```http
    Authorization: Bearer <JWT_ACCESS_TOKEN>
    Accept: application/json
    ```
* **Query Parameters:**
    * `status`: (Optional) Filter by `read` or `unread`.
    * `limit`: (Optional, default=20) For pagination chunking.
    * `page`: (Optional, default=1) Current page context.

* **Response Payload (`200 OK`):**
    ```json
    {
      "success": true,
      "data": {
        "notifications": [
          {
            "id": "notif_883f9a12-9c10-4b53",
            "title": "Security Alert",
            "message": "A new login was detected from a new Chrome instance.",
            "type": "SECURITY",
            "priority": "HIGH",
            "isRead": false,
            "createdAt": "2026-06-03T08:30:00Z"
          },
          {
            "id": "notif_112a7d45-3e89-1a22",
            "title": "Welcome Pack Loaded",
            "message": "Your profile verification is fully complete.",
            "type": "SYSTEM",
            "priority": "LOW",
            "isRead": true,
            "createdAt": "2026-06-03T07:15:00Z"
          }
        ],
        "pagination": {
          "totalCount": 2,
          "totalPages": 1,
          "currentPage": 1,
          "limit": 20
        }
      }
    }
    ```

---

### B. Mark Notification as Read
* **Endpoint:** `PATCH /v1/notifications/:id/read`
* **Description:** Updates the status of a specific notification instance to prevent double-displaying.
* **Headers:**
    ```http
    Authorization: Bearer <JWT_ACCESS_TOKEN>
    Content-Type: application/json
    ```
* **Response Payload (`200 OK`):**
    ```json
    {
      "success": true,
      "message": "Notification successfully updated to read state.",
      "updatedId": "notif_883f9a12-9c10-4b53"
    }
    ```
* **Error Response (`404 Not Found`):**
    ```json
    {
      "success": false,
      "error": "RESOURCE_NOT_FOUND",
      "message": "The notification ID specified does not exist or belong to this user account."
    }
    ```

---

### C. Update Notification Preferences
* **Endpoint:** `PUT /v1/notifications/preferences`
* **Description:** Configures user delivery preferences.
* **Headers:**
    ```http
    Authorization: Bearer <JWT_ACCESS_TOKEN>
    Content-Type: application/json
    ```
* **Request Body JSON Schema:**
    ```json
    {
      "preferences": {
        "marketingChannels": {
          "email": false,
          "push": true,
          "sms": false
        },
        "securityChannels": {
          "email": true,
          "push": true,
          "sms": true
        }
      }
    }
    ```
* **Response Payload (`200 OK`):**
    ```json
    {
      "success": true,
      "message": "User communication preferences saved successfully."
    }
    ```

---

## 3. Real-Time Notification Architecture Design

To drop notifications onto the user's screen instantly without constantly hitting our database with heavy HTTP polling loops, the platform implements **Server-Sent Events (SSE)**.

### Why Server-Sent Events (SSE) over WebSockets?
1. **Unidirectional Simplicity:** Notifications only flow *one way* (from the server down to the logged-in client UI). WebSockets offer bidirectional streaming, which introduces unnecessary overhead.
2. **Native HTTP Protocol Compatibility:** SSE operates directly over standard HTTP/1.1 or HTTP/2 transport pipes using standard text streaming headers. It bypasses corporate firewalls effortlessly and includes **built-in automatic reconnection handling** handled natively by the browser's `EventSource` API.

### Stream Implementation Spec
* **Endpoint:** `GET /v1/notifications/stream`
* **Connection Response Headers Required:**
    ```http
    Content-Type: text/event-stream
    Cache-Control: no-cache
    Connection: keep-alive
    ```
* **Real-time Event Wire Layout Example:**
    ```text
    event: new_notification
    data: {"id": "notif_999", "title": "Live Update", "message": "Your request status shifted to Active."}
    ```
    # Stage 2

## 1. Persistent Storage Recommendation & Justification

For an enterprise-scale notification platform, a hybrid storage approach or a high-performance **NoSQL Document Store (like MongoDB)** or **Time-Series / Wide-Column Store (like Cassandra)** is highly ideal. However, since notification system access patterns map directly to core relational constraints (such as users mapping to explicit user records), using a **Relational Database Management System (RDBMS) like PostgreSQL** paired with a caching layer (Redis) offers the highest level of reliability.

### Why PostgreSQL with Redis is selected:
* **ACID Compliance:** Transaction integrity guarantees that marking a critical notification as "read" updates across all consumer platforms instantly without stale reads or data race conditions.
* **Efficient Indexing Structures:** High volume reads (filtering by `user_id` and `is_read`) can be optimized to sub-millisecond speeds using structural B-Tree and partial indexes.
* **JSONB Support:** Offers the flexibility of NoSQL by allowing dynamic payload properties to be stored cleanly inside binary JSON columns, adapting smoothly if notification metadata attributes change.

---

## 2. Relational Database Schema Design (SQL)

Here is the physical database schema written in PostgreSQL syntax. It includes a `users` table, a `notification_preferences` table, and a partitioned-ready `notifications` core tracking log table.

```sql
-- 1. Users Directory Table
CREATE TABLE users (
    user_id VARCHAR(64) PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. Notification Main Log Storage Table
CREATE TABLE notifications (
    notification_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    type VARCHAR(32) NOT NULL, -- SECURITY, SYSTEM, MARKETING, TRANSACTIONAL
    priority VARCHAR(16) NOT NULL DEFAULT 'LOW', -- LOW, MEDIUM, HIGH
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. User Channel Preferences Table
CREATE TABLE notification_preferences (
    preference_id SERIAL PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL UNIQUE REFERENCES users(user_id) ON DELETE CASCADE,
    marketing_email BOOLEAN NOT NULL DEFAULT TRUE,
    marketing_push BOOLEAN NOT NULL DEFAULT TRUE,
    marketing_sms BOOLEAN NOT NULL DEFAULT FALSE,
    security_email BOOLEAN NOT NULL DEFAULT TRUE,
    security_push BOOLEAN NOT NULL DEFAULT TRUE,
    security_sms BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. High Performance Composite and Partial Indexes for Stage 1 Access Patterns
CREATE INDEX idx_notifications_user_unread 
ON notifications(user_id) 
WHERE is_read = FALSE;

CREATE INDEX idx_notifications_created_at 
ON notifications(created_at DESC);

### 3. Escalating Scale Problems & Solutions

As the user base expands and the volume of notifications reaches millions of rows per week, two significant system bottlenecks occur: 

#### Problem A: Read Amplification & Degraded Query Performance
* **The Issue:** When fetching unread notifications (`GET /v1/notifications?status=unread`), the database engine must scan expanding indexes or partition data spaces. As table size expands, performance shifts from O(1) or O(log N) toward linear scans, causing API responses to break the latency SLA.
* **The Solution:** 1. **Partial Indexing:** Implement conditional indexing (`WHERE is_read = FALSE`) so the index tree size remains tiny, capturing only active unread notification payloads.
  2. **In-Memory Read Cache (Redis):** Cache a counter or full array of unread notification fragments in memory. When a new notification arrives, push it to Redis. The API reads from Redis first, bypassing the disk database entirely for active feed updates.

#### Problem B: Exploding Storage Footprint & Slow Disk I/O
* **The Issue:** Read/Write operations on a single massive table degrade as old, stale notification logs consume space in active database memory blocks.
* **The Solution:**
  1. **Horizontal Database Partitioning:** Partition the `notifications` table by time ranges (e.g., monthly partitions). Active reads and writes target the current months partition table, while historical read logs sit safely on distinct historical storage partitions.
  2. **Data Retention & Archival Policies:** Implement a worker daemon that offloads read notifications older than 30 days to cold storage objects (like AWS S3 or cheap analytical databases) and deletes them from the main active operation table.


  These are the exact queries that bind your database engine to the REST contract endpoints built in Stage 1:

  Query A: Fetch Unread Notifications (For GET /v1/notifications?status=unread)

  Uses explicit pagination boundaries (LIMIT and OFFSET) to safeguard system memory profiles:

  SELECT notification_id, title, message, type, priority, is_read, created_at
FROM notifications
WHERE user_id = 'usr_test_12345' 
  AND is_read = FALSE
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

Query B: Mark Notification as Read (For PATCH /v1/notifications/:id/read)
Updates state cleanly while validating that the resource belongs strictly to the requesting authenticated user:

UPDATE notifications
SET is_read = TRUE
WHERE notification_id = 'notif_883f9a12-9c10-4b53' 
  AND user_id = 'usr_test_12345'
RETURNING notification_id;

Query C: Update Preferences (For PUT /v1/notifications/preferences)
Uses an atomic upsert routine (ON CONFLICT) to safely write new or modify existing database row definitions:

INSERT INTO notification_preferences (
    user_id, marketing_email, marketing_push, marketing_sms, security_email, security_push, security_sms, updated_at
) 
VALUES ('usr_test_12345', FALSE, TRUE, FALSE, TRUE, TRUE, TRUE, CURRENT_TIMESTAMP)
ON CONFLICT (user_id) 
DO UPDATE SET 
    marketing_email = EXCLUDED.marketing_email,
    marketing_push = EXCLUDED.marketing_push,
    marketing_sms = EXCLUDED.marketing_sms,
    security_email = EXCLUDED.security_email,
    security_push = EXCLUDED.security_push,
    security_sms = EXCLUDED.security_sms,
    updated_at = CURRENT_TIMESTAMP;

    ---

Stage 3

## 1. Analysis of the Existing Query

### Is this query accurate?
**Yes, structurally.** The query is syntactically sound and correctly filters by the target student and unread status while ordering by time. However, architectural flaws make it unusable at scale.

### Why is this query performing slowly?
1. **Full Table Scan (High I/O Cost):** With 5,000,000 rows and no dedicated indexes on `studentID` or `isRead`, the database engine is forced to scan every single data page on disk from row 1 to 5,000,000 to find matches.
2. **Heavy Sorting Overhead:** The `ORDER BY createdAt DESC` clause forces the database to sort the filtered records in memory or via temporary disk files. Without an index to provide pre-sorted data, this operation stalls completely at 5,000,000 records.
3. **The `SELECT *` Antipattern:** Pulling all columns indiscriminately blocks the database from utilizing lean index-only scans, unnecessarily inflating network payload size and disk read buffers.

---

## 2. Structural Mismatch & Indexing Evaluation

### Evaluation of Advice: "Add indexes on every single column"
**No, this advice is highly counterproductive and dangerous for a notification system.**

* **The Problem with Over-Indexing:** While adding individual indexes on every column speeds up specific single-column read searches, it severely degrades **Write Performance**. Every single time a new notification is sent (`INSERT`) or marked as read (`UPDATE`), the database engine must simultaneously modify every single index tree across the board.
* **Low Cardinality Pitfall:** Indexing boolean fields like `isRead` individually is ineffective. Booleans have low cardinality (only two possible values: true or false), meaning the database engine will likely ignore the index entirely and default back to a full table scan.

### What should be changed instead?
Instead of individual single-column indexes, a single **Composite (Multi-Column) Index** or a **Partial Index** must be created to target the exact query blueprint.

sql
-- Optimal Solution: A Composite B-Tree Index covering the search criteria
CREATE INDEX idx_notifications_student_unread 
ON notifications(studentID, isRead, createdAt DESC);]

Impact on Computation Cost:Before Indexing: Computational cost is $O(N)$ where $O(5,000,000)$ operations are performed for every single API fetch.After Composite Indexing: Computational cost drops exponentially to $O(\log N)$ for the lookup, and $O(1)$ for retrieval since the index keeps data pre-sorted by createdAt DESC

## 3 Advanced Analytical Query (Last 7 Days Placement Fetch)
To fetch all unique student records who received a "Placement" notification type within the trailing 7 days, we use relative interval time computations alongside explicit filtering constraints.

SELECT DISTINCT studentID
FROM notifications
WHERE notificationType = 'Placement'
  AND createdAt >= CURRENT_TIMESTAMP - INTERVAL '7 days'
ORDER BY studentID ASC;

DISTINCT studentID: Ensures each student is returned exactly once, preventing duplications if a single student received multiple placement updates.

Dynamic Date Range Handler: Using CURRENT_TIMESTAMP - INTERVAL '7 days' computes ranges dynamically on execution, allowing the database query planner to leverage any existing time-series indexes on the createdAt column efficiently.


"Stage 4 High-Availability and Caching Strategies"

## 1. System Diagnosis
Fetching notification records from the primary database on every single page load creates an anti-pattern known as a **Read-Heavy Bottleneck**. As active user concurrent sessions scale, the database connection pool starves, disk I/O hits 100% saturation, and read queries stall—causing cascading API latencies across the entire application workspace.

---

## 2. Proposed Architectural Solutions

To alleviate primary database pressure and improve user experience, we evaluate three major strategies:

### Strategy A: Cache-Aside Pattern with an In-Memory Database (Redis)
Introduce an in-memory key-value cache store between the application backend application server and the database.
* **Mechanism:** When a student loads a page, the application server checks Redis first (`Key: student:notif:1042`). If the data is present (Cache Hit), it returns immediately. If missing (Cache Miss), it reads from the primary SQL database, stores the result back in Redis with a Time-To-Live (TTL) expiration, and serves the user.
* **Trade-offs:**
    * **Pros:** Blazing-fast response times (< 2ms lookup); drops DB read operations by up to 90%.
    * **Cons:** Introduces cash invalidation complexities (when a new notification is added, the code must actively clear or update the Redis key to prevent stale data).

### Strategy B: Debounce or Local Storage Debouncing (Client-Side Caching)
Settle short-term state properties inside the browser application shell using `localStorage` or `sessionStorage`.
* **Mechanism:** When a user switches pages internally within the application web client, look at the last update timestamp. If less than 60 seconds have passed, skip making an HTTP network request entirely and read the previous notification state payload from client memory.
* **Trade-offs:**
    * **Pros:** Zero network latency or execution costs for rapid internal page clicks; drastically lowers total network requests striking the API gateway.
    * **Cons:** If an urgent security alert or instant critical notification triggers, the student might experience a small delivery delay until the local client cache interval lapses.

### Strategy C: Event-Driven Push Synchronization (SSE/WebSockets Retention)
Transition from active polling/fetching to maintaining an active, memory-driven communication pipeline.
* **Mechanism:** Establish a Server-Sent Events (SSE) socket channel on login. The application server retains active state in temporary server memory arrays. Instead of querying the database on every page mount, the database is only queried once on first login; subsequent alerts are pushed directly down the stream socket.
* **Trade-offs:**
    * **Pros:** Eliminates recurring page-load query traffic entirely.
    * **Cons:** Increases server memory utilization profiles due to maintaining continuous connection ports open for thousands of parallel users.

---

## 3. Comparative Architecture Assessment Matrix

| Metrics & Considerations | Strategy A: Redis Cache-Aside | Strategy B: Client-Side Caching | Strategy C: SSE Push Pipe |
| :--- | :--- | :--- | :--- |
| **Primary DB Relief** | **Excellent** (Intercepts majority of reads) | **Moderate** (Relies on user behavior models) | **Exceptional** (Reads once upon loading) |
| **Data Freshness** | High (When matched with explicit eviction) | Delayed (Bound to local interval timers) | **Instant** (Real-time distribution stream) |
| **Infrastructure Overhead** | High (Requires managing a Redis instance) | **None** (Handled natively by browser) | Moderate (Requires persistent state ports) |

---

## 4. Implementation Verdict
The ideal corporate production rollout combines **Strategy A (Redis Caching)** alongside the **SSE Pattern** designed in Stage 1. This ensures that even if a socket drops and forces a client-side layout reload, the incoming request terminates safely inside an in-memory Redis cache node, keeping your primary database footprint safe, steady, and insulated.

