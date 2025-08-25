# Technical Documentation

## Overview
A collaborative whiteboard enabling multiple users to draw together in real time. The system is composed of a React frontend using HTML5 Canvas with Rough.js and Perfect Freehand, a Node.js/Express backend with Socket.IO for real-time transport, and MongoDB (Mongoose) for persistence. Authentication uses JWT with bcrypt for password hashing. Access to canvases is restricted to the owner and explicitly shared users across both REST endpoints and Socket.IO events.

## System Architecture
- Client (React)
  - Canvas rendering: `components/Board` uses HTML5 Canvas with Rough.js for primitives and Perfect Freehand for brush strokes
  - State management: Context API providers (`BoardProvider`, `ToolboxProvider`) with reducers and hooks (`useReducer`, `useContext`, `useCallback`, `useEffect`)
  - Tools: brush, line, rectangle, circle, arrow, text, eraser; undo/redo; download as image
  - Networking: Axios for REST, Socket.IO client for low-latency collaboration
- Server (Express + Socket.IO)
  - REST: Auth and canvas CRUD with JWT middleware
  - WebSocket: Room-per-canvas with authorization using the same JWT. Emits/relays drawing updates and initializes clients with the latest elements
  - Persistence: Canvas elements stored in MongoDB; server also caches in-memory per-canvas elements for quick fan-out
- Database (MongoDB)
  - `User`: `email`, `password` (bcrypt hash)
  - `Canvas`: `owner`, `shared[]`, `elements[]`, `createdAt`

## Authentication and Authorization
- Registration (`POST /api/users/register`): stores email and bcrypt-hashed password
- Login (`POST /api/users/login`): issues JWT signed with `SECRET_KEY` containing `{ userId }`, 7d expiry
- Middleware (`authMiddleware`): validates `Authorization: Bearer <token>`, sets `req.userId`
- Access Control:
  - REST: Owner or any `shared` user can load/update; only owner can share/unshare/delete
  - Socket.IO: On `joinCanvas`, token from `Authorization` header in handshake must resolve to a user who is owner or in `shared` array; otherwise server emits `unauthorized`

## Realtime Collaboration Flow
1. Client loads route `/:id` and requests canvas via REST to seed state and history
2. Client emits `joinCanvas` with `canvasId`; server verifies JWT and membership, joins the socket to the room
3. Server emits `loadCanvas` with latest elements (in-memory cache or DB)
4. Client draws; for every move/up, client emits `drawingUpdate` with `{ canvasId, elements }`
5. Server updates in-memory `canvasData[canvasId]`, broadcasts `receiveDrawingUpdate` to the room, and writes elements to MongoDB

## REST API
Base URL: `/api`

### Users
- POST `/users/register`
  - Body: `{ email, password }`
  - 201: `{ message }`
- POST `/users/login`
  - Body: `{ email, password }`
  - 200: `{ message, token }`
- GET `/users/me` (auth)
  - Header: `Authorization: Bearer <token>`
  - 200: user object without password

### Canvas
- POST `/canvas/create` (auth)
  - 201: `{ message, canvasId }`
- PUT `/canvas/update` (auth)
  - Body: `{ canvasId, elements }`
  - 200: `{ message }`
- GET `/canvas/load/:id` (auth)
  - 200: canvas document
- PUT `/canvas/share/:id` (owner only)
  - Body: `{ email }`
  - 200: `{ message }`
- PUT `/canvas/unshare/:id` (owner only)
  - Body: `{ userIdToRemove }`
  - 200: `{ message }`
- DELETE `/canvas/delete/:id` (owner only)
  - 200: `{ message }`
- GET `/canvas/list` (auth)
  - 200: `[canvas]` owned or shared

## Socket.IO Events
- Client → Server: `joinCanvas` `{ canvasId }`
  - Server validates `Authorization: Bearer <token>` from handshake headers
  - On success: socket joins `room(canvasId)`, server emits `loadCanvas` with elements
  - On failure: emits `unauthorized` with message
- Client → Server: `drawingUpdate` `{ canvasId, elements }`
  - Server caches, persists, and broadcasts `receiveDrawingUpdate` to others in room
- Server → Client: `loadCanvas` `[elements]`
- Server → Client: `receiveDrawingUpdate` `[elements]`
- Server → Client: `unauthorized` `{ message }`

## Frontend Modules
- `components/Board`: Canvas rendering and event handling; subscribes to socket events and emits updates; keyboard shortcuts for undo/redo; initializes from REST
- `components/Toolbar`: Tool selection and undo/redo/download; uses `BoardContext`
- `components/Toolbox`: Stroke/fill/size controls based on active tool; uses `ToolboxContext`
- `store/BoardProvider`: Reducer handling draw/erase/text/undo/redo with history tracking
- `store/ToolboxProvider`: Provides color/size state per tool
- `utils/api`: Axios functions to `load` and `update` canvas
- `utils/socket`: Socket.IO client configured with `Authorization` header

## Data Models
```ts
User {
  _id: ObjectId,
  email: string,
  password: string // bcrypt hash
}

Canvas {
  _id: ObjectId,
  owner: ObjectId<User>,
  shared: ObjectId<User>[],
  elements: any[],
  createdAt: Date
}
```

## Configuration
- Environment variables (recommended):
  - `MONGODB_URI`, `SECRET_KEY`, `PORT`, `CORS_ORIGIN`
- Frontend base URLs:
  - `src/utils/api.js` → `API_BASE_URL`
  - `src/utils/socket.js` → Socket server URL

## Security Considerations
- Replace hard-coded `SECRET_KEY` and DB URI with environment variables
- Align CORS origins to deployed frontend domains
- Use HTTPS in production; secure token storage and transmission
- Validate canvas membership on both WebSocket and REST paths

## Local Development
- Backend: `cd backend && npm install && npm start` (defaults to 5000)
- Frontend: `cd frontend && npm install && npm start` (defaults to 3000)
- Update frontend URLs to point to `http://localhost:5000`

## Known Limitations
- In-memory server cache for canvas elements is ephemeral; consider Redis for scale
- High-frequency updates could be batched/throttled to reduce write frequency
- Missing granular roles beyond owner/shared

## Portfolio / CV Highlights
- Built a full stack whiteboard app using React, Socket.IO and MongoDB supporting real-time collaboration
- Secured the app with JWT-based authentication and bcrypt, using password hashing, token validation middleware and access control on REST APIs and Socket.IO events to ensure only authorized users can modify shared canvas
- Built interactive whiteboard tools using React hooks (useReducer, useContext, useCallback, useEffect) with HTML5 Canvas, Rough.js and Perfect Freehand, supporting freehand brush, shapes, text, eraser and robust undo/redo
