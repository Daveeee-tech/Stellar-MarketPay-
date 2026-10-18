# WebSocket Events API

The Stellar MarketPay backend emits real-time WebSocket events for bid updates, notifications, job state changes, and more. This document outlines connection options, subscriptions, emitted events, and their payload schemas.

## Connection, Authentication, and Rooms

Connect to `/ws/realtime`. A JWT is optional; provide it as a `token` query parameter when the client should receive user-specific notification delivery and replay. The server verifies it with `JWT_SECRET` and reads the user's `publicKey`. A missing, invalid, or unverifiable token does not reject the connection; it is treated as anonymous.

```text
wss://<host>/ws/realtime?token=<url-encoded-jwt>
```

Rooms can be requested on connection in either of these forms. `room` may be repeated; `rooms` is a comma-separated list. Values are trimmed before validation.

```text
wss://<host>/ws/realtime?room=job:job-uuid-1234&room=user:GAXJ4...
wss://<host>/ws/realtime?rooms=job:job-uuid-1234,user:GAXJ4...
```

Room IDs must match `^(job|user):[^\s:]+$`: the prefix is exactly `job` or `user`, followed by a colon and a non-empty ID containing no whitespace or colon. Invalid IDs are ignored. **The realtime handler does not authorize room membership**: any connected client can request any syntactically valid room. Do not treat a room subscription as access control or send sensitive data to a room on that assumption.

The server derives room targets from event payloads: `jobId` or `job_id` targets `job:<id>`; `userAddress`, `user_address`, or `userId` targets `user:<id>`; payload `rooms` arrays and `roomId` also add targets. If a broadcast has no room targets, it is sent to every realtime connection. The event name itself is not used to select a room. For example, current `job:<id>:bids` broadcasts do not include a job ID in their payload and therefore reach all realtime connections; clients should filter by `event`.

Notification delivery is separate from room membership. Live user notifications are sent to sockets authenticated as that user, and replay requires a valid JWT identifying the user.

### Client-to-Server Subscription Messages

The client may send JSON text frames to change room membership. `rooms` must be an array; IDs that fail the room grammar are ignored. Re-subscribing to an existing room has no additional effect.

Subscribe:

```json
{"type":"subscribe","rooms":["job:job-uuid-1234","user:GAXJ4..."]}
```

Unsubscribe:

```json
{"type":"unsubscribe","rooms":["job:job-uuid-1234"]}
```

Both valid message types receive a `subscribed` frame containing the socket's complete current room list, including after an unsubscribe. Malformed JSON, unknown message types, and messages whose `rooms` value is not an array are ignored without an error frame.

### Connection and Subscription Acknowledgements

After processing query-param room requests, the server sends:

```json
{"event":"connected","payload":{"channel":"realtime"}}
```

After either subscription message, the server sends the complete current room list:

```json
{"event":"subscribed","payload":{"rooms":["job:job-uuid-1234","user:GAXJ4..."]}}
```

All server frames use the envelope `{ "event": "<event_name>", "payload": <object> }`.

### Notification Replay on Reconnect

For a valid JWT, the server queries the user's 20 most recently created notifications when the socket connects. It sends the ones with `created_at` later than the timestamp recorded when that user's previous socket closed, ordered oldest first. Each is sent as a normal `notification:created` frame. On a user's first connection, the comparison starts at the Unix epoch, so at most the latest 20 notifications are replayed. The last-seen timestamp is held in process memory and is updated on socket close; it is not a delivery receipt, and it is lost when the server process restarts. Anonymous connections receive no notification replay. Replay failures are non-fatal and are not reported with a separate frame.

### Example: Client-Side Usage (JavaScript)

```javascript
// Example using native WebSocket API
const token = 'your.jwt.token.here';
const params = new URLSearchParams({ token });
params.append('room', 'job:job-uuid-1234');
const ws = new WebSocket(`ws://localhost:4000/ws/realtime?${params}`);

ws.onopen = () => {
  console.log('Connected to WebSocket server');
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(`Received event: ${data.event}`, data.payload);
  
  // Handle specific events
  if (data.event === 'notification:created') {
    alert(`New Notification: ${data.payload.title}`);
  }
};

ws.onerror = (error) => {
  console.error('WebSocket Error:', error);
};

