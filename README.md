# Classio

> **A real-time campus communication and collaboration platform designed for college communities.**  
> Connect through topic-based rooms, persistent real-time chat with delivery/read receipts, live typing indicators, and peer-to-peer WebRTC voice, video, and screen sharing.

### 🌐 Live Demo: [classio-five.vercel.app](https://classio-five.vercel.app/)

---

## Table of Contents

- [Live Demo](#-live-demo-classio-fivevercelapp)
- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Key Features](#key-features)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [1. Clone Repository](#1-clone-repository)
  - [2. Database Setup](#2-database-setup)
  - [3. Backend Setup](#3-backend-setup)
  - [4. Frontend Setup](#4-frontend-setup)
- [Environment Variables](#environment-variables)
- [API & WebSocket Protocol Overview](#api--websocket-protocol-overview)
- [Security & Engineering Highlights](#security--engineering-highlights)
- [Available Scripts](#available-scripts)
- [License](#license)

---

## Overview

**Classio** bridges the gap between campus collaboration and modern real-time communication tools like Discord and Slack. It provides university students, study groups, clubs, and campus organizations with dedicated digital spaces ("Rooms") to collaborate seamlessly.

Whether organizing project discussions, hanging out in study rooms, or jumping into high-definition 1-on-1 video calls with screen sharing, Classio offers a low-latency, resilient, and responsive full-stack experience.

---

## System Architecture

```
                                  +-----------------------------+
                                  |     React 19 Frontend       |
                                  |        (Vite 8 SPA)         |
                                  +--------------+--------------+
                                                 |
                       +-------------------------+-------------------------+
                       | HTTP (Cookies / REST)   | WebSockets (Socket.IO)  | WebRTC (P2P Media)
                       v                         v                         |
          +-----------------------------------------------+                |
          |           Node.js / Express 5 Backend         |                |
          |  +-----------------+   +--------------------+ |                |
          |  |  REST Routers   |   | Socket.IO Server   | |                |
          |  |  - Auth         |   | - Presence Sync    | |                |
          |  |  - Rooms        |   | - Live Messaging   | |                |
          |  |  - Users        |   | - Receipts & Typing| |                |
          |  |  - ICE Servers  |   | - WebRTC Signaling | |                |
          |  +--------+--------+   +---------+----------+ |                |
          +-----------|----------------------|------------+                |
                      |                      |                             |
                      +----------+-----------+                             |
                                 |                                         |
                                 v                                         v
                      +---------------------+                    +-------------------+
                      | PostgreSQL Database |                    | Peer WebRTC Media |
                      | (Connection Pool)   |                    | (Audio/Video/Share|
                      +---------------------+                    +-------------------+
```

---

## Key Features

### 1. Robust Authentication & Sessions

- **Dual-Token JWT Security**: Short-lived Access Tokens and persistent Refresh Tokens stored in secure, `httpOnly`, `SameSite` cookies.
- **Session Revocation**: Active refresh tokens are tracked in PostgreSQL (`user_sessions`), enabling instant logout and session invalidation.
- **Password Security**: Passwords hashed with `bcrypt` (10 salt rounds).
- **Brute-Force Protection**: Endpoint rate limiting via `express-rate-limit` prevents credential abuse.

### 2. Room & Community Management

- **Public & Private Rooms**: Create open communities that any campus student can join instantly, or private rooms that require owner approval.
- **Room Discovery**: Explore campus rooms with the `/discover` feed.
- **Join Request Workflow**: Private rooms feature an owner-moderated request approval and rejection pipeline.
- **Member Directory**: Track membership, join dates, and roles (`owner`, `member`).
- **Room Administration**: Room owners can edit room metadata or delete the room.

### 3. Real-Time Messaging & Receipts

- **Bi-directional WebSockets**: Built on Socket.IO for real-time messaging latency.
- **Persistent Message History**: All messages stored in PostgreSQL with pagination and message timestamps.
- **Delivery & Read Receipts**:
  - Single checkmark: Sent to server.
  - Double checkmarks: Delivered to online recipients (`delivered_at`).
  - Blue double checkmarks: Read by recipients (`read_at`).
- **Live Typing Indicators**: Broadcasted room-level typing status with auto-clearing timers.
- **Message Editing & Deletion**: Edit or remove messages with real-time UI updates.

### 4. WebRTC Video, Voice & Screen Sharing

- **1-on-1 Audio & Video Calls**: Peer-to-peer media communication using native browser WebRTC APIs.
- **Dynamic STUN/TURN Configuration**: Auto-configured STUN servers and support for custom TURN relays via `/api/v1/room/ice-servers`.
- **Screen Sharing**: Seamless track substitution allowing users to switch between webcam and screen capture during live calls.
- **Call Overlay & Controls**: Floating/minimized video preview overlay, mic mute, camera toggle, incoming call ringing, and busy state detection.
- **Reconnection Recovery**: Graceful call recovery mechanisms if temporary network hiccups occur.

### 5. Multi-Device Presence System

- **Real-Time Online Status**: Live user tracking across multiple tabs or devices simultaneously using user-to-socket set mappings.
- **Instant Status Broadcasts**: Connected peers are immediately alerted when friends go online or offline.

---

## Technology Stack

| Layer        | Technologies                                                                                                                                                 |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Frontend** | React 19, Vite 8, Socket.IO Client, Native WebRTC (`RTCPeerConnection`), Vanilla CSS (Custom Design System)                                                  |
| **Backend**  | Node.js, Express 5, Socket.IO 4, `pg` (PostgreSQL Client Pool), `bcrypt`, `jsonwebtoken`, `express-validator`, `express-rate-limit`, `cors`, `cookie-parser` |
| **Database** | PostgreSQL with raw parameterized SQL queries & ACID transactions                                                                                            |
| **Tooling**  | Prettier, ESLint 10, Nodemon                                                                                                                                 |

---

## Project Structure

```
Classio/
├── README.md                          # Repository root documentation
├── schema.sql                         # Complete PostgreSQL database schema
├── backend/
│   ├── README.md                      # Backend service documentation
│   ├── package.json                   # Backend dependencies and scripts
│   ├── .env.example                   # Backend environment variables template
│   ├── .prettierrc                    # Code formatting configuration
│   └── src/
│       ├── app.js                     # Express app setup, middlewares & error handling
│       ├── index.js                   # Server bootstrap, Socket.IO & WebRTC signaling
│       ├── controller/                # Request handlers & SQL queries
│       │   ├── auth.controller.js     # Register, login, refresh, logout
│       │   ├── room.controller.js     # Room CRUD, members, messages, join requests, ICE
│       │   └── user.controller.js     # User directory
│       ├── db/
│       │   └── database.js            # PostgreSQL connection pool
│       ├── middleware/
│       │   ├── auth.middleware.js     # HTTP JWT verification (verifyJWT)
│       │   ├── rateLimit.middleware.js# Auth and API rate limiters
│       │   └── socket.middleware.js   # WebSocket cookie JWT authentication
│       ├── routes/
│       │   ├── auth.routes.js         # Auth routing
│       │   ├── room.routes.js         # Room, message, and membership routing
│       │   └── user.routes.js         # User routing
│       ├── utils/
│       │   ├── ApiError.js            # Custom error class
│       │   ├── ApiResponse.js         # Standard response wrapper
│       │   ├── asyncHandler.js        # Async route wrapper
│       │   ├── password.js            # Bcrypt hashing & verification
│       │   └── token.js               # Access & refresh token generators
│       └── validators/                # Input validation schemas
│           ├── auth.validator.js      # Auth body validators
│           ├── room.validator.js      # Room body validators
│           └── validate.js            # Express-validator error handler
└── frontend/
    ├── README.md                      # Frontend application documentation
    ├── package.json                   # Frontend dependencies and scripts
    ├── index.html                     # HTML entry point
    ├── vite.config.js                 # Vite bundler configuration
    ├── eslint.config.js               # ESLint configuration
    ├── .env.example                   # Frontend environment variables template
    └── src/
        ├── main.jsx                   # React root mount
        ├── App.jsx                    # Root app state, auth routing, presence sync
        ├── assets/                    # Static assets & brand logos
        ├── components/
        │   ├── VideoCallOverlay.jsx   # Draggable/floating WebRTC call interface
        │   ├── common/
        │   │   └── CreateRoomModal.jsx# New room creation modal
        │   └── layout/
        │       └── AppShell.jsx       # Main navigation header & user menu
        ├── contexts/
        │   └── CallContext.jsx        # WebRTC call state machine & media handling
        ├── pages/
        │   ├── Login.jsx              # User login screen
        │   ├── Register.jsx           # Account creation screen
        │   ├── Messages.jsx           # Messages layout & room selector
        │   ├── ChatRoom.jsx           # Real-time room chat, receipts & call actions
        │   └── Rooms.jsx              # Campus rooms discovery feed
        ├── services/
        │   └── socket.js              # Socket.IO client singleton
        └── styles/
            ├── globals.css            # Dark mode tokens, reset, utility classes
            ├── responsive.css         # Mobile & tablet responsiveness
            └── typing.css             # Typing indicator animation styles
```

---

## Database Schema

The database schema is defined in [schema.sql](schema.sql). It consists of 7 normalized relational tables:

```mermaid
erDiagram
    users ||--o{ user_sessions : has
    users ||--o{ rooms : owns
    users ||--o{ room_members : joins
    users ||--o{ room_join_requests : requests
    users ||--o{ messages : sends
    users ||--o{ message_receipts : receives
    rooms ||--o{ room_members : contains
    rooms ||--o{ room_join_requests : receives
    rooms ||--o{ messages : contains
    messages ||--o{ message_receipts : generates
```

- **`users`**: User identities, credentials, and profile metadata.
- **`user_sessions`**: Stored refresh tokens with expiration timestamps for session management.
- **`rooms`**: Community room metadata, descriptions, privacy settings (`is_private`), and ownership.
- **`room_members`**: Junction table mapping users to rooms with roles (`owner`, `member`) and `joined_at` timestamps.
- **`room_join_requests`**: Moderation table for private room join applications (`pending`, `approved`, `rejected`).
- **`messages`**: Room messages containing content, timestamps, and foreign keys to room and author.
- **`message_receipts`**: Granular per-recipient delivery (`delivered_at`) and read (`read_at`) receipts.

---

## Getting Started

### Prerequisites

- **Node.js**: `v18.x` or `v20.x` or later
- **PostgreSQL**: `v14` or later (local installation or hosted service such as Supabase / Neon / AWS RDS)
- **npm**: `v9.x` or later

---

### 1. Clone Repository

```bash
git clone https://github.com/Harsh-sorout-2007/classio.git
cd classio
```

---

### 2. Database Setup

1. Create a PostgreSQL database (e.g., `classio_db`):
   ```sql
   CREATE DATABASE classio_db;
   ```
2. Execute the schema script located at the root of the project:
   ```bash
   psql -d classio_db -f schema.sql
   ```
   _(Or copy the contents of `schema.sql` into pgAdmin, DBeaver, or your cloud SQL console)._

---

### 3. Backend Setup

1. Navigate to the backend directory:
   ```bash
   cd backend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Create your `.env` file based on `.env.example`:
   ```bash
   cp .env.example .env
   ```
4. Populate the `.env` variables:
   ```env
   PORT=5000
   NODE_ENV=development
   CLIENT_URL=http://localhost:5173
   DATABASE_URL=postgresql://postgres:your_password@localhost:5432/classio_db
   ACCESS_TOKEN_SECRET=replace_with_a_long_random_secret_for_access_tokens
   REFRESH_TOKEN_SECRET=replace_with_a_long_random_secret_for_refresh_tokens
   ACCESS_TOKEN_EXPIRY=1d
   REFRESH_TOKEN_EXPIRY=15d
   ```
5. Start the backend development server:
   ```bash
   npm run dev
   ```
   The backend will run at `http://localhost:5000`.

---

### 4. Frontend Setup

1. Open a new terminal and navigate to the frontend directory:
   ```bash
   cd frontend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Create your `.env` file based on `.env.example`:
   ```bash
   cp .env.example .env
   ```
4. Configure the API endpoint:
   ```env
   VITE_API_URL=http://localhost:5000
   ```
5. Start the Vite development server:
   ```bash
   npm run dev
   ```
6. Open your browser and navigate to `http://localhost:5173`.

---

## Environment Variables

### Backend (`backend/.env`)

| Variable               | Required | Description                                                                         | Example                                            |
| ---------------------- | -------- | ----------------------------------------------------------------------------------- | -------------------------------------------------- |
| `PORT`                 | No       | Server port (default: `5000`)                                                       | `5000`                                             |
| `NODE_ENV`             | Yes      | Runtime environment (`development` or `production`)                                 | `development`                                      |
| `CLIENT_URL`           | Yes      | Allowed frontend origin for CORS and cookies                                        | `http://localhost:5173`                            |
| `DATABASE_URL`         | Yes      | PostgreSQL connection string                                                        | `postgresql://user:pass@localhost:5432/classio_db` |
| `ACCESS_TOKEN_SECRET`  | Yes      | Secret key used to sign JWT access tokens                                           | `your_access_secret`                               |
| `REFRESH_TOKEN_SECRET` | Yes      | Secret key used to sign JWT refresh tokens                                          | `your_refresh_secret`                              |
| `ACCESS_TOKEN_EXPIRY`  | No       | Expiration time for access tokens (default: `1d`)                                   | `1d`                                               |
| `REFRESH_TOKEN_EXPIRY` | No       | Expiration time for refresh tokens (default: `15d`)                                 | `15d`                                              |
| `TRUST_PROXY`          | No       | Trust proxy configuration when deployed behind a reverse proxy (e.g. Render, Nginx) | `1`                                                |
| `COOKIE_SAME_SITE`     | No       | Cookie `sameSite` policy (`lax` or `none`)                                          | `lax`                                              |
| `TURN_URL`             | No       | Optional TURN server URL for WebRTC NAT traversal                                   | `turn:turn.example.com:3478`                       |
| `TURN_USERNAME`        | No       | Optional TURN server username                                                       | `user`                                             |
| `TURN_PASSWORD`        | No       | Optional TURN server password                                                       | `password`                                         |

### Frontend (`frontend/.env`)

| Variable       | Required | Description                                     | Example                 |
| -------------- | -------- | ----------------------------------------------- | ----------------------- |
| `VITE_API_URL` | Yes      | Base URL of the backend REST & Socket.IO server | `http://localhost:5000` |

---

## API & WebSocket Protocol Overview

### REST API Highlights

- **Authentication**:
  - `POST /api/v1/auth/register` — Register a new account.
  - `POST /api/v1/auth/login` — Authenticate and receive `httpOnly` cookies.
  - `POST /api/v1/auth/refresh` — Issue a new access token using the stored refresh cookie.
  - `POST /api/v1/auth/logout` — Revoke database session and clear cookies.
- **Rooms & Membership**:
  - `POST /api/v1/room/` — Create room (atomic SQL transaction with owner membership).
  - `GET /api/v1/room/` — Fetch all rooms joined by the current user.
  - `GET /api/v1/room/discover` — Discover rooms available to join.
  - `GET /api/v1/room/:roomId` — Fetch room details.
  - `PATCH /api/v1/room/:roomId` — Update room metadata (owner only).
  - `DELETE /api/v1/room/:roomId` — Delete room (owner only).
  - `GET /api/v1/room/:roomId/members` — Fetch room member list.
  - `POST /api/v1/room/:roomId/join` — Join public room or submit private room join request.
  - `POST /api/v1/room/:roomId/leave` — Leave a joined room.
  - `GET /api/v1/room/:roomId/join-requests` — View pending requests (owner only).
  - `POST /api/v1/room/:roomId/join-requests/:requestId/approve` — Approve request (owner only).
  - `POST /api/v1/room/:roomId/join-requests/:requestId/reject` — Reject request (owner only).
- **Messages**:
  - `GET /api/v1/room/:roomId/messages` — Fetch message history with delivery/read statistics.
  - `POST /api/v1/room/:roomId/messages` — Send message (HTTP fallback).
  - `PATCH /api/v1/room/:roomId/messages/:messageId` — Edit message content.
  - `DELETE /api/v1/room/:roomId/messages/:messageId` — Delete message.
- **Signaling & Configuration**:
  - `GET /api/v1/room/ice-servers` — Retrieve STUN/TURN server configurations for WebRTC.
- **Users**:
  - `GET /api/v1/users` — Fetch directory of registered users.

### Real-Time WebSocket & Signaling Events

- **Presence**: `online-users`, `user-online`, `user-offline`.
- **Rooms & Chat**: `join-room`, `send-message`, `new-message`, `mark-room-read`, `message-read`, `sync-room-receipts`, `receipt-update`.
- **Typing Status**: `start-typing`, `stop-typing`, `user-typing`, `user-stopped-typing`.
- **WebRTC Signaling**: `call-user`, `accept-call`, `reject-call`, `end-call`, `incoming-call`, `call-accepted`, `call-rejected`, `call-ended`, `webrtc-offer`, `webrtc-answer`, `ice-candidate`, `camera-state`, `call-recovery-ready`, `resume-call`.

---

## Security & Engineering Highlights

1. **Defense-in-Depth Authentication**: Access tokens never touch `localStorage` or `sessionStorage`, protecting against token leakage via XSS.
2. **SQL Injection Immunity**: 100% of database interactions utilize PostgreSQL parameterized queries (`$1, $2, ...`).
3. **Transaction Safety**: Multi-table insertions (such as room creation + owner membership assignment) run within isolated SQL transactions with automatic rollback on error.
4. **WebSocket Cookie Authentication**: Handshake validation verifies user tokens before any socket connection is accepted.
5. **Rate Limiting**: Differentiates between sensitive authentication endpoints (10 attempts/15 min) and regular API calls (300 requests/15 min).
6. **Graceful Error Handling**: Custom `ApiError` class combined with `asyncHandler` ensures expressive, safe error responses without server crashes.

---

## Available Scripts

### Backend (`/backend`)

- `npm run dev` — Starts backend in development mode with `nodemon`.
- `npm start` — Starts backend in production mode with `node`.
- `npm run format` — Formats all backend files using Prettier.

### Frontend (`/frontend`)

- `npm run dev` — Starts Vite dev server with Hot Module Replacement (HMR).
- `npm run build` — Bundles production assets into `dist/`.
- `npm run preview` — Locally previews the production build.
- `npm run lint` — Runs ESLint on all frontend files.
- `npm run format` — Formats all frontend files using Prettier.

---
