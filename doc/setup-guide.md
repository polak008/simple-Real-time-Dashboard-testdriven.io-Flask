# Real-Time Inventory Dashboard — Setup Guide

## Prerequisites

- Docker (with compose plugin) — https://docs.docker.com/engine/install/
- Python 3.12+
- Git (optional)

---

## Quick Start

```bash
# 1. Clone the project
git clone <repo-url> dashboard-flask
cd dashboard-flask

# 2. Start PostgreSQL 18 with Docker (bind mount for persistence)
mkdir -p vol/psql-18

docker run -d \
  --name inventory-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=inventory_db \
  -p 5432:5432 \
  -v $(pwd)/vol/psql-18:/var/lib/postgresql \
  postgres:18

# 3. Run the database setup (create tables, trigger function, trigger)
docker exec -i inventory-db psql -U postgres -d inventory_db < setup_database.sql

# 4. Install Python dependencies
pip install -r requirements.txt

# 5. Start the Flask app
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/inventory_db" python3 main.py
```

Visit **http://localhost:8000** in your browser.

---

## Project Structure

```
dashboard-flask/
├── main.py               # Flask app entry point, routes, Socket.IO handlers
├── database.py           # SQLAlchemy init, table creation, DB URL helper
├── models.py             # Inventory ORM model
├── notify.py             # PostgreSQL LISTEN/NOTIFY listener (threaded)
├── setup_database.sql    # SQL schema, trigger function, and trigger
├── requirements.txt      # Python dependencies
├── static/
│   ├── index.html        # Dashboard frontend HTML + CSS
│   └── index.js          # Dashboard frontend JS (Socket.IO client)
├── vol/psql-18/          # PostgreSQL 18 data directory (bind mount)
└── doc/
    └── setup-guide.md    # This file
```

---

## Detailed Steps

### 1. PostgreSQL 18 via Docker

Pull and run:

```bash
docker pull postgres:18

docker run -d \
  --name inventory-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=inventory_db \
  -p 5432:5432 \
  -v $(pwd)/vol/psql-18:/var/lib/postgresql \
  postgres:18
```

> **Note for PG 18+:** The data directory is mounted at `/var/lib/postgresql` (not `/var/lib/postgresql/data`). The image creates a version-specific subdirectory automatically.

Verify it's running:

```bash
docker ps --filter name=inventory-db
```

### 2. Database Schema

The `setup_database.sql` file creates:
- `inventory` table (`id`, `name`, `quantity`, `updated_at`)
- `notify_inventory_changes()` trigger function — sends JSON payload to `inventory_channel` via `pg_notify` on INSERT/UPDATE/DELETE
- `inventory_trigger` — fires the function after each row change

Run it:

```bash
docker exec -i inventory-db psql -U postgres -d inventory_db < setup_database.sql
```

### 3. Python Environment

```bash
pip install -r requirements.txt
```

**Dependencies:**

| Package              | Version |
|----------------------|---------|
| Flask                | 3.1.2   |
| Flask-SQLAlchemy     | 3.1.1   |
| Flask-SocketIO       | 5.5.1   |
| psycopg2-binary      | 2.9.10  |
| python-socketio      | 5.13.0  |
| python-engineio      | 4.12.2  |

### 4. Run the Application

```bash
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/inventory_db" python3 main.py
```

The app starts on **http://localhost:8000**.

**Environment variables:**

| Variable       | Default                                                   | Description              |
|----------------|-----------------------------------------------------------|--------------------------|
| `DATABASE_URL` | `postgresql://localhost/inventory_db`                     | PostgreSQL connection URI |

---

## API Reference

| Method   | Endpoint                          | Description          |
|----------|-----------------------------------|----------------------|
| `GET`    | `/`                               | Dashboard HTML page  |
| `GET`    | `/api/inventories`                | List all items       |
| `POST`   | `/api/inventories`                | Create an item       |
| `PUT`    | `/api/inventories/<id>`           | Update item quantity |
| `DELETE` | `/api/inventories/<id>`           | Delete an item       |

### Examples

```bash
# Create item
curl -s -X POST http://localhost:8000/api/inventories \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","quantity":10}'

# List items
curl -s http://localhost:8000/api/inventories

# Update quantity
curl -s -X PUT http://localhost:8000/api/inventories/1 \
  -H "Content-Type: application/json" \
  -d '{"quantity":8}'

# Delete item
curl -s -X DELETE http://localhost:8000/api/inventories/1
```

---

## How Real-Time Updates Work

1. A CRUD operation changes a row in the `inventory` table.
2. The Postgres trigger `inventory_trigger` fires and calls `notify_inventory_changes()`, which sends a JSON payload to the `inventory_channel` via `pg_notify`.
3. A background thread in `notify.py` listens on `inventory_channel` using `psycopg2` and `LISTEN`.
4. When a notification arrives, it is broadcast to all connected Socket.IO clients through Flask-SocketIO.
5. The frontend JavaScript (`index.js`) receives the event and updates the dashboard UI in real time.

---

## Useful Commands

```bash
# Stop PostgreSQL container
docker stop inventory-db

# Start it again
docker start inventory-db

# View container logs
docker logs inventory-db -f

# Enter psql shell
docker exec -it inventory-db psql -U postgres -d inventory_db

# Kill Flask app
pkill -f "python3 main.py"

# Check if Postgres port is open
ss -tlnp | grep 5432
```

---

## Troubleshooting

**Port 5432 already in use** — stop the system PostgreSQL or change the mapped port:

```bash
# Option A: stop system PostgreSQL
sudo systemctl stop postgresql

# Option B: use a different host port (update DATABASE_URL accordingly)
docker run -d ... -p 5433:5432 ...
# DATABASE_URL="postgresql://postgres:postgres@localhost:5433/inventory_db"
```

**psycopg2 installation fails** — install build dependencies:

```bash
sudo apt install python3-dev libpq-dev gcc
```

**"There appears to be PostgreSQL data in /var/lib/postgresql/data"** — the PG 18+ image changed the expected mount point. Mount at `/var/lib/postgresql` instead of `/var/lib/postgresql/data`.