ws.onclose = () => {
  console.log('WebSocket Connection Closed');
};
```

---

## Server Events

Each example below is a complete JSON frame, including the common envelope.

### `notification:created`
Emitted when a new in-app notification is generated for the authenticated user.
```json
{
  "event": "notification:created",
  "payload": {
    "id": 123,
    "userAddress": "GAXJ4...",
    "type": "escrow_created",
    "title": "Notification Title",
    "body": "Notification Body",
    "read": false,
    "jobId": "job-uuid-1234",
    "linkPath": "/jobs/job-uuid-1234",
    "createdAt": "2026-07-29T11:14:00Z"
  }
}
```

### `job:{jobId}:bids`
Application updates use an event name such as `job:job-uuid-1234:bids`. The current frontend handles these payload types:

New bid:
```json
{
  "event": "job:job-uuid-1234:bids",
  "payload": {
    "type": "new_bid",
    "application": {
      "id": 456,
      "freelancerAddress": "GBYJ4...",
      "bidAmount": 500,
      "proposal": "Proposal text...",
      "estimatedDuration": 7,
      "createdAt": "2026-07-29T11:14:00Z",
      "status": "pending"
    },
    "jobTitle": "Job Title"
  }
}
```

Withdrawn application:
```json
{"event":"job:job-uuid-1234:bids","payload":{"type":"application:withdrawn","applicationId":"application-456"}}
```

Accepted application:
```json
{"event":"job:job-uuid-1234:bids","payload":{"type":"application:accepted","applicationId":"application-456"}}
```

Bulk status update:
```json
{"event":"job:job-uuid-1234:bids","payload":{"type":"applications:bulk_update","applicationIds":["application-456","application-789"],"status":"rejected"}}
```

### `job:invited`
Emitted when a freelancer is invited to a private/invite-only job.
```json
{
  "event": "job:invited",
  "payload": {
    "jobId": "job-uuid-1234",
    "recipientAddress": "GBYJ4...",
    "invitedAt": "2026-07-29T11:14:00Z"
  }
}
```

### `job:status-changed`
Emitted when the indexer observes an on-chain status change for a job (e.g., escrow released).
```json
{
  "event": "job:status-changed",
  "payload": {
    "jobId": "job-uuid-1234",
    "status": "completed",
    "txHash": "abc123hash...",
    "ledger": 12345678
  }
}
```

### `job:expiry-warning`
Emitted by the job expiry checker to warn users about jobs expiring within 3 days.
```json
{
  "event": "job:expiry-warning",
  "payload": {
    "count": 2,
    "jobs": [
      {
        "id": "job-uuid-1234",
        "title": "Expiring Job Title",
        "expiresAt": "2026-08-01T00:00:00Z"
      }
    ]
  }
}
```

### `jobs:expired`
Emitted when old jobs have been automatically expired by the system.
```json
{
  "event": "jobs:expired",
  "payload": {
    "count": 5,
    "timestamp": "2026-07-29T11:14:00Z"
  }
}
```

### `price:alert`
Emitted when an XLM price threshold is crossed for a user's price alert preferences.
```json
{
  "event": "price:alert",
  "payload": {
    "recipientAddress": "GAXJ4...",
    "kind": "min",
    "currentPriceUsd": 0.12,
    "threshold": 0.15,
    "triggeredAt": "2026-07-29T11:14:00Z"
  }
}
```

### `contract:event`
Emitted when a tracked Soroban contract event occurs.
```json
{
  "event": "contract:event",
  "payload": {
    "jobId": "job-uuid-1234",
    "eventType": "payment_released",
    "txHash": "abc123hash..."
  }
}
```

### `contract:transaction`
Emitted when a relevant transaction to the tracked Soroban contract happens.
```json
{
  "event": "contract:transaction",
  "payload": {
    "txHash": "abc123hash...",
    "contractId": "CAXJ..."
  }
}
```

### `analytics:leaderboard-updated`
Emitted when the donation leaderboard updates.
```json
{
  "event": "analytics:leaderboard-updated",
  "payload": {
    "leaderboard": [
      {
        "address": "GAXJ4...",
        "total_donated_xlm": "100.5",
        "donation_count": 5
      }
    ],
    "txHash": "abc123hash..."
  }
}
```
