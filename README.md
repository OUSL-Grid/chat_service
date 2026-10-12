# Chat Service

Real-time chat microservice for the **Campus Marketplace & Skill Share** platform.

It lets two students talk about a **listing inquiry** or a **skill trade request**, and lets them share a one-time **meetup pin** (e.g. "Library front entrance") so they can meet safely on campus.

> **Status:** 🚧 In development (university group project)
> **Language:** Python 3.11+ (the original design suggested Go; we use Python for this service, but keep the same API contract and events so it stays compatible with the other services).

---

## What this service does

- Opens a **chat room** for each listing inquiry or trade request.
- Sends and receives messages in real time using **WebSockets**.
- Stores message history so users can reload a conversation.
- Tracks **presence** (online/offline) using Redis.
- Lets a user share a **point-in-time meetup pin**. Pins are temporary and auto-delete after a set time.
- Publishes and listens to events on **NATS** so it does not need to call other services directly.

### What this service does NOT do

- It does not verify emails or issue tokens (that is the **Auth Service**).
- It does not create listings or trades (that is **Listings** / **Skill Exchange**).
- It does not track continuous location. Only user-initiated, expiring pins.

---

## Tech stack

| Purpose | Tool |
|---|---|
| Web framework + WebSockets | FastAPI |
| Server | Uvicorn |
| Database | PostgreSQL (via SQLAlchemy + Alembic migrations) |
| Presence / pub-sub | Redis |
| Events between services | NATS (`nats-py`) |
| Token validation | PyJWT (RS256, public key only) |
| Tests | pytest |
| Containers | Docker + Docker Compose |

---

## Project structure

```
chat-service/
├── app/
│   ├── main.py            # App entry point
│   ├── config.py          # Settings read from environment variables
│   ├── auth.py            # Validates JWTs issued by the Auth Service
│   ├── db.py              # Database connection/session
│   ├── models.py          # chat_rooms, messages, meetup_pins tables
│   ├── schemas.py         # Request/response shapes (Pydantic)
│   ├── routes/
│   │   ├── rooms.py       # REST: create/list rooms, message history
│   │   ├── pins.py        # REST: create/list meetup pins
│   │   └── health.py      # Health check
│   ├── websocket/
│   │   ├── manager.py     # Tracks connected users per room
│   │   └── handlers.py    # WebSocket endpoint logic
│   ├── presence.py        # Redis online/offline tracking
│   └── events.py          # NATS publish/subscribe
├── migrations/            # Alembic migration files
├── tests/
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

## Data this service owns

Matches the shared system design:

```
chat_rooms(id, context_type[listing|trade], context_id, created_at)
messages(id, room_id, sender_id, body, created_at)
meetup_pins(id, room_id, lat, lng, label, expires_at)
```

Rules:
- One room per `trade_request` or `listing inquiry`.
- Only the two participants of a room can read or write in it.
- `meetup_pins` rows are deleted after `expires_at`.

---

## Getting started

### 1. Prerequisites

- Python 3.11 or newer
- Docker and Docker Compose
- Git

### 2. Clone and set up

```bash
git clone <your-repo-url>
cd chat-service

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

### 3. Configure environment

```bash
cp .env.example .env
```

Then edit `.env` (see the table below).

### 4. Start dependencies (Postgres, Redis, NATS)

```bash
docker compose up -d
```

### 5. Run database migrations

```bash
alembic upgrade head
```

### 6. Run the service

```bash
uvicorn app.main:app --reload --port 8003
```

Open http://localhost:8003/docs to see the interactive API docs.

---

## Environment variables

| Variable | Example | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+psycopg://user:pass@localhost:5432/marketplace` | Postgres connection |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection |
| `NATS_URL` | `nats://localhost:4222` | NATS connection |
| `JWT_PUBLIC_KEY_PATH` | `./keys/public.pem` | Auth Service public key (RS256) |
| `JWT_ALGORITHM` | `RS256` | Token signing algorithm |
| `PIN_TTL_HOURS` | `4` | How long a meetup pin lives |
| `PORT` | `8003` | Service port |

Never commit your real `.env` file or any private keys.

---

## API overview

### REST

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Service health check |
| POST | `/rooms` | Create a room for a listing or trade |
| GET | `/rooms` | List rooms for the current user |
| GET | `/rooms/{room_id}/messages` | Message history (paginated) |
| POST | `/rooms/{room_id}/pins` | Share a meetup pin |
| GET | `/rooms/{room_id}/pins` | Active (non-expired) pins |

All endpoints except `/health` require `Authorization: Bearer <access_token>`.

### WebSocket

```
ws://localhost:8003/ws/rooms/{room_id}?token=<access_token>
```

Client sends:
```json
{ "type": "message", "body": "Hi, is the textbook still available?" }
```

Server sends:
```json
{ "type": "message", "id": "...", "sender_id": "...", "body": "...", "created_at": "..." }
{ "type": "presence", "user_id": "...", "status": "online" }
{ "type": "pin", "label": "Library front entrance", "lat": 6.9, "lng": 79.8 }
```

---

## Events (NATS)

| Direction | Subject | When |
|---|---|---|
| Subscribe | `trade.accepted` | Create a chat room for the trade |
| Subscribe | `listing.inquiry.created` | Create a chat room for the inquiry |
| Subscribe | `listing.removed` | Close related rooms |
| Publish | `chat.message.sent` | After a message is saved (used by Moderation) |
| Publish | `chat.room.created` | After a room is created |

> Subject names must match the shared event contract agreed with the team. If the team changes them, update this table.

---

## Authentication

This service **does not issue tokens**. It only validates JWTs created by the Auth Service, using the Auth Service's **public key**. From the token it reads `user_id`, `role` and `verified_domain`. Requests with missing, expired or invalid tokens are rejected with `401`.

---

## Running tests

```bash
pytest
```

---

## Safety & privacy

- No continuous location tracking, only one-time pins.
- Pins expire automatically.
- Only room participants can access a room.
- Message rate limiting to reduce spam.
- Messages can be flagged through the Moderation Service.

---

## Roadmap

- [ ] Project setup and health check
- [ ] JWT validation
- [ ] Database models and migrations
- [ ] Room REST endpoints
- [ ] WebSocket messaging
- [ ] Redis presence
- [ ] Meetup pins with expiry
- [ ] NATS events
- [ ] Rate limiting
- [ ] Tests and Docker image
- [ ] (Nice to have) Suggested safe meetup spots list

---

## Team

| Service | Owner |
|---|---|
| Chat Service | _your name_ |
| Auth + Gateway | _name_ |
| Listings + Moderation | _name_ |
| Skill Exchange | _name_ |

---

## License

For academic use as part of a university project.
