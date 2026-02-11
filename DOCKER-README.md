# Three-Tier App

A full-stack three-tier application demonstrating Docker containerization, networking, volumes, and nginx reverse proxy configuration.

## Architecture

| Tier | Technology | Description |
|------|-----------|-------------|
| **Frontend** | React.js | Single-page application served via nginx |
| **Backend** | Node.js (Express) | REST API server |
| **Database** | PostgreSQL | Data persistence |

```
Browser → nginx (React + Reverse Proxy) → Express API → PostgreSQL
```

## Project Structure

```
├── backend/                 # Node.js API
│   ├── src/
│   │   ├── config/         # Database configuration
│   │   ├── migrations/     # Database migrations
│   │   ├── routes/         # API routes
│   │   ├── services/       # Business logic
│   │   └── server.js       # Express server
│   ├── entrypoint.sh       # Docker entrypoint (runs migrations + starts server)
│   ├── Dockerfile
│   └── package.json
├── frontend/               # React application
│   ├── src/
│   │   ├── components/     # React components
│   │   ├── services/       # API services
│   │   └── tests/          # Component tests
│   ├── nginx.conf          # Nginx config for Docker
│   ├── nginx-cp.conf       # Nginx reference/notes
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
└── backend-secrets.env     # Secrets (DO NOT commit to git!)
```

---

## Getting Started

### Prerequisites

- Node.js 20
- Docker
- PostgreSQL (for local development)
- npm

### Environment Variables

#### Backend

```
PORT=3001
NODE_ENV=development
DATABASE_URL=postgresql://devops_user:devops_password@localhost:5432/devops
```

#### Frontend

```
REACT_APP_API_URL=http://localhost:3001/api
```

---

## Local Development (Without Docker)

### 1. Start PostgreSQL Container

```bash
docker run -itd --rm --name devops-postgres \
  -e POSTGRES_DB=devops \
  -e POSTGRES_USER=devops_user \
  -e POSTGRES_PASSWORD=devops1234 \
  -p 5432:5432 postgres:15-alpine
```

PostgreSQL connection details:

```
POSTGRES_DB=devops
POSTGRES_USER=devops_user
POSTGRES_PASSWORD=devops1234
POSTGRES_HOSTNAME=localhost
```

### 2. Start Backend

```bash
cd backend
npm install
npm run migrate           # Run database migrations
npm run dev               # Start with auto-migration
```

### 3. Start Frontend

```bash
cd frontend
npm install
npm start
```

---

## Docker Compose (Easiest Way)

> **⚠️ Windows Users Only**: Fix line endings first:
>
> ```bash
> dos2unix backend/entrypoint.sh
> ```

> **⚠️ Remove standalone PostgreSQL container** if already running:
>
> ```bash
> docker stop devops-postgres
> ```

```bash
docker compose up
```

---

## Building Docker Images Manually

### Backend Image — With Entrypoint (runs migrations automatically)

Dockerfile uses:

```dockerfile
CMD ["./entrypoint.sh"]
```

```bash
docker build -t backend-app-wm:latest ./backend
```

### Backend Image — Without Entrypoint (server only)

Dockerfile uses:

```dockerfile
CMD ["node", "src/server.js"]
```

```bash
docker build -t backend-app:latest ./backend
```

### Frontend Image

```bash
cd frontend
docker build -t frontend-app .
```

---

## Running Containers Manually

### Backend Container (Basic)

```bash
docker run -it --name backend-app --rm \
  -p 5001:5001 \
  backend-app:latest
```

> **⚠️ This won't work!** The host port is mapped to 5001, but the server defaults to PORT=3001 inside the container since no environment variable was provided. The port mapping and the app port must match.

### Backend Container (With Correct Port)

```bash
docker run -it --name backend-app --rm \
  -p 5001:5001 \
  -e PORT=5001 \
  backend-app:latest
```

Now you can access it at `http://<vm-ip>:5001`.

---

## Docker Networking

### Why Do We Need Custom Networks?

By default, each container gets its own isolated network. Containers can't talk to each other using container names unless they're on the same custom network.

### Why `npm run dev` on Host Can Connect to PostgreSQL in Docker

When you run PostgreSQL in Docker with `-p 5432:5432` and run `npm run dev` directly on the host, it works because:

