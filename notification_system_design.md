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