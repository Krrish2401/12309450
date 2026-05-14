# Stage 1

## Campus Notification Platform — REST API Design

### Core Actions

| Action | Description |
|---|---|
| Create Notification | Admin/system sends a notification to one or more students |
| Fetch All Notifications | Student retrieves notifications with optional filters |
| Fetch Single Notification | Retrieve one notification by ID |
| Mark as Read | Student marks a specific notification as read |
| Mark All as Read | Student marks all their notifications as read |
| Get Unread Count | Fetch count of unread notifications |
| Delete Notification | Remove a notification |
| Stream Notifications | Real-time delivery of new notifications to connected clients |

---

### Base URL

/api/v1
---

### Common Request Headers

| Header | Value | Required |
|---|---|---|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes (for POST/PATCH) |
| `Accept` | `application/json` | Recommended |

---

### Authorization and Access Rules

- Student-scoped endpoints require the access token subject to match `:studentId`.
- Admin/system tokens can access any student and create broadcast notifications.

### Notification Schema

```json
{
  "id": "uuid-v4",
  "type": "Placement | Event | Result",
  "message": "string",
  "isRead": false,
  "studentId": "string",
  "timestamp": "2026-05-14T10:00:00Z",
  "createdAt": "2026-05-14T10:00:00Z",
  "updatedAt": "2026-05-14T10:00:00Z"
}
```

---

### Endpoints

---

#### 1. Create Notification

POST /api/v1/notifications

**Request Headers:**
Authorization: Bearer <token>

**Request Body:**
```json
{
  "type": "Placement",
  "message": "Campus drive by Google on 28th May 2026",
  "studentIds": ["student-uuid-1", "student-uuid-2"],
  "broadcast": false
}
```

> Set `broadcast: true` to send to all students. `studentIds` is ignored when broadcast is true.

**Response — 201 Created:**
```json
{
  "success": true,
  "data": {
    "broadcast": false,
    "studentIds": ["student-uuid-1", "student-uuid-2"],
    "createdCount": 2,
    "notificationIds": [
      "d146095a-0d86-4a34-9e69-3900a14576bc",
      "f7e3a1b1-3f6c-4f04-8d1f-82c1a1f42c8a"
    ]
  }
}
```

**Error — 400 Bad Request:**
```json
{
  "success": false,
  "error": "Invalid notification type. Must be one of: Placement, Event, Result"
}
```

---

#### 2. Get Notifications for a Student

GET /api/v1/students/:studentId/notifications

**Request Headers:**
Authorization: Bearer <token>

**Query Parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `type` | string | — | Filter by `Placement`, `Event`, or `Result` |
| `isRead` | boolean | — | Filter by read status |
| `page` | number | 1 | Pagination page |
| `limit` | number | 20 | Results per page (max: 100) |
| `sortBy` | string | `timestamp` | Sort field |
| `order` | string | `desc` | `asc` or `desc` |

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "id": "d146095a-0d86-4a34-9e69-3900a14576bc",
        "type": "Result",
        "message": "mid-sem",
        "timestamp": "2026-04-22T17:51:30Z",
        "isRead": false,
        "studentId": "student-uuid-1"
      }
    ],
    "pagination": {
      "total": 150,
      "page": 1,
      "limit": 20,
      "totalPages": 8
    }
  }
}
```

---

#### 3. Get Single Notification

GET /api/v1/notifications/:id

**Request Headers:**
Authorization: Bearer <token>

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "id": "d146095a-0d86-4a34-9e69-3900a14576bc",
    "type": "Placement",
    "message": "Campus drive by Google on 28th May 2026",
    "timestamp": "2026-05-14T10:00:00Z",
    "isRead": true,
    "studentId": "student-uuid-1"
  }
}
```

**Error — 404 Not Found:**
```json
{
  "success": false,
  "error": "Notification not found"
}
```

---

#### 4. Mark Notification as Read

PATCH /api/v1/notifications/:id/read

**Request Headers:**
Authorization: Bearer <token>

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "id": "d146095a-0d86-4a34-9e69-3900a14576bc",
    "isRead": true,
    "updatedAt": "2026-05-14T10:05:00Z"
  }
}
```

---

#### 5. Mark All Notifications as Read

PATCH /api/v1/students/:studentId/notifications/read-all

**Request Headers:**
Authorization: Bearer <token>

**Response — 200 OK:**
```json
{
  "success": true,
  "message": "All notifications marked as read",
  "data": {
    "updatedCount": 12
  }
}
```

---

#### 6. Get Unread Count

GET /api/v1/students/:studentId/notifications/unread-count

**Request Headers:**
Authorization: Bearer <token>

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "unreadCount": 5
  }
}
```

---

#### 7. Delete Notification

DELETE /api/v1/notifications/:id

**Request Headers:**
Authorization: Bearer <token>

**Response — 200 OK:**
```json
{
  "success": true,
  "message": "Notification deleted successfully"
}
```

---

### Real-Time Notification Mechanism

**Chosen Approach: Server-Sent Events (SSE)**

#### Why SSE over WebSockets?