```
Your Host Machine
┌──────────────────────────────────────────────────┐
│                                                  │
│  npm run dev (Express app)                       │
│  connects to → localhost:5432  ✅                │
│                    │                             │
│                    │ (loopback / 127.0.0.1)      │
│                    ↓                             │
│  ┌──────────────────────────────────┐            │
│  │ Docker: PostgreSQL container     │            │
│  │ -p 5432:5432                     │← port      │
│  │ listening on container:5432      │  mapped to  │
│  └──────────────────────────────────┘  host:5432 │
└──────────────────────────────────────────────────┘
```

**Two things make it work:**

1. **`-p 5432:5432`** on the PostgreSQL container maps the host's port 5432 to the container's port 5432, making it accessible at `localhost:5432` on the host.

2. **The fallback DATABASE_URL in the code** (`backend/src/config/database.js`):

```javascript
connectionString: process.env.DATABASE_URL || 'postgresql://devops_user:devops_password@localhost:5432/devops'
```

Since no `DATABASE_URL` env var is set when running `npm run dev`, the code falls back to `localhost:5432` — which works because the app runs on the host and `-p 5432:5432` exposes PostgreSQL there.

### Why `localhost` Breaks When Backend Is Also in a Container

```
Your Host Machine
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ┌─────────────────────────┐                         │
│  │ Backend container       │                         │
│  │ connects to →           │                         │
│  │ localhost:5432  ❌      │  ← "localhost" = THIS   │
│  │ (nothing here!)         │     container, NOT host │
│  └─────────────────────────┘                         │
│                                                      │
│  ┌─────────────────────────┐                         │
│  │ PostgreSQL container    │                         │
│  │ listening on 5432       │  ← unreachable from     │
│  └─────────────────────────┘    backend container    │
└──────────────────────────────────────────────────────┘
```

Inside a container, `localhost` refers to **that container itself** — not the host, and not other containers. That's why you need:

- A **custom Docker network** so both containers can communicate
- Use the **container name** (e.g., `devops-postgres`) as the hostname instead of `localhost`

```
DATABASE_URL=postgresql://devops_user:devops1234@devops-postgres:5432/devops
                                                  ^^^^^^^^^^^^^^^^
                                                  Container name, resolved by Docker DNS
```

#### DATABASE_URL Breakdown

```
postgresql://devops_user:devops1234@devops-postgres:5432/devops
```

| Part | Value | Meaning |
|------|-------|---------|
| `postgresql://` | Protocol | Use the PostgreSQL driver |
| `devops_user` | **Username** | PostgreSQL user (set by `POSTGRES_USER`) |
| `:` | Separator | Separates username from password |
| `devops1234` | **Password** | User's password (set by `POSTGRES_PASSWORD`) |
| `@` | Separator | Separates credentials from the host |
| `devops-postgres` | **Hostname** | Container name — Docker DNS resolves it to the container's IP |
| `5432` | **Port** | PostgreSQL's default port inside the container |
| `/devops` | **Database** | Database name (set by `POSTGRES_DB`) |

#### Summary: When Does `localhost` Work?

| Scenario | `localhost` means | Works? |
|---|---|---|
| `npm run dev` on host + Postgres in Docker with `-p 5432:5432` | The host machine | ✅ Yes |
| Backend in container + Postgres in container (no custom network) | The backend container itself | ❌ No |
| Backend in container + Postgres in container (same custom network, using container name) | N/A — uses `devops-postgres` | ✅ Yes |

---

### Step-by-Step: Connecting Backend to Database

#### 1. Create a custom network

```bash
docker network create net1
docker network ls
```

#### 2. Run PostgreSQL on the network

```bash
docker run -itd --rm --name devops-postgres \
  --network net1 \
  -e POSTGRES_DB=devops \
  -e POSTGRES_USER=devops_user \
  -e POSTGRES_PASSWORD=devops1234 \
  -p 5432:5432 postgres:15-alpine
```

#### 3. Run Backend on the same network

```bash
docker run -it --name backend-app --rm \
  --network net1 \
  -p 5001:5001 \
  -e PORT=5001 \
  -e DATABASE_URL=postgresql://devops_user:devops1234@devops-postgres:5432/devops \
  backend-app-wm:latest
```

> Notice `@devops-postgres` in the DATABASE_URL — Docker DNS resolves the container name to its IP on the custom network.

#### 4. Test it

```bash
# GET request
curl http://<vm-ip>:5001/api/users

# POST request (e.g., via ThunderClient)
# Body:
# {
#   "name": "Bibek Labh",
#   "email": "bkarna@gmail.com"
# }
```

