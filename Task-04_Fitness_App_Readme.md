# Fitness Tracker — Manual Setup Guide (Without Docker)

This guide explains how to run the Fitness Tracker application manually across two Azure VMs — without using Docker.

---

## Architecture

![Architecture Diagram](architecture.png)

> The diagram shows the full request flow:
> - User sends a request via Chrome browser to VM1 (Central India)
> - VM1 processes the request through the NSG and Node.js app
> - Database operations are tunnelled from VM1 to VM2 (South India) via VNet Peering
> - VM2 fetches the data from MongoDB and returns it back to VM1
> - VM1 sends the response back to the browser

```
VM1 (App Server) — Central India        VM2 (Database Server) — South India
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  Node.js App (port 5000)    │ ──────► │  MongoDB (port 27017)        │
│  Serves Frontend (static)   │         │  Private IP: 10.1.1.4        │
│  MONGODB_URI = 10.1.1.4     │         │                              │
└─────────────────────────────┘         └──────────────────────────────┘
```

- VM1 runs the Node.js backend and serves the frontend static files
- VM2 runs MongoDB and is accessible only via private IP
- Both VMs are connected via Azure VNet Peering

---

## Prerequisites

| VM | What's Needed |
|---|---|
| VM1 | Node.js v18+, npm |
| VM2 | MongoDB 7.0 |

---

## VM2 Setup — MongoDB Server

### 1. Install MongoDB

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt update
sudo apt install -y mongodb-org
```

### 2. Allow Remote Connections

By default MongoDB only listens on `127.0.0.1`. Open it to accept connections from VM1.

```bash
sudo nano /etc/mongod.conf
```

Find the `net:` section and update `bindIp`:

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0    # Changed from 127.0.0.1
```

### 3. Start and Enable MongoDB

```bash
sudo systemctl start mongod
sudo systemctl enable mongod

# Verify it is running
sudo systemctl status mongod
```

You should see `Active: active (running)`.

---

## VM1 Setup — App Server

### 1. Install Node.js v18

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node -v    # Should show v18.x.x
npm -v     # Should show 9.x.x
```

### 2. Clone the Repository

```bash
git clone https://github.com/Sathvik307393/Fitness_Tracker.git
cd Fitness_Tracker
```

### 3. Configure the MongoDB URI

Open `server/app.js` and update the MongoDB connection URI to point to VM2's private IP instead of localhost:

```javascript
// Before
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/fitness-tracker';

// After — use VM2's private IP
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://10.1.1.4:27017/fitness-tracker';
```

> **Note:** `10.1.1.4` is the private IP of VM2 (MongoDB server). Use the private IP and not the public IP for VM-to-VM communication within Azure VNet.

### 4. Install Dependencies

```bash
# Install server dependencies (express, mongoose, dotenv, dd-trace, etc.)
cd server
npm install
cd ..
```

### 5. Create the `.env` File

```bash
cp .env.example .env
nano .env
```

Set the following values:

```env
NODE_ENV=production
PORT=5000
HOST=0.0.0.0
MONGODB_URI=mongodb://10.1.1.4:27017/fitness-tracker
```

### 6. Start the Application

```bash
node server/app.js
```

On successful startup you should see:

```
✅ Connected to MongoDB: 10.1.1.4
🚀 Server running on http://0.0.0.0:5000
📊 Environment: production
🗄️  Database: mongodb://10.1.1.4:27017/fitness-tracker
```

---

## Azure NSG Rules

### VM1 — Inbound Rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH access |
| 5000 | TCP | Any | App access from browser |

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

## Verify MongoDB Connectivity from VM1

Before starting the app, confirm VM1 can reach VM2 on port 27017:

```bash
nc -zv 10.1.1.4 27017
# Expected: Connection to 10.1.1.4 27017 port [tcp] succeeded!
```

If it fails, check:
- VM2's `mongod.conf` has `bindIp: 0.0.0.0`
- VM2's NSG inbound rule allows port `27017` from VM1's private IP
- Both VMs are connected via VNet Peering

---

## Keep the App Running After SSH Logout (Recommended)

Without Docker, the app stops when you close the SSH session. Use `pm2` to keep it alive:

```bash
# Install pm2 globally
sudo npm install -g pm2

# Start the app with pm2
pm2 start server/app.js --name fitness-tracker

# Save the process list and auto-start on reboot
pm2 startup
pm2 save
```

Useful pm2 commands:

```bash
pm2 status                        # Check if app is running
pm2 logs fitness-tracker          # View live logs
pm2 restart fitness-tracker       # Restart the app
pm2 stop fitness-tracker          # Stop the app
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| `Cannot find module 'dd-trace'` | Dependencies not installed | Run `cd server && npm install` |
| `ECONNREFUSED 10.1.1.4:27017` | MongoDB not reachable | Check NSG rules and `mongod.conf` bindIp |
| `MongoNetworkError` | Wrong private IP | Verify VM2 private IP in `server/app.js` |
| App stops after SSH logout | No process manager | Use `pm2` as shown above |
| Port 5000 not accessible | NSG rule missing | Add inbound rule for port 5000 in VM1 NSG |
| `apt` lock error during install | Background update running | Run `sudo kill <pid>` then `sudo dpkg --configure -a` |
