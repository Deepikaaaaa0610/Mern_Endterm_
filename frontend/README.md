# Classio Frontend

> **React 19 & Vite 8 Single-Page Application (SPA)**  
> Campus collaboration client featuring persistent real-time chat, delivery and read receipts, live typing indicators, and peer-to-peer WebRTC voice, video, and screen sharing.

---

## Table of Contents

- [Overview](#overview)
- [Technology Stack](#technology-stack)
- [Directory Structure](#directory-structure)
- [Application Architecture](#application-architecture)
  - [Bootstrap & Authentication Lifecycle](#bootstrap--authentication-lifecycle)
  - [Presence & Socket State Synchronization](#presence--socket-state-synchronization)
  - [Layout & Navigation Flow](#layout--navigation-flow)
- [Features & Modules](#features--modules)
  - [1. Authentication Module](#1-authentication-module)
  - [2. App Navigation Shell](#2-app-navigation-shell)
  - [3. Real-Time Chat & Receipts](#3-real-time-chat--receipts)
  - [4. WebRTC Audio, Video & Screen Sharing](#4-webrtc-audio-video--screen-sharing)
  - [5. Room Discovery & Creation](#5-room-discovery--creation)
- [WebRTC Architecture & Implementation](#webrtc-architecture--implementation)
- [Design System & Styling](#design-system--styling)
- [Environment Configuration](#environment-configuration)
- [Getting Started](#getting-started)
- [Available Scripts](#available-scripts)

---

## Overview

The Classio frontend is a fast, responsive Single Page Application built on **React 19** and **Vite 8**. It communicates with the backend via:

1. **REST APIs** (with `credentials: "include"` for HTTP-only cookie session handling).
2. **Socket.IO** (bi-directional events for chat, presence, typing, and notifications).
3. **WebRTC PeerConnection** (peer-to-peer audio, video, and screen sharing streams).

---

## Technology Stack

| Library / Tool           | Version    | Description                                            |
| ------------------------ | ---------- | ------------------------------------------------------ |
| **react**                | `^19.2.8`  | Core UI library                                        |
| **react-dom**            | `^19.2.8`  | React DOM renderer                                     |
| **vite**                 | `^8.3.0`   | Next-generation frontend build tool and dev server     |
| **socket.io-client**     | `^4.8.4`   | WebSocket client for real-time messaging and signaling |
| **@vitejs/plugin-react** | `^6.1.1`   | Vite official plugin for React Fast Refresh            |
| **eslint**               | `^10.10.0` | Code quality and linting                               |
| **prettier**             | `^3.9.9`   | Opinionated code formatter                             |

---

## Directory Structure

```
frontend/
├── index.html                         # Entry HTML template
├── vite.config.js                     # Vite build & plugin configurations
├── eslint.config.js                   # ESLint configuration
├── .env.example                       # Environment variables template
├── package.json                       # Dependencies and build scripts
└── src/
    ├── main.jsx                       # Application root mount (React.StrictMode)
    ├── App.jsx                        # App shell, auth checking, presence management
    ├── assets/
    │   └── logo/
    │       └── classio-logo.png       # Classio brand mark
    ├── components/
    │   ├── VideoCallOverlay.jsx       # Floating / Fullscreen WebRTC call interface
    │   ├── common/
    │   │   └── CreateRoomModal.jsx    # Modal popup to create new public/private room
    │   └── layout/
    │       └── AppShell.jsx           # Top navigation bar, avatar, tab switches
    ├── contexts/
    │   └── CallContext.jsx            # WebRTC state machine, media stream controller
    ├── pages/
    │   ├── Login.jsx                  # Login view with error states and validation
    │   ├── Register.jsx               # Registration view
    │   ├── Messages.jsx               # Conversation list sidebar + active room container
    │   ├── ChatRoom.jsx               # Active chat room, message feed, receipts, calls
    │   └── Rooms.jsx                  # Public room discovery feed and join requests
    ├── services/
    │   └── socket.js                  # Socket.IO client singleton instance
    └── styles/
        ├── globals.css                # Dark theme design system, CSS tokens, resets
        ├── responsive.css             # Tablet and mobile breakpoints
        └── typing.css                 # Bouncing dot animation for typing indicators
```

---

## Application Architecture

### Bootstrap & Authentication Lifecycle

```
App Mount (src/App.jsx)
        │
        ├── 1. Dispatches POST /api/v1/auth/refresh (credentials: "include")
        │       ├── Success: Sets currentUser, setIsLoggedIn(true)
        │       └── Failure: Remains logged out, displays Login screen
        │
        └── 2. When logged in:
                ├── Connects Socket.IO singleton (src/services/socket.js)
                ├── Listens for "online-users", "user-online", "user-offline"
                └── Mounts CallProvider and AppShell
```

---

### Presence & Socket State Synchronization

- **Socket Singleton**: Initialized in `src/services/socket.js` with `autoConnect: false` and `withCredentials: true`.
- **Automatic Connect/Disconnect**: Managed by an effect in `App.jsx` tied directly to `isLoggedIn`.
- **Global Presence Set**: A React `Set<userId>` tracks online users in real time.

---

### Layout & Navigation Flow

The top-level `AppShell` provides tab-based navigation:

- **`messages`**: Opens the active messaging view (`Messages.jsx`), displaying the user's joined room list and the selected room's chat feed (`ChatRoom.jsx`).
- **`rooms`**: Opens the campus discovery view (`Rooms.jsx`), listing public and private rooms available to join.
- **`+ Create Room`**: Opens `CreateRoomModal.jsx` from anywhere in the app.

---

## Features & Modules

### 1. Authentication Module (`Login.jsx`, `Register.jsx`)

- Sleek dark-mode interface with the Classio brand identity.
- Client-side form validation before submission.
- Supports smooth toggling between Login and Registration views without page reload.
- Secure session establishment using `httpOnly` cookies returned from the server.

---

### 2. App Navigation Shell (`AppShell.jsx`)

- Fixed top navigation header containing the Classio logo.
- Primary navigation tabs (`Messages`, `Rooms`).
- Action button to trigger room creation.
- User profile badge displaying the user's initial.
- Secure logout button triggering session invalidation and cookie clearing via `/api/v1/auth/logout`.

---

### 3. Real-Time Chat & Receipts (`Messages.jsx`, `ChatRoom.jsx`)

- **Conversation List**:
  - Displays all joined rooms with real-time active room highlights.
  - Search filter allowing instant search by room name.
- **Message Feed**:
  - Auto-scrolls to the newest message on receipt.
  - Supports loading message history with delivery and read statuses.
- **Receipts System**:
  - Automatically emits `mark-room-read` when entering a room.
  - Individual messages display receipt state:
    - Sent: Single grey checkmark.
    - Delivered: Double grey checkmark.
    - Read: Double blue checkmark.
  - Live updates pushed directly through Socket.IO `receipt-update` events.
- **Typing Indicators**:
  - Broadcasts `start-typing` on user input with a 2-second debounce timer before emitting `stop-typing`.
  - Animated bouncing dot indicator (`typing.css`) showing names of room members currently typing.
- **Room Administration & Member Management**:
  - View member list with roles (`owner`, `member`) and join dates.
  - Room owners can view, approve, or reject pending join requests for private rooms.
  - Room owners can update room details (name, description) or delete the room.
  - Members can leave the room at any time.
- **Message Editing & Deletion**:
  - Inline message editing and deletion for messages sent by the current user.

---

### 4. WebRTC Audio, Video & Screen Sharing (`CallContext.jsx`, `VideoCallOverlay.jsx`)

- Initiate 1-on-1 audio or video calls directly from any room member's profile.
- **Incoming Call Notification**: Modal popup allowing the user to Accept or Decline incoming calls with caller identity preview.
- **Live In-Call Controls**:
  - **Mute / Unmute Audio**: Toggles local audio track.
  - **Camera On / Off**: Toggles local video track and notifies peer via `camera-state`.
  - **Screen Sharing**: Calls `getDisplayMedia()` and swaps the video track on the active `RTCPeerConnection` sender in real time. Automatically reverts to webcam if screen sharing stops.
  - **Minimize / Expand**: Minimizes the call into a compact floating preview window so students can continue reading or sending text messages while on the call.
  - **End Call**: Tears down media tracks, closes peer connection, and notifies peer.

---

### 5. Room Discovery & Creation (`Rooms.jsx`, `CreateRoomModal.jsx`)

- **Discover Campus Rooms**:
  - Displays cards for all public and private rooms the user has not yet joined.
  - Filter rooms dynamically by name or description.
- **One-Click Joining**:
  - Public rooms: Instantly joins the room and redirects to chat.
  - Private rooms: Sends a join request and notifies the room owner.
- **Room Creation**:
  - Name, description, and privacy toggle (`Public` vs `Private`).
  - Automatically establishes owner status and redirects to the new room upon creation.

---

## WebRTC Architecture & Implementation

The WebRTC engine lives inside `CallContext.jsx` and provides a clean hook `useCallContext()`:

```
[User A]                               [Backend Signaling]                          [User B]
   │                                            │                                      │
   ├────── call-user { to: B, callType } ──────>│                                      │
   │                                            ├────── incoming-call { from: A } ────>│
   │                                            │                                      │
   │                                            │<───── accept-call { to: A } ─────────┤
   │<───── call-accepted { from: B } ───────────┤                                      │
   │                                            │                                      │
   ├────── webrtc-offer { offer } ─────────────>│                                      │
   │                                            ├────── webrtc-offer { offer } ───────>│
   │                                            │                                      │
   │                                            │<───── webrtc-answer { answer } ──────┤
   │<───── webrtc-answer { answer } ────────────┤                                      │
   │                                            │                                      │
   ├────── ice-candidate ──────────────────────>│                                      │
   │                                            ├────── ice-candidate ────────────────>│
   │                                            │                                      │
   ========================== Peer-to-Peer Media Stream Active =========================
```

- **Dynamic ICE Servers**: On mount, `CallContext` fetches recommended STUN/TURN configurations from `/api/v1/room/ice-servers`.
- **ICE Candidate Queuing**: Any candidates arriving before `setRemoteDescription` completes are held in a pending queue and added sequentially once the description is set.
- **Track Substitution**: Screen sharing utilizes `RTCRtpSender.replaceTrack()` without requiring renegotiation.
- **Network Recovery**: If a socket disconnects briefly during a call, `call-recovery-ready` is triggered on reconnection to safely restore the call.

---

## Design System & Styling

The frontend utilizes a customized Vanilla CSS design system:

- **`globals.css`**: Defines CSS variables for dark-mode palettes:
  - Backgrounds: `--color-background` (`#0f172a`), `--color-bg-secondary` (`#1e293b`), `--color-bg-elevated` (`#334155`).
  - Accents: `--color-primary` (`#6366f1` / Indigo), `--color-accent` (`#38bdf8` / Cyan).
  - Status: `--color-success` (`#22c55e`), `--color-danger` (`#ef4444`).
  - Glassmorphic card styling, custom dark scrollbars, accessible form inputs, and modal backdrops.
- **`responsive.css`**: Provides layout adaptations for screens under `768px` and `1024px`.
- **`typing.css`**: Micro-animations for the bouncing dots typing status.

---

## Environment Configuration

Create a `.env` file in the `frontend/` directory:

```env
# URL pointing to the Classio backend
VITE_API_URL=http://localhost:5000
```

---

## Getting Started

1. **Install dependencies**:
   ```bash
   npm install
   ```
2. **Configure environment**:
   ```bash
   cp .env.example .env
   ```
3. **Start the development server**:
   ```bash
   npm run dev
   ```
4. **Open in browser**: Visit `http://localhost:5173`.

---

## Available Scripts

- `npm run dev`: Starts Vite dev server with Hot Module Replacement.
- `npm run build`: Compiles production build to the `dist/` directory.
- `npm run preview`: Locally previews the production build.
- `npm run lint`: Runs ESLint across the codebase.
- `npm run format`: Formats all files using Prettier according to `.prettierrc`.
