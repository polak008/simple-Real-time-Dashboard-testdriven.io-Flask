# Real-Time Inventory Dashboard

A real-time inventory tracking dashboard built with **Flask**, **PostgreSQL** (LISTEN/NOTIFY), and **Socket.IO**. Live inventory updates (add, update, delete) stream instantly to all connected browser tabs.

## Features

- RESTful API for inventory CRUD operations
- Real-time updates via PostgreSQL triggers + LISTEN/NOTIFY
- WebSocket broadcasting with Flask-SocketIO
- Responsive dashboard UI (vanilla HTML/CSS/JS)
- Threaded Postgres listener — non-blocking

## Quick Start

```bash
mkdir -p vol/psql-18

docker run -d --name inventory-db \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=inventory_db \
  -p 5432:5432 \
  -v $(pwd)/vol/psql-18:/var/lib/postgresql \
  postgres:18

docker exec -i inventory-db psql -U postgres -d inventory_db < setup_database.sql

pip install -r requirements.txt

DATABASE_URL="postgresql://postgres:postgres@localhost:5432/inventory_db" python3 main.py
```

Open **http://localhost:8000**.

## How It Works

```
Client action → HTTP API → DB change → PG trigger → NOTIFY
                                                        ↓
Dashboard UI ← Socket.IO ← broadcast ← PostgresListener (bg thread)
```

## API

| Method | Endpoint                     | Description    |
|--------|------------------------------|----------------|
| GET    | `/api/inventories`           | List all items |
| POST   | `/api/inventories`           | Create item    |
| PUT    | `/api/inventories/<id>`      | Update item    |
| DELETE | `/api/inventories/<id>`      | Delete item    |

## Project Layout

```
├── main.py              # App entry, routes, Socket.IO events
├── database.py          # SQLAlchemy init & helpers
├── models.py            # Inventory ORM model
├── notify.py            # Postgres LISTEN/NOTIFY threaded listener
├── setup_database.sql   # Schema, trigger function, trigger
├── requirements.txt
├── static/
│   ├── index.html       # Dashboard UI
│   └── index.js         # Socket.IO client
├── vol/psql-18/         # PG data (bind mount)
└── doc/setup-guide.md   # Full setup guide
```

---

Based on the tutorial [Developing a Real-time Dashboard with Flask, Postgres, and Socket.IO](https://testdriven.io/blog/flask-postgres-socketio/) on TestDriven.io.