> **⚠️ Without volumes**: If you stop the database and restart, **all data is lost!**

---

### Docker Network Demonstration (Failure vs Success)

This demonstrates why custom networks are necessary:

#### Run WITHOUT network — Connection Failure

```bash
docker run --name backend-test \
  -p 5001:5001 \
  -e DATABASE_URL="postgresql://devops_user:devops_password@localhost:5432/devops" \
  backend-app:latest
```

**Expected:** ❌ Connection refused — `localhost` inside the container is the container itself, not the host.

```bash
docker stop backend-test && docker rm backend-test
```

#### Run WITH network — Successful Connection

```bash
docker network create three-tier-network

docker run -d \
  --name postgres-db \
  --network three-tier-network \
  -e POSTGRES_USER=devops_user \
  -e POSTGRES_PASSWORD=devops_password \
  -e POSTGRES_DB=devops \
  -p 5432:5432 \
  postgres:15-alpine

docker run --name backend-app \
  --network three-tier-network \
  -p 5001:5001 \
  -e DATABASE_URL="postgresql://devops_user:devops_password@postgres-db:5432/devops" \
  backend-app:latest
```

**Expected:** ✅ Backend connects to the database using the container name `postgres-db` as the hostname.

---

### Docker Network Types

#### 1. Bridge Network (Default) 🌉

Creates a private internal network on your host. Containers can talk to each other, and Docker does NAT to reach the internet.

**When to use:** Default for most applications, microservices on a single host.

```
Host Machine (192.168.1.100)
    │
    ├─ docker0 bridge (172.17.0.1)
    │   │
    │   ├─ container1 (172.17.0.2)
    │   ├─ container2 (172.17.0.3)
    │   └─ container3 (172.17.0.4)
    │
    └─ Internet ←→ NAT ←→ containers
```

#### 2. Host Network 🏠

Container shares the host's network stack directly. No network isolation!

**When to use:** Maximum performance, monitoring tools, network utilities.

```
Host Machine (192.168.1.100)
    │
    └─ Container (uses host's 192.168.1.100 directly)
       No NAT, no bridge, no isolation!
```

#### 3. Overlay Network ☁️

Multi-host networking for Docker Swarm. Containers on different machines can communicate!

**When to use:** Docker Swarm, distributed applications, microservices across hosts.

```
Host 1 (192.168.1.10)          Host 2 (192.168.1.20)
    │                              │
    ├─ container1 ←─────┬─────→ ├─ container3
    │  (10.0.0.2)       │         │  (10.0.0.4)
    │                   │         │
    ├─ container2       │         ├─ container4
       (10.0.0.3)       │            (10.0.0.5)
                        │
                VXLAN Tunnel (overlay)
         (Encrypted cross-host communication)
```

#### 4. None Network 🚫

No network at all! Complete isolation.

**When to use:** Maximum security, batch processing, testing.

```
Container
    │
    └─ No network interface
       Can't access anything!
```

---

### Bridge vs Host Network — Nginx Example

#### Bridge Network (with port mapping)

```
Your Host Machine (192.168.1.100)
    │
    ├─ eth0 (192.168.1.100) ← Host's real IP
    │
    └─ docker0 bridge (172.17.0.1) ← Virtual bridge
        │
        ├─ nginx container (172.17.0.2) ← Private IP
        │   Port 80 inside container
        │   │
        │   └─ NAT/Port Forward ─→ Host's port 8080
        │
        └─ Internet ←─ NAT ─→ Container

Access: curl http://192.168.1.100:8080  (goes through NAT to container's port 80)
```

```bash
# Run nginx on bridge with port mapping
docker run -d \
  --name nginx-bridge \
  -p 9090:80 \
  nginx

# Check container's IP
docker inspect nginx-bridge
# Output: 172.17.0.2  ← Private IP

# Check from host
curl http://localhost:9090  # ✅ Works (port 9090 mapped to container's 80)
curl http://localhost:80    # ❌ Fails (nothing on host's port 80)

# Check listening ports on host
netstat -tuln | grep 9090
# tcp  0.0.0.0:9090  ← Docker proxy listening

# What's happening:
# Request → Host:9090 → Docker proxy → NAT → Container:80
```

#### Host Network (no port mapping)

```
Your Host Machine (192.168.1.100)
    │
    └─ eth0 (192.168.1.100) ← Container uses THIS directly!
        │
        └─ nginx container (uses host's 192.168.1.100)
            Port 80 ← Binds directly to host's port 80
            │
            └─ No NAT, No bridge, No translation!

Access: curl http://192.168.1.100:80  (direct access, no NAT!)
```

