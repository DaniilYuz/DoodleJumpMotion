

# Arena Service

Arena Service is a microservice responsible for managing competitive gaming arenas (leagues/tiers) based on player cup counts. It provides arena configuration and access control for a multiplayer game ecosystem.

## Overview

This service implements a tiered arena system where players are matched to specific arenas based on their accumulated cup count (trophy/achievement points). Each arena defines a cup range (minimum and maximum), creating a competitive ladder system.

## Features

- **Arena Management**: CRUD operations for arena configurations (admin-only)
- **Cup-Based Matching**: Automatic arena resolution based on player cup count
- **Role-Based Access Control**: Separate permissions for players and administrators
- **User Service Integration**: Communicates with external User Service to fetch player data
- **JWT Authentication**: Secure API access with role and cup count claims

## Architecture

The service follows a layered architecture pattern:

| Layer | Responsibility |
|-------|---------------|
| **Handler** (HTTP) | Gin-based REST API endpoints, request validation |
| **Service** | Business logic, user verification, arena resolution |
| **Repository** | PostgreSQL data access via GORM |
| **Domain** | Core entities (Arena, User) |





# Motion-Controlled Doodle Jump Game

A web-based multiplayer "Doodle Jump" style game controlled by body movements via webcam. Players jump on platforms by physically jumping in front of the camera, and move left/right by leaning, creating an immersive motion-controlled gaming experience.

## Overview

This is a React Native Web application that uses computer vision (MediaPipe Pose Landmarker) to track player body movements in real-time. The game features multiplayer support via WebSocket connections, arena-based progression with different physics configurations, and a seeded platform generation system for fair competitive play.

## Key Features

- **Motion Controls**: Full body tracking using webcam - jump to make the character jump, lean left/right to move
- **Multiplayer Support**: Real-time competitive play with score synchronization via WebSocket
- **Arena System**: Multiple arenas with unique physics (gravity, jump height, movement speed) and visual themes
- **Seeded Generation**: Deterministic platform placement using seed-based random generation for fair competition
- **Game Shell Integration**: Embeddable iframe architecture with parent window communication
- **Camera Toggle**: Players can enable/disable camera controls with 'C' key

## Architecture

The application follows a modular React architecture with clear separation of concerns:

| Module | Responsibility |
|--------|---------------|
| **GameScreen** | Main game loop, physics engine, rendering |
| **PoseContext** | Global state for body tracking coordinates and jump detection |
| **usePoseLandmarker** | MediaPipe integration for real-time pose detection |
| **useSeededPlatforms** | Deterministic platform generation using LCG random |
| **gameShell** | Parent window controller, WebSocket client, session management |
| **SeedModule** | Cross-window communication API (scores, deaths, initialization) |

## Tech Stack

- **Framework**: React Native (Web platform)
- **Animation**: React Native Reanimated 2 (Shared Values for 60fps performance)
- **Computer Vision**: MediaPipe Pose Landmarker (GPU-accelerated)
- **Build Tool**: Expo
- **Communication**: WebSocket (Session Service), postMessage API (iframe)




# Leaderboard Service

A high-performance leaderboard microservice managing player rankings by cup count (trophies/achievements). Provides real-time global leaderboards with dual-layer caching (Redis for speed, PostgreSQL for persistence) and automatic synchronization with the User Service.

## Overview

This service maintains a competitive ranking system where players are sorted by their accumulated cup count. It implements a hybrid storage architecture using Redis Sorted Sets for O(log N) ranking operations and PostgreSQL for persistent user data storage, ensuring both speed and data durability.

## Key Features

- **Real-time Rankings**: Instant rank calculation and top-N queries via Redis Sorted Sets
- **Dual-Layer Storage**: Redis cache + PostgreSQL persistence with automatic synchronization
- **Global Leaderboard**: Cross-arena player rankings based on total cup count
- **User Sync**: Background synchronization with User Service on startup
- **RESTful API**: Simple HTTP endpoints for score updates and leaderboard queries
- **CORS Support**: Pre-configured for web frontend integration

## Architecture

The service implements a layered architecture with repository pattern:

| Layer | Responsibility |
|-------|---------------|
| **Handler** (HTTP) | Gin-based REST API, request validation |
| **Service** | Business logic, cross-repository coordination, external sync |
| **Repository (PostgreSQL)** | Persistent user data, username resolution |
| **Repository (Redis)** | Sorted set operations, real-time rankings |
| **Models** | GORM entities and domain DTOs |

## Data Flow
User Service → Sync → PostgreSQL (persistent store)
↓
Leaderboard Service
↓
Redis (sorted set cache) ←→ API Responses




# Matchmaking Service

A high-performance matchmaking microservice that pairs players for competitive 1v1 matches based on their trophy count (cup count) and arena selection. Uses Redis Sorted Sets for efficient player queuing and NATS for event-driven match notifications.

## Overview

This service implements a skill-based matchmaking algorithm that groups players by arena (1-10) and trophy ranges (buckets of 100). Players are matched within their arena and adjacent trophy buckets to ensure fair competition. Once matched, the service creates a game session and notifies both players via polling and NATS events.

## Key Features

- **Skill-Based Matching**: Players queued by arena ID and trophy count buckets (100-cup ranges)
- **Redis-Backed Queue**: High-performance sorted set operations for O(log N) enqueue/dequeue
- **Fair Pairing**: Matches players within ±100 trophy range of their bucket
- **Session Management**: Automatic game session creation via Session Service
- **Real-time Notifications**: NATS pub/sub for match found events
- **Status Polling**: HTTP endpoint for clients to check match status
- **Anti-Cheat**: Trophy count verified against User Service (cannot be spoofed)

## Architecture

The service follows an event-driven microservices pattern:

