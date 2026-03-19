# 🐳 Docker-Based 3-Tier Application
**FiftyFive Technologies · DevOps Intern Assessment**

---

## 📐 Architecture Diagram

```
[ Browser ]
     |
     ▼
[ Nginx :80 ]  ──── serves ────▶  index.html (static frontend)
     |
     | /api/* requests
     ▼
[ Node.js Backend :3000 ]
     |
     | TCP :3306
     ▼
[ MySQL 8.0 :3306 ]  ◀──── named volume: mysql_persistent_data
```

**Network:** All services communicate via custom bridge network `threeTier_network` using Docker service names (`db`, `backend`, `frontend`) — no hardcoded IPs.

---

## 🚀 Setup Instructions

### Prerequisites
- Docker Desktop (or Docker Engine + Docker Compose v2)
- Git

### Steps

```bash
# 1. Clone the repo
git clone <your-repo-url>
cd <repo-folder>

# 2. Create your .env file from example
cp .env.example .env
# Edit .env and set your passwords

# 3. Start everything
docker compose up --build
```

That's it. Single command: `docker compose up --build`

Access the app at: **http://localhost**

---

## 🗂 Repository Structure

```
.
├── frontend/
│   ├── Dockerfile              # Multi-stage, non-root user
│   ├── nginx.conf.template     # Dynamic BACKEND_URL via envsubst
│   ├── .dockerignore
│   └── index.html              # Static frontend with live health checks
├── backend/
│   ├── Dockerfile              # Multi-stage, non-root user
│   ├── .dockerignore
│   ├── app.js                  # Express API server
│   └── package.json
├── docker-compose.yml          # All services, health checks, volumes, network
├── .env.example                # Template — commit this, NOT .env
└── README.md
```

---

## ⚙️ How It Works

### 1. How Backend Waits for MySQL

The backend uses **two layers** of wait logic:

- **Docker Compose `depends_on` with `condition: service_healthy`** — Compose waits until MySQL's healthcheck (`mysqladmin ping`) passes before starting the backend container.
- **Application-level retry loop** — `connectWithRetry()` in `app.js` attempts to connect to MySQL up to 10 times with 3-second delays. This handles cases where MySQL is up but not yet fully accepting connections.

> ⚠️ `depends_on` alone is NOT sufficient (MySQL process starts before it's ready to accept connections). The app-level retry handles this gap.

### 2. How Nginx Gets the Backend URL

Nginx uses `nginx.conf.template` with the `${BACKEND_URL}` placeholder.  
The official `nginx:alpine` image automatically runs `envsubst` on all files in `/etc/nginx/templates/` at container startup, injecting the `BACKEND_URL` environment variable from `.env` (via Docker Compose).

**No hardcoded URLs anywhere.**

### 3. How Services Communicate

All services are on a custom bridge network `threeTier_network`. They resolve each other by service name:
- Frontend → Backend: `http://backend:3000`
- Backend → MySQL: `db:3306`

---

## 🧪 Testing Steps

### Access Frontend
Open browser → `http://localhost`  
You'll see a status dashboard showing live health of all 3 services.

### Hit API Directly via Nginx Proxy
```bash
# Root endpoint (via Nginx proxy)
curl http://localhost/api/

# Health endpoint — shows DB status
curl http://localhost/api/health
```

Expected responses:
```json
// GET /api/
{"status":"OK","message":"API root reached via Nginx proxy"}

// GET /api/health — when DB is connected
{"status":"healthy","database":"connected"}

// GET /api/health — when DB is down
{"status":"unhealthy","database":"disconnected","error":"..."}
```

### View All Logs
```bash
docker compose logs -f
```

---

## 💥 Failure Scenario: MySQL Restart

### What Happens
```bash
docker restart mysql_db
```

1. **MySQL goes down** — backend's next DB query (e.g. `/health`) fails immediately with a connection error.
2. **Backend stays running** — it does NOT crash. The `GET /health` returns `503 unhealthy` while MySQL is restarting.
3. **MySQL comes back up** (~15–30 seconds typically)
4. **Backend auto-reconnects** — the `getDb()` function detects `dbConnection = null` (set on connection error event) and triggers `connectWithRetry()` on the next request.
5. **Recovery complete** — `GET /health` returns `200 healthy` again.

### Recovery Time
Typically **15–30 seconds** from restart to full recovery:
- MySQL restart itself: ~10–20s
- Backend retry detects reconnection: on next incoming request

### How It's Handled
```js
// In app.js — connection error handler resets the connection
dbConnection.on('error', async (err) => {
  console.error('DB connection lost:', err.message);
  dbConnection = null; // triggers reconnect on next request
});
```

Combined with `connectWithRetry()`, the backend heals itself without any manual intervention.

---

## ⭐ Bonus Features Implemented

- ✅ **Multi-stage Docker builds** — both frontend and backend use multi-stage builds to minimize image size
- ✅ **Non-root USER** — both containers run as non-root users (`appuser`, `nginxuser`)

---

## 🔐 Security Notes

- `.env` is in `.gitignore` — never committed
- Only `.env.example` with placeholder values is committed
- No secrets hardcoded anywhere in source files
