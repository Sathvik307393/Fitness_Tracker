# Fitness Tracker — Docker Compose Setup Guide (Two VMs)

This guide explains how to run the Fitness Tracker application using Docker Compose across two Azure VMs — the web app on VM1 and MongoDB on VM2.

---

## Architecture

![Architecture Diagram](architecture.png)

> The diagram shows the full request flow:
> - User sends a request via Chrome browser to VM1 (Central India)
> - Request passes through NSG → Docker container (fitness-app)
> - Database operations are tunnelled from VM1 to VM2 (South India) via VNet Peering
> - VM2's MongoDB container fetches the data and returns it back to VM1
> - VM1 sends the response back to the browser

```
VM1 (App Server) — Central India        VM2 (Database Server) — South India
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  docker-compose up -d       │         │  docker run mongo:7.0-jammy  │
│                             │         │                              │
│  fitness-app (port 5000)    │ ──────► │  MongoDB (port 27017)        │
│  datadog-agent              │         │  Private IP: 10.1.1.4        │
│                             │         │                              │
│  MONGODB_URI = 10.1.1.4     │         │                              │
└─────────────────────────────┘         └──────────────────────────────┘
```

- VM1 runs the Node.js app and Datadog agent via Docker Compose
- VM2 runs MongoDB as a standalone Docker container
- Both VMs communicate via Azure VNet private IP (`10.1.1.4`)

---

## Prerequisites

| VM | What's Needed |
|---|---|
| VM1 | Docker, Docker Compose |
| VM2 | Docker |

---

## VM2 Setup — MongoDB Container

### 1. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl start docker
sudo systemctl enable docker
```

### 2. Run MongoDB Container

```bash
docker run -d \
  --name mongodb \
  --restart unless-stopped \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  mongo:7.0-jammy
```

### 3. Verify MongoDB is Running

```bash
docker ps
```

You should see:

```
CONTAINER ID   IMAGE             STATUS         PORTS
xxxxxxxxxxxx   mongo:7.0-jammy   Up X minutes   0.0.0.0:27017->27017/tcp
```

### 4. Check MongoDB Logs

```bash
docker logs mongodb
```

Look for this line confirming MongoDB is ready:

```
{"msg":"Waiting for connections","attr":{"port":27017}}
```

---

## VM1 Setup — App Server

### 1. Install Docker and Docker Compose

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl start docker
sudo systemctl enable docker

# Verify
docker --version
docker compose version
```

### 2. Clone the Repository

```bash
git clone https://github.com/Sathvik307393/Fitness_Tracker.git
cd Fitness_Tracker
```

### 3. Update `docker-compose.yml`

The original `docker-compose.yml` had all three services (app + mongodb + datadog) together. Since MongoDB is now on VM2, update the file as follows:

**Remove the `mongodb` service** and **update the `MONGODB_URI`** to point to VM2's private IP. Also remove `mongodb` from `depends_on`.

Your updated `docker-compose.yml` on VM1 should look like this:

```yaml
services:
  fitness-app:
    build: .
    ports:
      - "5000:5000"
    labels:
      com.datadoghq.ad.logs: '[{"source":"nodejs","service":"fitness-tracker"}]'
    environment:
      - NODE_ENV=production
      - MONGODB_URI=mongodb://10.1.1.4:27017/fitness-tracker
      - DD_AGENT_HOST=datadog-agent
      - DD_TRACE_AGENT_PORT=8126
      - DD_SERVICE=fitness-tracker
      - DD_ENV=production
      - DD_VERSION=1.0.0
      - DD_LOGS_INJECTION=true
    depends_on:
      - datadog-agent

  datadog-agent:
    image: gcr.io/datadoghq/agent:7
    environment:
      - DD_API_KEY=17f70c28759e381392e0f9efd594582b
      - DD_SITE=datadoghq.com
      - DD_APM_ENABLED=true
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
```