> **⚠️ Port 80 conflict!** If something else uses port 80 on the host, nginx will fail with `bind() to 0.0.0.0:80 failed (98: Address already in use)`. Use a custom config to listen on a different port:

Create `custom-nginx.conf`:

```nginx
server {
    listen 8081;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

```bash
# Run nginx on host network (NO -p flag!)
docker run -it \
  --name nginx-host \
  --network host \
  -v $(pwd)/custom-nginx.conf:/etc/nginx/conf.d/default.conf \
  nginx

# Check container's IP
docker inspect nginx-host
# Output: (empty) ← No separate IP!

# Container uses host's network stack directly
docker exec -it nginx-host /bin/sh
apt-get update
apt-get install -y iproute2
ip addr
# Shows: 192.168.1.100 (same as host!)

# Check from host
curl http://localhost:8081  # ✅ Works directly!
curl http://localhost:8080  # ❌ Nothing (we didn't use -p, and it's ignored anyway)

# What's happening:
# Request → Host:8081 → nginx (no translation!)
```

---

## Docker Volumes

### Problem: Data Loss Without Volumes

Without volumes, all data inside a container is lost when the container stops.

### Named Volumes (Persist Data)

```bash
docker run -itd --rm --name devops-postgres \
  --network net1 \
  -e POSTGRES_DB=devops \
  -e POSTGRES_USER=devops_user \
  -e POSTGRES_PASSWORD=devops1234 \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15-alpine
```

```bash
docker run -it --name backend-app --rm \
  --network net1 \
  -p 5001:5001 \
  -e PORT=5001 \
  -e DATABASE_URL="postgresql://devops_user:devops1234@devops-postgres:5432/devops" \
  backend-app-wm:latest
```

> Now data persists even after stopping and restarting the containers!

### Bind Mounts (Mount Host Directories)

Useful for live development — mount your local build output into the container:

```bash
# Build frontend locally first
cd frontend
npm run build
cd ..

# Run frontend with bind mount
docker run -it --rm --name frontend-app \
  --network net1 \
  -p 8080:80 \
  -v "$PWD/frontend/build":/usr/share/nginx/html \
  frontend-app:latest
```

The full request flow:

```
Browser → http://localhost:8080/api/users
    ↓
nginx (frontend container)
    ↓
proxy_pass http://backend-app:5001
    ↓
Backend container (internal Docker network)
    ↓
Database container
```

### tmpfs Volumes (In-Memory, Ephemeral)

Useful for secrets — data exists only in RAM and is never written to disk:

```bash
# Create secrets file (DO NOT commit to git!)
cat > backend-secrets.env << 'EOF'
DATABASE_URL=postgresql://devops_user:devops_password@postgres-db:5432/devops
NODE_ENV=production
JWT_SECRET=your-super-secret-jwt-key
EOF

# Secure the file
chmod 600 backend-secrets.env
```

```bash
# Run with tmpfs for secrets
docker run -d \
  --name backend-app-tmpfs \
  --network net1 \
  -p 5001:5001 \
  --tmpfs /run/secrets:size=1024m \
  -v $(pwd)/backend-secrets.env:/run/secrets/env:ro \
  backend-app:latest
```

Testing the tmpfs size limit:

```bash
# Write multiple files to fill the tmpfs
docker exec backend-app sh -c 'dd if=/dev/zero of=/run/secrets/test1 bs=1M count=1024'  # 1GB
docker exec backend-app sh -c 'dd if=/dev/zero of=/run/secrets/test2 bs=1M count=1024'  # 1GB
docker exec backend-app sh -c 'dd if=/dev/zero of=/run/secrets/test3 bs=1M count=1024'  # 1GB
docker exec backend-app sh -c 'dd if=/dev/zero of=/run/secrets/test4 bs=1M count=1024'  # 1GB
# Will fail when exceeding the 1024m limit
```

---

## Exit Status Codes

Every command returns an exit status code. `0` = success, anything else = failure.

```bash
# Example 1: Success
ls /home
echo $?     # Prints: 0 (success)

# Example 2: Failure
ls /fake/dir
echo $?     # Prints: 2 (error - no such file)

# Example 3: Node.js crash
node broken.js
echo $?     # Prints: 1 (error - node exited with error)
```