| Factor | SSE | WebSockets |
|---|---|---|
| Direction | Server to client only, fits notifications | Two-way, best for chat or live collaboration |
| Protocol | Standard HTTP stream | HTTP upgrade and separate protocol |
| Reconnection | Built-in auto-reconnect | Client must implement reconnect |
| Infra fit | Works with most HTTP proxies; buffering must be disabled | Needs WS-friendly proxies and idle timeouts tuned |
| Payload | Text only (JSON) | Text or binary |

Notifications are a one-way comm from server to client. SSE keeps the implementation simpler and also supports automatic reconnection and works well with typical HTTP infrastructure. WebSockets are better only if the client must send real-time messages back on the same channel (chat, live edits) or if you need binary payloads.

---

#### SSE Endpoint

GET /api/v1/notifications/stream

**Request Headers:**
Authorization: Bearer <token>
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

> The stream is scoped to the authenticated user. For missed messages, clients should call the standard fetch endpoint.

**Response — 200 OK (streaming):**
```Content-Type: text/event-stream
retry: 5000
event: notification
id: 7b9c1b3d-2a7d-4f8a-9f9f-7f3c92f2b111
data: {"id":"d146095a-0d86-4a34-9e69-3900a14576bc","type":"Placement","message":"Google drive tomorrow","timestamp":"2026-05-14T10:00:00Z","isRead":false,"studentId":"student-uuid-1"}
event: notification
id: 7b9c1b3d-2a7d-4f8a-9f9f-7f3c92f2b112
data: {"id":"f7e3a1b1-3f6c-4f04-8d1f-82c1a1f42c8a","type":"Result","message":"Semester results published","timestamp":"2026-05-14T10:05:00Z","isRead":false,"studentId":"student-uuid-1"}
: keep-alive
```

> The server sends a `: keep-alive` comment every 30 seconds to prevent connection timeouts.

---

# Stage 2

## Persistent Storage — Database Design

---

### Database Choice: PostgreSQL

**Recommended DB: PostgreSQL (Relational)**

#### Why?

| Factor | Reason |
|---|---|
| **Structured, relational data** | Students, notifications, and read-status have clear relationships — a relational model maps naturally |
| **ACID compliance** | Critical for correctness: marking a notification as read must never result in a partial or duplicate update |
| **Rich query support** | Filtering by `type`, `isRead`, `studentId`, `timestamp` with pagination is straightforward with SQL |
| **ENUM support** | `notification_type` (Placement, Event, Result) can be made in an ENUM |
| **Mature ecosystem** | Excellent support for indexing, and query optimisation at scale, which can come into play at large scales|
| **UUID support** | Native `uuid` type for primary keys |

> ummmm, I thought of a NoSQL database but the data we have is well structured and the reads or extraction will be performed better in a SQL enviornment.

---

### Database Schema

-- Notification type enum
CREATE TYPE notification_type AS ENUM ('Placement', 'Event', 'Result');

-- Students table
students (
  id         UUID PRIMARY KEY,
  name
  email
  roll_no
  created_at
  updated_at
);

-- Notifications table (stores the notification content once)
notifications (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type
  message
  created_at
  updated_at
);

-- Junction table: maps notifications to students, tracks read status
student_notifications (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id
  notification_id UUID
  is_read
  read_at
  created_at

  CONSTRAINT uq_student_notification UNIQUE (student_id, notification_id)
);


---


### SQL Queries (mapped to Stage 1 APIs)

#### 1. Create a Notification
```sql
INSERT INTO notifications (type, message)
VALUES ($1, $2)
RETURNING id, type, message, created_at;
```

#### 2. Deliver Notification to Specific Students
```sql
INSERT INTO student_notifications (student_id, notification_id)
SELECT unnest($1::uuid[]), $2;
```

#### 3. Broadcast to All Students
```sql
INSERT INTO student_notifications (student_id, notification_id)
SELECT id, $1 FROM students;
```

#### 4. Get Notifications for a Student
```sql
SELECT
  n.id,
  n.type,
  n.message,
  n.created_at  AS timestamp,
  sn.is_read,
  sn.read_at
FROM notifications n
JOIN student_notifications sn ON n.id = sn.notification_id
WHERE sn.student_id = $1
  AND ($2::notification_type IS NULL OR n.type = $2)   -- optional type filter
  AND ($3::boolean          IS NULL OR sn.is_read = $3) -- optional isRead filter
ORDER BY n.created_at DESC
```

#### 5. Get Single Notification
```sql
SELECT
  n.id, n.type, n.message, n.created_at AS timestamp,
  sn.is_read, sn.read_at, sn.student_id
FROM notifications n
JOIN student_notifications sn ON n.id = sn.notification_id
WHERE n.id = $1
  AND sn.student_id = $2;
```

---

### Problems as Data Volume Increases

At scale (50,000 students, 5,000,000 notifications), `student_notifications` can grow to **billions of rows**. The following problems emerge:

---

#### Problem 1 — Sequential Scans on Large Tables

Without indexes, every query for a student's notifications results in a full table scan across hundreds of millions of rows.

**Solution: Targeted Indexes**

---

#### Problem 2 — Broadcast Insert Bottleneck

Sending a notification to all 50,000 students means inserting 50,000 rows in one operation — this locks the table and blocks reads.

**Solution: Bulk/batched async Inserts via Queues**

Instead of inserting 50,000 rows synchronously, publish the broadcast to a message queue (e.g. BullMQ / RabbitMQ). Workers consume batches of 500–1,000 rows and insert asynchronously.