> **Key change:** `MONGODB_URI=mongodb://10.1.1.4:27017/fitness-tracker`
> `10.1.1.4` is the private IP of VM2. Private IP is used because both VMs are on the same Azure VNet — traffic stays internal and never hits the internet.

---

## Order of Operations — Always Start VM2 First

### Step 1 — On VM2, confirm MongoDB is running

```bash
docker ps | grep mongo
```

### Step 2 — On VM1, verify connectivity to VM2

```bash
nc -zv 10.1.1.4 27017
# Expected: Connection to 10.1.1.4 27017 port [tcp] succeeded!
```

### Step 3 — On VM1, start the app

```bash
docker-compose up -d
```

### Step 4 — Check containers are running

```bash
docker-compose ps
```

You should see:

```
NAME             IMAGE          STATUS         PORTS
fitness-app      fitness-app    Up X minutes   0.0.0.0:5000->5000/tcp
datadog-agent    agent:7        Up X minutes
```

### Step 5 — Check app logs for MongoDB connection

```bash
docker-compose logs -f fitness-app
```

Look for:

```
✅ Connected to MongoDB: 10.1.1.4
🚀 Server running on http://0.0.0.0:5000
```

---

## Azure NSG Rules

### VM1 — Inbound Rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH access |
| 5000 | TCP | Any | App access from browser |

### VM1 — Outbound Rules

| Port | Protocol | Destination | Purpose |
|---|---|---|---|
| 27017 | TCP | VM2 Private IP | Talk to MongoDB on VM2 |

### VM2 — Inbound Rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH access |
| 27017 | TCP | VM1 Private IP | MongoDB access from VM1 only |

> Never open port `27017` to `0.0.0.0/0` — this exposes your database to the entire internet.

---

## Accessing the Application

Open your browser and navigate to:

```
http://<VM1_PUBLIC_IP>:5000
```

### Available Pages

| URL | Page |
|---|---|
| `http://<VM1_PUBLIC_IP>:5000` | Home |
| `http://<VM1_PUBLIC_IP>:5000/pages/login.html` | Login |
| `http://<VM1_PUBLIC_IP>:5000/pages/log-workout.html` | Log Workout |
| `http://<VM1_PUBLIC_IP>:5000/pages/progress.html` | Progress |
| `http://<VM1_PUBLIC_IP>:5000/pages/track-metrics.html` | Track Metrics |
| `http://<VM1_PUBLIC_IP>:5000/pages/trainer-panel.html` | Trainer Panel |

---

## Useful Docker Commands

### On VM1

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View live logs
docker-compose logs -f

# View logs for a specific service
docker-compose logs -f fitness-app

# Restart a specific service
docker-compose restart fitness-app

# Rebuild and restart (after code changes)
docker-compose up -d --build
```

### On VM2

```bash
# Check MongoDB container
docker ps

# View MongoDB logs
docker logs -f mongodb

# Enter MongoDB shell
docker exec -it mongodb mongosh

# Inside mongosh — verify database exists
show dbs
use fitness-tracker
show collections
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| `ECONNREFUSED 10.1.1.4:27017` | MongoDB not reachable from VM1 | Check VM2 NSG inbound rule for port 27017 |
| `nc -zv` hangs from VM1 | NSG blocking port 27017 | Add inbound rule on VM2 NSG allowing VM1 private IP |
| `nc -zv` connection refused | MongoDB not listening on `0.0.0.0` | Verify `docker ps` shows `0.0.0.0:27017->27017` on VM2 |
| App container exits immediately | Build error or missing env | Run `docker-compose logs fitness-app` to see error |
| `mongodb` service name not found | Old `depends_on` still references it | Remove `mongodb` from `depends_on` in `docker-compose.yml` |
| Port 5000 not accessible in browser | NSG rule missing on VM1 | Add inbound rule for port 5000 in VM1 NSG |
| Changes not reflected after edit | Old image cached | Run `docker-compose up -d --build` to rebuild |
