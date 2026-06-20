# Docker Setup Guide

## Overview

This project runs two Docker containers orchestrated via Docker Compose:

| Service | Container name | Image              | Role                     |
|---------|---------------|--------------------|--------------------------|
| `db`    | `inventory-db`  | `postgres:18`      | PostgreSQL 18 database   |
| `web`   | `inventory-web` | `dashboard-flask-web` | Flask + Socket.IO app |

---

## Quick Start

```bash
docker compose up -d --build
```

Open http://localhost:8000.

Stop everything:

```bash
docker compose down
```

---

## File Reference

### `docker-compose.yml`

```yaml
services:

  db:
    image: postgres:18
    container_name: inventory-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: inventory_db
    ports:
      - "5432:5432"
    volumes:
      - ./vol/psql-18:/var/lib/postgresql
      - ./setup_database.sql:/docker-entrypoint-initdb.d/01-setup.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d inventory_db"]
      interval: 5s
      timeout: 3s
      retries: 10

  web:
    build: .
    container_name: inventory-web
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/inventory_db
      FLASK_DEBUG: "1"
    volumes:
      - .:/app
    depends_on:
      db:
        condition: service_healthy
```

#### `db` service

- **Image**: `postgres:18` — official PostgreSQL 18 image.
- **Environment**: sets user, password, and the default database (`inventory_db`).
- **Ports**: exposes `5432` on the host so you can connect with any PSQL client.
- **Volumes**:
  - `./vol/psql-18:/var/lib/postgresql` — **bind mount** for persistent data. PG 18+ uses a version-specific subdirectory under this path (`18/`). The mount point is `/var/lib/postgresql` (not `/var/lib/postgresql/data`) as required by the PG 18+ Docker images.
  - `./setup_database.sql:/docker-entrypoint-initdb.d/01-setup.sql` — auto-runs the SQL script on **first database initialization**. Creates the `inventory` table, trigger function, and trigger. Only executes if the database is being created for the first time (i.e. `vol/psql-18` is empty).
- **Healthcheck**: runs `pg_isready` every 5 seconds. The `web` service waits for this to pass before starting.

#### `web` service

- **Build**: builds from the `Dockerfile` in the project root.
- **Ports**: exposes `8000` on the host.
- **Environment**:
  - `DATABASE_URL` — connects to the `db` service over the internal Docker network using the service name as hostname.
  - `FLASK_DEBUG=1` — enables Flask's debug mode (auto-reload on code changes).
- **Volumes**: `.:/app` — bind mounts the entire project directory. Any local edit triggers the Flask reloader (**hot reloading**).
- **Depends on**: waits for the `db` healthcheck to pass before starting, avoiding race conditions.

### `Dockerfile`

```dockerfile
FROM python:3.14-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc build-essential && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
```

- Base: `python:3.14-slim` — minimal Python 3.14 image.
- Build deps: `libpq-dev`, `gcc`, `build-essential` — required to compile `psycopg2-binary` from source (the slim image lacks a C compiler and `assert.h` otherwise).
- Dependencies installed **before** copying the full source (layers caching — if `requirements.txt` doesn't change, pip doesn't re-run).
- `CMD` runs `python main.py` directly. The app reads `DATABASE_URL` and `FLASK_DEBUG` from the environment.

### `.dockerignore`

```
vol/
venv/
__pycache__/
*.pyc
.git/
.gitignore
doc/
README.md
```

Prevents the Docker build context from including the PostgreSQL data directory (`vol/`, permission issues), local virtual environment, and git history. Keeps builds fast and clean.

### `setup_database.sql`

Mounted as an **init script** at `/docker-entrypoint-initdb.d/01-setup.sql`. PostgreSQL executes all `.sql` files in this directory the first time the database cluster is initialized (i.e., when `vol/psql-18/` is empty). This creates:

1. `inventory` table
2. `notify_inventory_changes()` trigger function (sends JSON payloads to `inventory_channel` via `pg_notify`)
3. `inventory_trigger` (fires on INSERT/UPDATE/DELETE)

On subsequent starts, the script is **skipped** because the data directory already exists.

---

## Hot Reloading

The `web` service uses two mechanisms together:

1. **Bind mount**: `.:/app` — the container runs the code **from your host filesystem**, not a frozen image.
2. **`FLASK_DEBUG=1`** — enables the Werkzeug reloader. When any Python, HTML, or JS file changes on the host, the Flask process restarts automatically.

No rebuild or restart needed during development.

---

## Networking

Docker Compose creates a default bridge network (`dashboard-flask_default`). Services communicate by service name:

- From `web` → `db`: hostname `db`, port `5432`
- `DATABASE_URL=postgresql://postgres:postgres@db:5432/inventory_db`

The `db` service also binds to host port `5432`, allowing external tools (e.g., `psql`, DBeaver) to connect via `localhost`.

---

## Volumes & Persistence

| Mount                     | Host path               | Container path                  | Purpose                        |
|---------------------------|-------------------------|---------------------------------|--------------------------------|
| `db` bind mount           | `./vol/psql-18`         | `/var/lib/postgresql`           | PG data files                  |
| `db` init script          | `./setup_database.sql`  | `/docker-entrypoint-initdb.d/`  | First-run schema setup         |
| `web` bind mount          | `.` (project root)      | `/app`                          | Source code (hot reload)       |

The PostgreSQL data persists in `vol/psql-18/18/` across container restarts. To reset the database:

```bash
docker compose down
rm -rf vol/psql-18
docker compose up -d
```

---

## Useful Commands

```bash
# Build and start everything
docker compose up -d --build

# View logs
docker compose logs -f
docker compose logs -f web
docker compose logs -f db

# Restart a single service
docker compose restart web

# Stop without removing volumes
docker compose stop

# Full cleanup (removes containers + network, keeps volumes)
docker compose down

# Full cleanup including volumes (WARNING: deletes database)
docker compose down -v

# Enter the web container
docker compose exec web bash

# Run psql against the database
docker compose exec db psql -U postgres -d inventory_db

# Rebuild without cache
docker compose build --no-cache web
```

---

## Troubleshooting

**`could not translate host name "db"`** — the `web` container cannot resolve the `db` hostname. Usually a transient DNS issue. Run `docker compose down && docker compose up -d` to recreate the network.

**Port 8000 already in use** — another process is using the port. Kill it or change the host port in `docker-compose.yml`:

```yaml
ports:
  - "8001:8000"  # host:8001 -> container:8000
```

**Port 5432 already in use** — a local PostgreSQL or another Docker container occupies the port. Stop the system PostgreSQL or use a different host port.

**Permission denied on `vol/psql-18`** — the PG data directory is owned by root (from the container). This is expected. The `.dockerignore` excludes it from the build context. Do not `chmod` it.

**Hot reload not working** — ensure `FLASK_DEBUG=1` is set and the `.:/app` bind mount is present. Changes outside the project root won't trigger reloads.
