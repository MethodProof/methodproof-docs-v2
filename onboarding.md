# MethodProof — Developer Setup Guide

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Docker | 24+ | [docker.com/get-docker](https://docs.docker.com/get-docker/) |
| Docker Compose | v2+ | Included with Docker Desktop |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Node.js | 20+ | `brew install node` or [nodejs.org](https://nodejs.org/) |
| Neo4j | 5.x | Docker (recommended) or `brew install neo4j` |
| PostgreSQL | 15+ | Docker (recommended) or `brew install postgresql@15` |
| Redis | 7+ | Docker (recommended) or `brew install redis` |
| cypher-shell | 5.x | Included with Neo4j, or `brew install cypher-shell` |

---

## 1. methodproof-platform (API Server)

### 1.1 Start dependencies

```bash
# Start Neo4j, Postgres, Redis, and MinIO via Docker
docker run -d --name mp-neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/methodproof \
  neo4j:5

docker run -d --name mp-postgres \
  -p 5432:5432 \
  -e POSTGRES_USER=methodproof \
  -e POSTGRES_PASSWORD=methodproof \
  -e POSTGRES_DB=methodproof \
  postgres:15

docker run -d --name mp-redis \
  -p 6379:6379 \
  redis:7

docker run -d --name mp-minio \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  minio/minio server /data --console-address ":9001"
```

### 1.2 Apply Neo4j schema

```bash
cd methodproof-graph

# Wait for Neo4j to be ready (~10s)
until cypher-shell -u neo4j -p methodproof "RETURN 1" 2>/dev/null; do sleep 1; done

cat schema/constraints.cypher | cypher-shell -u neo4j -p methodproof
cat schema/indexes.cypher | cypher-shell -u neo4j -p methodproof
cat migrations/001_seed.cypher | cypher-shell -u neo4j -p methodproof
```

### 1.3 Configure environment

```bash
cd methodproof-platform

cat > .env <<'EOF'
NEO4J_URI=bolt://localhost:7687
NEO4J_PASSWORD=methodproof
DATABASE_URL=postgresql+asyncpg://methodproof:methodproof@localhost:5432/methodproof
JWT_SECRET=dev-secret-change-in-production
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=methodproof
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
LOG_LEVEL=DEBUG
SERVICE_NAME=methodproof-platform
NARRATION_PROVIDER=local
EOF
```

### 1.4 Install dependencies and run migrations

```bash
uv sync
uv run alembic upgrade head
```

### 1.5 Start the server

```bash
uv run uvicorn app.main:app --reload
# API at http://localhost:8000
# Health check: curl http://localhost:8000/health
```

### 1.6 Verify

```bash
curl http://localhost:8000/health
# {"status":"ok"}

curl http://localhost:8000/health/deep
# {"status":"ok","checks":{"neo4j":{"status":"ok",...},...}}
```

---

## 2. methodproof-runtime (Devcontainer)

### 2.1 Build the image

```bash
cd methodproof-runtime
docker build -t methodproof-runtime:dev .
```

### 2.2 Launch with docker compose

```bash
# Create .env for the runtime
cat > .env <<'EOF'
SESSION_TOKEN=test-session-token
PLATFORM_API_URL=http://host.docker.internal:8000
EOF

docker compose up
```

The devcontainer starts with:
- VS Code Server on port 8443
- MCP server on localhost:3000 (inside container)
- mitmproxy on localhost:8080 (inside container)
- Redis buffer for local telemetry
- FS watcher, terminal monitor, telemetry streamer

### 2.3 Verify

```bash
# Check container is running
docker compose ps

# Check entrypoint started all processes
docker compose exec devcontainer ls /opt/methodproof/pids/
```

---

## 3. methodproof-dashboard / methodproof-portal (React apps)

```bash
cd methodproof-dashboard  # or methodproof-portal
npm install
npm run dev
# Dashboard at http://localhost:5173
```

---

## 4. methodproof-extension (Chrome extension)

```bash
cd methodproof-extension
npm install
npm run build
# Load unpacked extension from dist/ in chrome://extensions
```

---

## Troubleshooting

### Neo4j connection refused

**Symptom:** `ServiceUnavailable: Failed to establish connection to ResolvedIPv4Address(('127.0.0.1', 7687))`

**Fixes:**
1. Check Neo4j is running: `docker ps | grep neo4j`
2. If not running: `docker start mp-neo4j`
3. Check port binding: `lsof -i :7687` — another process may hold the port
4. Check NEO4J_URI in `.env` matches the port (default: `bolt://localhost:7687`)
5. Wait for startup — Neo4j can take 10-15s to accept connections on first boot: `until cypher-shell -u neo4j -p methodproof "RETURN 1" 2>/dev/null; do sleep 1; done`
6. Check logs: `docker logs mp-neo4j --tail 30`

### Postgres migration failures

**Symptom:** `alembic upgrade head` fails with connection errors or SQL errors.

**Fixes:**
1. Check Postgres is running: `docker ps | grep postgres`
2. Verify `DATABASE_URL` in `.env` matches the container credentials
3. If "database does not exist": `docker exec mp-postgres createdb -U methodproof methodproof`
4. If "relation already exists" (re-running migrations): `uv run alembic downgrade base && uv run alembic upgrade head`
5. If alembic is not finding migrations: ensure you are in the `methodproof-platform/` directory
6. Check logs: `docker logs mp-postgres --tail 30`

### JWT decode errors

**Symptom:** `401 Unauthorized` with `"detail": "Invalid or expired token"` on protected endpoints.

**Fixes:**
1. Check `JWT_SECRET` in `.env` matches the secret used when the token was issued — if you changed the secret, all existing tokens are invalid
2. Tokens expire after 24 hours. Re-login: `curl -X POST http://localhost:8000/auth/login -H "Content-Type: application/json" -d '{"email":"...","password":"..."}'`
3. Ensure the Authorization header format is exactly `Bearer <token>` (capital B, space, no quotes around token)
4. Decode the token at jwt.io to check expiry (`exp` claim) and payload structure
5. If the token has no `company_id` claim, it was issued by an older schema — re-register or re-login

### MinIO / S3 connection errors

**Symptom:** `botocore.exceptions.EndpointConnectionError` or screenshot uploads fail.

**Fixes:**
1. Check MinIO is running: `docker ps | grep minio`
2. Create the bucket if it doesn't exist: `docker exec mp-minio mc mb /data/methodproof` or visit http://localhost:9001 (admin console)
3. Verify `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` in `.env`

### Redis connection errors

**Symptom:** `ConnectionError: Error connecting to localhost:6379`

**Fixes:**
1. Check Redis is running: `docker ps | grep redis`
2. If not running: `docker start mp-redis`
3. Test connection: `redis-cli ping` (should return `PONG`)
