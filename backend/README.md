# Classio Backend

> **Express 5, Socket.IO & PostgreSQL Real-Time API Engine**  
> Powers real-time campus room collaboration, persistent messaging with delivery/read receipts, WebRTC signaling, and dual-token cookie authentication.

---

## Table of Contents

- [Overview](#overview)
- [Technology Stack](#technology-stack)
- [Directory Structure](#directory-structure)
- [Server Architecture & Startup Flow](#server-architecture--startup-flow)
- [Database Schema & Models](#database-schema--models)
- [Authentication & Security](#authentication--security)
  - [Dual-Token JWT Strategy](#dual-token-jwt-strategy)
  - [Rate Limiting](#rate-limiting)
  - [Input Validation](#input-validation)
  - [Error Handling](#error-handling)
- [REST API Reference](#rest-api-reference)
  - [Authentication Endpoints](#authentication-endpoints)
  - [User Endpoints](#user-endpoints)
  - [Room Management Endpoints](#room-management-endpoints)
  - [Room Membership & Request Endpoints](#room-membership--request-endpoints)
  - [Message Endpoints](#message-endpoints)
  - [WebRTC ICE Configuration Endpoint](#webrtc-ice-configuration-endpoint)
- [WebSocket & WebRTC Signaling Specification](#websocket--webrtc-signaling-specification)
  - [Handshake Authentication](#handshake-authentication)
  - [Presence & Multi-Device State](#presence--multi-device-state)
  - [Messaging & Receipts Protocol](#messaging--receipts-protocol)
  - [Typing Indicators](#typing-indicators)
  - [WebRTC Signaling State Machine](#webrtc-signaling-state-machine)
- [Environment Configuration](#environment-configuration)
- [Getting Started](#getting-started)
- [Scripts](#scripts)

---

## Overview

The Classio backend is a Node.js server that combines Express 5 REST APIs with Socket.IO real-time bi-directional events. It is backed by a PostgreSQL database accessed via native connection pooling (`pg`) with parameterized SQL queries and transactional integrity.

---

## Technology Stack

| Package                | Version   | Purpose                                                   |
| ---------------------- | --------- | --------------------------------------------------------- |
| **express**            | `^5.2.1`  | HTTP web server and REST routing                          |
| **socket.io**          | `^4.8.4`  | WebSocket engine for chat, presence, and WebRTC signaling |
| **pg**                 | `^8.23.0` | PostgreSQL client pool and query executor                 |
| **jsonwebtoken**       | `^9.0.3`  | Access & refresh token signing and verification           |
| **bcrypt**             | `^6.0.0`  | Password hashing (salt rounds: 10)                        |
| **cookie-parser**      | `^1.4.7`  | HTTP request cookie parser                                |
| **express-validator**  | `^7.3.2`  | Request body schema validation                            |
| **express-rate-limit** | `^8.7.0`  | IP-based rate limiting on sensitive routes                |
| **cors**               | `^2.8.6`  | Cross-Origin Resource Sharing configuration               |
| **dotenv**             | `^18.0.3` | Environment variable loader                               |
| **nodemon**            | `^3.1.14` | Hot-reloading development server                          |
| **prettier**           | `^3.9.9`  | Code formatter                                            |

---

## Directory Structure

```
backend/
├── .env.example                       # Template for environment variables
├── package.json                       # Backend scripts and dependencies
├── .prettierrc                        # Prettier formatting configuration
└── src/
    ├── app.js                         # Express application setup, middlewares, routes & error handling
    ├── index.js                       # Server entry point, DB connectivity, Socket.IO & WebRTC signaling
    ├── controller/
    │   ├── auth.controller.js         # Register, login, refresh tokens, logout
    │   ├── room.controller.js         # Rooms, members, messages, join requests, ICE config
    │   └── user.controller.js         # User listing
    ├── db/
    │   └── database.js                # PostgreSQL Pool instance
    ├── middleware/
    │   ├── auth.middleware.js         # HTTP JWT verification middleware (verifyJWT)
    │   ├── rateLimit.middleware.js    # Auth and API rate limit configurations
    │   └── socket.middleware.js       # WebSocket cookie JWT authentication middleware
    ├── routes/
    │   ├── auth.routes.js             # /api/v1/auth routes
    │   ├── room.routes.js             # /api/v1/room routes
    │   └── user.routes.js             # /api/v1/users routes
    ├── utils/
    │   ├── ApiError.js                # Standardized error class
    │   ├── ApiResponse.js             # Standardized API response wrapper
    │   ├── asyncHandler.js            # Async wrapper passing errors to next()
    │   ├── password.js                # Password hashing and comparison
    │   └── token.js                   # JWT generation helpers
    └── validators/
        ├── auth.validator.js          # Registration and login body validation
        ├── room.validator.js          # Room creation and update body validation
        └── validate.js                # Express-validator error aggregation middleware
```

---

## Server Architecture & Startup Flow

```
1. src/index.js loads environment variables via dotenv.
2. Production check validates essential variables (DATABASE_URL, ACCESS_TOKEN_SECRET, REFRESH_TOKEN_SECRET, CLIENT_URL).
3. Express app instantiated from src/app.js.
4. HTTP Server created via Node's createServer(app).
5. Socket.IO Server attached to HTTP Server with CORS credentials enabled.
6. Socket middleware (verifySocketJWT) attached to validate incoming WebSocket handshakes.
7. testDatabase() runs "SELECT 1" on PostgreSQL pool.
8. Server starts listening on process.env.PORT (default: 5000).
```

---

## Database Schema & Models

The database schema is defined in [schema.sql](../schema.sql).

### 1. `users`

| Column          | Type           | Constraints                             | Description                |
| --------------- | -------------- | --------------------------------------- | -------------------------- |
| `id`            | `UUID`         | `PRIMARY KEY DEFAULT gen_random_uuid()` | Unique user identifier     |
| `username`      | `VARCHAR(50)`  | `NOT NULL UNIQUE`                       | Unique student username    |
| `email`         | `VARCHAR(255)` | `NOT NULL UNIQUE`                       | Unique student email       |
| `password_hash` | `VARCHAR(255)` | `NOT NULL`                              | Bcrypt password hash       |
| `avatar_url`    | `VARCHAR(255)` | `NULLABLE`                              | Profile avatar URL         |
| `created_at`    | `TIMESTAMPTZ`  | `DEFAULT CURRENT_TIMESTAMP`             | Account creation timestamp |

### 2. `user_sessions`

| Column          | Type                       | Constraints                              | Description              |
| --------------- | -------------------------- | ---------------------------------------- | ------------------------ |
| `user_id`       | `UUID`                     | `REFERENCES users(id) ON DELETE CASCADE` | Owner of the session     |
| `refresh_token` | `TEXT`                     | `NOT NULL`                               | Issued JWT refresh token |
| `expires_at`    | `TIMESTAMPTZ`              | `NOT NULL`                               | Token expiration date    |
| _Primary Key_   | `(user_id, refresh_token)` |                                          | Composite primary key    |

### 3. `rooms`

| Column        | Type          | Constraints                               | Description                    |
| ------------- | ------------- | ----------------------------------------- | ------------------------------ |
| `id`          | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`   | Unique room identifier         |
| `name`        | `VARCHAR(50)` | `NOT NULL`                                | Room display name              |
| `description` | `TEXT`        | `NULLABLE`                                | Room description / topic       |
| `owner_id`    | `UUID`        | `REFERENCES users(id) ON DELETE SET NULL` | Creator and owner              |
| `is_private`  | `BOOLEAN`     | `DEFAULT false`                           | Requires join approval if true |
| `created_at`  | `TIMESTAMPTZ` | `DEFAULT CURRENT_TIMESTAMP`               | Room creation timestamp        |

### 4. `room_members`

| Column        | Type                 | Constraints                              | Description                     |
| ------------- | -------------------- | ---------------------------------------- | ------------------------------- |
| `room_id`     | `UUID`               | `REFERENCES rooms(id) ON DELETE CASCADE` | Joined room                     |
| `user_id`     | `UUID`               | `REFERENCES users(id) ON DELETE CASCADE` | Member user                     |
| `role`        | `VARCHAR(20)`        | `DEFAULT 'member'`                       | Member role (`owner`, `member`) |
| `joined_at`   | `TIMESTAMPTZ`        | `DEFAULT CURRENT_TIMESTAMP`              | Membership start timestamp      |
| _Primary Key_ | `(room_id, user_id)` |                                          | Composite primary key           |

### 5. `room_join_requests`

| Column       | Type                 | Constraints                              | Description                          |
| ------------ | -------------------- | ---------------------------------------- | ------------------------------------ |
| `id`         | `UUID`               | `PRIMARY KEY DEFAULT gen_random_uuid()`  | Unique request identifier            |
| `room_id`    | `UUID`               | `REFERENCES rooms(id) ON DELETE CASCADE` | Requested room                       |
| `user_id`    | `UUID`               | `REFERENCES users(id) ON DELETE CASCADE` | Requesting user                      |
| `status`     | `VARCHAR(20)`        | `DEFAULT 'pending'`                      | `pending`, `approved`, or `rejected` |
| `created_at` | `TIMESTAMPTZ`        | `DEFAULT CURRENT_TIMESTAMP`              | Request timestamp                    |
| _Unique_     | `(room_id, user_id)` |                                          | Prevents duplicate requests          |

### 6. `messages`

| Column       | Type          | Constraints                              | Description               |
| ------------ | ------------- | ---------------------------------------- | ------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`  | Unique message identifier |
| `room_id`    | `UUID`        | `REFERENCES rooms(id) ON DELETE CASCADE` | Target room               |
| `user_id`    | `UUID`        | `REFERENCES users(id) ON DELETE CASCADE` | Message sender            |
| `content`    | `TEXT`        | `NOT NULL`                               | Message body text         |
| `created_at` | `TIMESTAMPTZ` | `DEFAULT CURRENT_TIMESTAMP`              | Sent timestamp            |
| `updated_at` | `TIMESTAMPTZ` | `DEFAULT CURRENT_TIMESTAMP`              | Edit timestamp            |

### 7. `message_receipts`

| Column         | Type                    | Constraints                                 | Description                       |
| -------------- | ----------------------- | ------------------------------------------- | --------------------------------- |
| `message_id`   | `UUID`                  | `REFERENCES messages(id) ON DELETE CASCADE` | Reference message                 |
| `user_id`      | `UUID`                  | `REFERENCES users(id) ON DELETE CASCADE`    | Recipient user                    |
| `room_id`      | `UUID`                  | `REFERENCES rooms(id) ON DELETE CASCADE`    | Target room                       |
| `delivered_at` | `TIMESTAMPTZ`           | `NULLABLE`                                  | Delivery timestamp (double check) |
| `read_at`      | `TIMESTAMPTZ`           | `NULLABLE`                                  | Read timestamp (blue check)       |
| _Primary Key_  | `(message_id, user_id)` |                                             | Per-recipient receipt record      |

---

## Authentication & Security

### Dual-Token JWT Strategy

1. **Access Token**:
   - Short-lived token signed with `ACCESS_TOKEN_SECRET`.
   - Stored in an `httpOnly` cookie named `accessToken`.
   - Verified by `verifyJWT` (HTTP) and `verifySocketJWT` (WebSocket).
2. **Refresh Token**:
   - Long-lived token signed with `REFRESH_TOKEN_SECRET`.
   - Stored in an `httpOnly` cookie named `refreshToken`.
   - Persisted in the `user_sessions` database table.
3. **Cookie Attributes**:
   - `httpOnly: true` (inaccessible to JavaScript, prevents XSS extraction).
   - `secure: process.env.NODE_ENV === "production"`.
   - `sameSite`: `lax` (development) or `none` (cross-site production).

### Rate Limiting

Configured in `src/middleware/rateLimit.middleware.js`:

- **`authRateLimiter`**: Max 10 attempts per 15-minute window for `/api/v1/auth/register` and `/api/v1/auth/login`.
- **`apiRateLimiter`**: Max 300 requests per 15-minute window mounted globally at `/api`.

### Input Validation

Utilizes `express-validator` chains:

- **`registerValidator`**: Validates `username` (non-empty, max 50), `email` (valid format), `password` (min 6 chars).
- **`loginValidator`**: Validates `email` format and non-empty `password`.
- **`createRoomValidator`**: Validates `name` (required, max 50), optional `description`, optional `is_private` boolean.
- **`updateRoomValidator`**: Validates optional `name` and `description`.
- Errors are collected by `src/validators/validate.js` and returned as a 400 Bad Request with detailed field errors.

### Error Handling

- Custom **`ApiError`** class standardizes `statusCode`, `message`, and `errors` array.
- Controllers wrap handlers in **`asyncHandler`**, forwarding exceptions to Express.
- Global Error Middleware in `src/app.js` formats all thrown errors into consistent JSON responses:
  ```json
  {
    "success": false,
    "message": "Error description",
    "errors": []
  }
  ```

---

## REST API Reference

All successful responses are formatted via `ApiResponse`:

```json
{
  "statusCode": 200,
  "data": { ... },
  "message": "Success message",
  "success": true
}
```

### Authentication Endpoints

#### `POST /api/v1/auth/register`

- **Auth**: None
- **Rate Limit**: 10 req / 15 min
- **Body**: `{ "username": "alice", "email": "alice@campus.edu", "password": "password123" }`
- **Response**: `201 Created` with created user object and sets `accessToken` & `refreshToken` cookies.

#### `POST /api/v1/auth/login`

- **Auth**: None
- **Rate Limit**: 10 req / 15 min
- **Body**: `{ "email": "alice@campus.edu", "password": "password123" }`
- **Response**: `200 OK` with user profile and sets session cookies.

#### `POST /api/v1/auth/refresh`

- **Auth**: Requires `refreshToken` cookie
- **Response**: `200 OK` with user profile and refreshed `accessToken` cookie.

#### `POST /api/v1/auth/logout`

- **Auth**: Requires cookies
- **Response**: `200 OK` (deletes session from `user_sessions` and clears cookies).

---

### User Endpoints

#### `GET /api/v1/users`

- **Auth**: Required (`verifyJWT`)
- **Response**: `200 OK` with array of registered users:
  ```json
  [
    {
      "id": "uuid",
      "username": "alice",
      "email": "alice@campus.edu",
      "avatar_url": null,
      "created_at": "timestamp"
    }
  ]
  ```

---

### Room Management Endpoints

#### `POST /api/v1/room`

- **Auth**: Required (`verifyJWT`)
- **Body**:
  ```json
  {
    "name": "CS 101 Study Group",
    "description": "Weekly study discussions",
    "is_private": false
  }
  ```
- **Response**: `201 Created`. Executes an ACID transaction creating the room and adding the creator to `room_members` as `owner`.

#### `GET /api/v1/room`

- **Auth**: Required (`verifyJWT`)
- **Response**: `200 OK` with array of rooms the requesting user is a member of.

#### `GET /api/v1/room/discover`

- **Auth**: Required (`verifyJWT`)
- **Response**: `200 OK` with rooms the requesting user has _not_ joined yet.

#### `GET /api/v1/room/:roomId`

- **Auth**: Required (`verifyJWT` + room membership check)
- **Response**: `200 OK` with room metadata.

#### `PATCH /api/v1/room/:roomId`

- **Auth**: Required (`verifyJWT` + must be room owner)
- **Body**: `{ "name": "New Name", "description": "New Desc" }`
- **Response**: `200 OK` with updated room.

#### `DELETE /api/v1/room/:roomId`

- **Auth**: Required (`verifyJWT` + must be room owner)
- **Response**: `200 OK`. Deletes the room and cascades to all members, messages, and receipts.

---

### Room Membership & Request Endpoints

#### `GET /api/v1/room/:roomId/members`

- **Auth**: Required (`verifyJWT` + room membership check)
- **Response**: `200 OK` with member array including `role` and `joined_at`.

#### `POST /api/v1/room/:roomId/join`

- **Auth**: Required (`verifyJWT`)
- **Behavior**:
  - If room is public (`is_private: false`): Adds user immediately to `room_members` (`200 OK`).
  - If room is private (`is_private: true`): Creates a pending record in `room_join_requests` (`200 OK`).

#### `POST /api/v1/room/:roomId/leave`

- **Auth**: Required (`verifyJWT`)
- **Behavior**: Removes user from `room_members`. If owner leaves, ownership transfers or room can be managed accordingly.

#### `GET /api/v1/room/:roomId/join-requests`

- **Auth**: Required (`verifyJWT` + must be room owner)
- **Response**: `200 OK` with list of pending join requests.

#### `POST /api/v1/room/:roomId/join-requests/:requestId/approve`

- **Auth**: Required (`verifyJWT` + must be room owner)
- **Response**: `200 OK`. Adds requesting user to `room_members` and marks request status as `approved`.

#### `POST /api/v1/room/:roomId/join-requests/:requestId/reject`

- **Auth**: Required (`verifyJWT` + must be room owner)
- **Response**: `200 OK`. Updates request status to `rejected`.

---

### Message Endpoints

#### `GET /api/v1/room/:roomId/messages`

- **Auth**: Required (`verifyJWT` + room membership check)
- **Query Params**: `limit` (default: 50), `cursor` (optional pagination cursor)
- **Response**: `200 OK` with messages array ordered chronologically, along with delivery & read counts.

#### `POST /api/v1/room/:roomId/messages`

- **Auth**: Required (`verifyJWT` + room membership check)
- **Body**: `{ "content": "Hello campus!" }`
- **Response**: `201 Created` (REST fallback for sending messages).

#### `PATCH /api/v1/room/:roomId/messages/:messageId`

- **Auth**: Required (`verifyJWT` + must be message author)
- **Body**: `{ "content": "Updated text" }`
- **Response**: `200 OK`.

#### `DELETE /api/v1/room/:roomId/messages/:messageId`

- **Auth**: Required (`verifyJWT` + must be message author)
- **Response**: `200 OK`.

---

### WebRTC ICE Configuration Endpoint

#### `GET /api/v1/room/ice-servers`

- **Auth**: Required (`verifyJWT`)
- **Response**: `200 OK` returning configured STUN servers (e.g. Google public STUN) and optional TURN credentials configured in `.env`.

---

## WebSocket & WebRTC Signaling Specification

The WebSocket server runs inside `src/index.js` using Socket.IO.

### Handshake Authentication

Before establishing a WebSocket connection, `verifySocketJWT` middleware executes:

1. Extracts `accessToken` from handshake cookie headers (`socket.handshake.headers.cookie`).
2. Verifies token with `process.env.ACCESS_TOKEN_SECRET`.
3. Attaches decoded user payload to `socket.user`.
4. Rejects invalid connections with `"Unauthorized request"` or `"Invalid access token"`.

---

### Presence & Multi-Device State

- The backend maintains an in-memory `onlineUsers` map: `Map<userId, Set<socketId>>`.
- Supports users logged in simultaneously from multiple browser tabs or devices.
- **Events**:
  - `online-users`: Emitted to newly connected client containing list of active user IDs.
  - `user-online`: Broadcast when a user transitions from 0 connected sockets to 1 socket.
  - `user-offline`: Broadcast when all active sockets for a user disconnect.

---

### Messaging & Receipts Protocol

1. **`join-room`**:
   - Client sends `(roomId)`.
   - Server checks membership in `room_members`. If valid, invokes `socket.join(roomId)` and responds with `joined-room`.
2. **`send-message`**:
   - Client sends `{ roomId, content }` with an acknowledgement callback.
   - Server validates membership, inserts message into `messages` table, and immediately returns the saved message via the acknowledgement callback.
   - Server queries online room members and creates batch `message_receipts` with `delivered_at = CURRENT_TIMESTAMP`.
   - Broadcasts `new-message` to all online recipient sockets.
   - Emits `receipt-update` back to the sender.
3. **`mark-room-read`**:
   - Client sends `(roomId)` upon opening a room.
   - Server updates `message_receipts` setting `read_at = CURRENT_TIMESTAMP` for all unread messages.
   - Notifies senders with `receipt-update`.
4. **`message-read`**:
   - Client sends `{ messageId }`.
   - Server marks single message receipt as read and emits `receipt-update` to sender.
5. **`sync-room-receipts`**:
   - Client requests `(roomId, callback)`.
   - Server aggregates delivery and read counts for all sent messages in that room.

---

### Typing Indicators

- **`start-typing`**: Client sends `{ roomId }`. Server broadcasts `user-typing` (`{ userId, username }`) to room peers.
- **`stop-typing`**: Client sends `{ roomId }`. Server broadcasts `user-stopped-typing` (`{ userId }`) to room peers.

---

### WebRTC Signaling State Machine

The backend acts as the signaling channel for 1-on-1 peer-to-peer WebRTC calls. It tracks ongoing calls in `activeCalls`: `Map<userId, { peerId, status, callType, timeoutId }>`.

| Event                 | Direction       | Payload                               | Description                                                                                                     |
| --------------------- | --------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `call-user`           | Client → Server | `{ to, callType }`                    | Initiates call. Validates busy status, registers call in `activeCalls`, and emits `incoming-call` to recipient. |
| `incoming-call`       | Server → Client | `{ from, username, callType }`        | Alerts recipient of incoming audio/video call.                                                                  |
| `accept-call`         | Client → Server | `{ to, callType }`                    | Recipient accepts call. Server updates status to `in-call` and emits `call-accepted`.                           |
| `call-accepted`       | Server → Client | `{ from, username }`                  | Notifies caller to begin WebRTC offer generation.                                                               |
| `reject-call`         | Client → Server | `{ to }`                              | Recipient declines call. Cleans up `activeCalls` and emits `call-rejected`.                                     |
| `end-call`            | Client → Server | `{ to }`                              | Terminates call. Cleans up `activeCalls` and emits `call-ended`.                                                |
| `webrtc-offer`        | Client → Server | `{ to, offer, callType, isRecovery }` | Relays SDP offer to recipient socket.                                                                           |
| `webrtc-answer`       | Client → Server | `{ to, answer }`                      | Relays SDP answer back to caller socket.                                                                        |
| `ice-candidate`       | Client → Server | `{ to, candidate }`                   | Relays ICE candidate to peer socket.                                                                            |
| `camera-state`        | Client → Server | `{ to, enabled }`                     | Notifies peer when camera track is toggled on/off.                                                              |
| `call-recovery-ready` | Client → Server | (none)                                | Reconnecting client signals readiness to recover interrupted call.                                              |
| `resume-call`         | Server → Client | `{ remoteUserId, callType }`          | Server triggers peer re-negotiation following recovery.                                                         |

---

## Environment Configuration

Create a `.env` file in the `backend/` directory:

```env
# Server
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:5173

# Database
DATABASE_URL=postgresql://postgres:your_password@localhost:5432/classio_db

# JWT Secrets
ACCESS_TOKEN_SECRET=your_super_secret_access_token_here_replace_in_production
REFRESH_TOKEN_SECRET=your_super_secret_refresh_token_here_replace_in_production
ACCESS_TOKEN_EXPIRY=1d
REFRESH_TOKEN_EXPIRY=15d

# Proxy & Cookie (Optional, recommended for deployment)
# TRUST_PROXY=1
# COOKIE_SAME_SITE=lax

# WebRTC TURN Relay (Optional)
# TURN_URL=turn:turn.example.com:3478
# TURN_USERNAME=username
# TURN_PASSWORD=password
```

---

## Getting Started

1. **Install dependencies**:
   ```bash
   npm install
   ```
2. **Ensure database is running and schema applied**:
   ```bash
   psql -d classio_db -f ../schema.sql
   ```
3. **Start development server**:
   ```bash
   npm run dev
   ```

---

## Scripts

- `npm run dev`: Runs server with `nodemon` for auto-restart on file changes.
- `npm start`: Runs server in production mode using `node src/index.js`.
- `npm run format`: Formats code using Prettier according to `.prettierrc`.
