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

### Base 
---

### Common Request Headers

| Header | Value | Required |
|---|---|---|
| `Authorization` | `Bearer <access_token>` | Yes |
| `Content-Type` | `application/json` | Yes (for POST/PATCH) |
| `Accept` | `application/json` | Recommended |

---

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
    "id": "d146095a-0d86-4a34-9e69-3900a14576bc",
    "type": "Placement",
    "message": "Campus drive by Google on 28th May 2026",
    "timestamp": "2026-05-14T10:00:00Z",
    "isRead": false,
    "studentIds": ["student-uuid-1", "student-uuid-2"]
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