| Component | Responsibility |
|-----------|--------------|
| **EnqueueHandler** | HTTP API for joining matchmaking queue |
| **StatusHandler** | Polling endpoint for match status checks |
| **QueueService** | Business logic, in-memory state management |
| **QueueRepo** | Redis sorted set operations (ZADD, ZRANGEBYSCORE, ZREM) |
| **ScannerService** | Background worker scanning for matchable pairs |
| **NatsPublisher** | Event publishing to NATS topics |
| **UserClient** | User Service integration for trophy verification |
| **SessionClient** | Session Service integration for game creation |

## Matchmaking Algorithm

### Queue Structure
Redis Key: arena:{arena_id}:{bucket}
arena_id: 1-10 (selected arena)
bucket: floor(trophies / 100) * 100 (0, 100, 200, ... 8000)
Score: trophy count (int)
Member: "{player_id}.{request_id}" (composite key)
### Matching Process

1. **Enqueue**: Player joins specific arena + trophy bucket
2. **Scan**: Every 50ms, scanner iterates all arenas and buckets
3. **Match**: Finds 2 players within bucket ± delta (100 trophies)
4. **Validate**: Ensures different player IDs (prevent self-matching)
5. **Create**: Calls Session Service to create game session
6. **Notify**: Updates in-memory status + publishes NATS events
7. **Deliver**: Players poll status endpoint to receive session details

Tech Stack
Language: Go
Web Framework: Gin
Queue Store: Redis (Sorted Sets)
Message Bus: NATS
Authentication: JWT (HS256)
HTTP Client: Standard library with context support

# Session Service

Real-time multiplayer game session manager handling WebSocket connections, in-game events, and competitive match lifecycle with cup-based ranking adjustments.

## Overview

This service manages active 1v1 game sessions from player connection through match completion. It handles real-time score synchronization, player death tracking, session state transitions, and post-match cup calculations integrated with the User Service.

## Tech Stack

- **Language**: Go
- **Web Framework**: Gin
- **WebSocket**: Gorilla WebSocket
- **Database**: PostgreSQL (GORM)
- **Architecture**: Hub-pattern for connection management, in-memory score tracking

## Architecture

| Component | Responsibility |
|-----------|--------------|
| **WSHandler** | WebSocket upgrade, message routing, game loop coordination |
| **Hub** | Connection registry, broadcast messaging, room management |
| **SessionService** | Session CRUD, database transactions, cup calculation logic |
| **SessionRepository** | GORM-based session persistence |
| **UserClient** | HTTP client for User Service cup updates |
| **SessionScores** | Thread-safe in-memory score tracking per session |

Session Lifecycle
waiting → active → finished

1. Create: Matchmaking creates session (status: waiting)
2. Join: Both players connect via WebSocket → status: active
3. Play: Real-time score/death updates
4. End: Both dead or 2x score lead → status: finished
5. Finalize: Cup updates + match record + broadcast

Key Features
Atomic Updates: Database transactions for session state changes
Connection Safety: Deferred cleanup, graceful goroutine shutdown
Score Deduplication: Only higher scores update (anti-cheat)
Self-Match Prevention: Player ID validation in matching
Real-time Sync: Sub-100ms score propagation via WebSocket




# User Service

Core user management microservice handling authentication, player progression, and cup-based ranking system with automatic arena assignment.

## Overview

This service manages user accounts, authentication via JWT, and the competitive progression system based on cup count (trophies). It automatically assigns players to arenas based on their cup count ranges and syncs ranking updates to the Leaderboard Service.

## Tech Stack

- **Language**: Go
- **Web Framework**: Gin
- **Database**: PostgreSQL (GORM)
- **Authentication**: JWT (HS256), bcrypt password hashing
- **Security**: Role-based access (player/admin), internal API tokens

## Architecture

| Component | Responsibility |
|-----------|--------------|
| **AuthHandler** | Registration, login, cup updates, user queries |
| **UserService** | Business logic, JWT generation, password hashing |
| **UserRepository** | GORM-based user CRUD operations |
| **Middleware** | JWT validation, internal auth, role checks |

Key Features
Auto Arena Calculation: CurrentArenaID updates automatically on cup changes
Anti-Decrement Protection: CupCount floor at 0
Async Leaderboard Sync: Non-blocking notification on cup updates
Migration Safety: Boot-time arena recalculation for legacy users
Eternal Admin Token: Generated on startup for service auth




# Game Frontend

Simple web frontend for player interaction with the Doodle Jump game backend services. Provides UI for authentication, matchmaking, leaderboards, and game session embedding.

## Purpose

Client-side interface connecting players to the game microservices:
- User Service (auth, profiles)
- Arena Service (arena data)
- Matchmaking Service (queue, match status)
- Session Service (WebSocket game connection)
- Leaderboard Service (rankings)

## Tech Stack

- **Vanilla JavaScript** (no framework)
- **HTML/CSS** static pages
- **WebSocket** for real-time game communication
- **JWT** stored in localStorage

## Structure

| File | Purpose |
|------|---------|
| `index.js` | Main dashboard, arena display, matchmaking UI |
| `login.js` / `register.js` | Authentication flows |
| `leaderboard.js` | Top-10 and personal ranking display |
| `gameShell.js` | WebSocket proxy between iframe game and Session Service |
| `config.js` | API endpoint configuration |

## Key Features

- **JWT Auth**: Login/register, token persistence in localStorage
- **Arena Browser**: Visual arena selection with cup-based unlocking
- **Matchmaking**: Queue polling, opponent matching, session redirect
- **Game Embedding**: Iframe game loader with seed/userId injection
- **Real-time Proxy**: WebSocket message forwarding (game ↔ Session Service)
- **Leaderboard**: Top players and personal rank display
