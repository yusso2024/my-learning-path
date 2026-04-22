# Palo Alto Batfish Simulator

A full-stack network validation platform that simulates Palo Alto Networks firewall environments using **Batfish** for configuration analysis. Built with Node.js, React, MongoDB, and Batfish — all running in Docker.

**Live URL:** `http://<VM_IP>:8090`

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Docker Compose                        │
│                                                           │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐         │
│  │ Frontend  │───▶│  Backend  │───▶│  Batfish   │         │
│  │ React 19  │    │ Node.js   │    │ allinone   │         │
│  │ + Nginx   │    │ Express   │    │ (Java)     │         │
│  │ :80→8090  │    │ :3001     │    │ :9996      │         │
│  └───────────┘    └─────┬─────┘    └───────────┘         │
│                         │                                 │
│                    ┌────▼────┐                            │
│                    │ MongoDB │                            │
│                    │ :27017  │                            │
│                    └─────────┘                            │
└──────────────────────────────────────────────────────────┘
```

| Service    | Image                        | Purpose                                              |
|------------|------------------------------|------------------------------------------------------|
| `frontend` | React 19 + Vite + Nginx      | Dark terminal-themed UI for topology, config, analysis |
| `backend`  | Node.js 22 + Express + pybatfish | REST API, config management, analysis orchestration |
| `database` | MongoDB 7                    | Stores devices, versioned configs, analysis results   |
| `batfish`  | batfish/allinone             | Network config parser and analysis engine (2 GB heap) |

---

## Features

### Config Management
- **View and edit** Palo Alto set-format configurations in a syntax-highlighted editor
- **Version control** — every save creates a new version; old versions are never overwritten
- **Syntax validation** — checks `set`/`delete` line format, warns about missing zones or interfaces
- **File upload** — drag-and-drop `.set` / `.cfg` files for bulk import; device name extracted from filename
- **Add new devices** — inline form in the sidebar (name, type, role, model, IP, status)

### Batfish Analysis
When you click **"Re-run Analysis"**, the backend:

1. Collects the latest config version for every device from MongoDB
2. Creates a temporary Batfish snapshot directory with config files
3. Spawns a Python process (`batfish_runner.py`) that uses **pybatfish** to run:
   - `initIssues` — parse validation
   - `interfaceProperties` — interface and zone info
   - `bgpSessionStatus` + `bgpRib` — BGP analysis
   - `filterLineReachability` — find shadowed ACL rules
   - `routes` — full routing table
   - 4 `reachability` tests (trust→internet, untrust→trust blocked, untrust→dmz, mgmt→trust)
   - `traceroute` — sample path trace
4. Parses results into structured checks (pass / fail / warn) and stores them in MongoDB
5. Frontend polls every 3 seconds until complete, then renders results

### HTML Report Generation
- Click **"Generate Report"** after any completed analysis
- Produces a self-contained HTML file with the same dark theme
- Includes: validation checks, BGP routes, interfaces, ACL reachability
- Opens in a new browser tab
- Reports persist on disk at `/api/reports/files/<filename>.html`

---

## Project Structure

```
paloalto-batfish-simulator/
├── docker-compose.yml              # All 4 services
├── .env                            # Credentials and ports
├── backend/
│   ├── Dockerfile                  # Node 22-slim + Python 3 + pybatfish
│   ├── package.json
│   └── src/
│       ├── server.js               # Express entry point
│       ├── config/db.js            # MongoDB connection with retry
│       ├── models/
│       │   ├── Device.js           # 8 network devices
│       │   ├── Config.js           # Versioned PA configs
│       │   └── Analysis.js         # Batfish analysis results
│       ├── routes/
│       │   ├── devices.js          # CRUD: GET / POST / PUT / DELETE
│       │   ├── configs.js          # Config editing + validation
│       │   ├── analysis.js         # Run / poll / history
│       │   └── reports.js          # HTML report generation
│       ├── services/
│       │   └── batfish.js          # Orchestrates Python runner
│       └── scripts/
│           └── batfish_runner.py   # pybatfish analysis (called via child_process)
├── frontend/
│   ├── Dockerfile                  # Multi-stage: Vite build → Nginx serve
│   ├── nginx/default.conf          # Proxies /api → backend:3001
│   └── src/
│       ├── App.jsx                 # Main orchestrator
│       ├── api.js                  # Fetch wrapper for all endpoints
│       ├── styles/theme.css        # Dark terminal theme
│       └── components/
│           ├── Header.jsx          # Logo + badges
│           ├── Sidebar.jsx         # Devices, zones, checks, add-device form
│           ├── TopologyPanel.jsx   # SVG network diagram
│           ├── ConfigPanel.jsx     # Config editor + file upload
│           ├── AnalysisPanel.jsx   # Pass / fail / warn checks
│           └── RightPanel.jsx      # Device info, routes, issues
└── database/
    └── init-mongo.js               # Seeds 8 devices on first boot
```

---

## API Endpoints

| Method | Endpoint                     | Purpose                           |
|--------|------------------------------|-----------------------------------|
| GET    | `/api/health`                | Health check                      |
| GET    | `/api/devices`               | List all devices                  |
| POST   | `/api/devices`               | Add a new device                  |
| PUT    | `/api/devices/:name`         | Update a device                   |
| DELETE | `/api/devices/:name`         | Remove a device                   |
| GET    | `/api/configs`               | List all configs                  |
| GET    | `/api/configs/:deviceName`   | Get latest config for a device    |
| PUT    | `/api/configs/:deviceName`   | Save new config version           |
| POST   | `/api/configs/validate`      | Validate config syntax            |
| POST   | `/api/analysis/run`          | Start Batfish analysis            |
| GET    | `/api/analysis/status/:id`   | Poll analysis status              |
| GET    | `/api/analysis/results/:id`  | Get full results with checks      |
| GET    | `/api/analysis/latest`       | Get most recent completed analysis|
| GET    | `/api/analysis/history`      | List past analyses                |
| POST   | `/api/reports/generate`      | Generate HTML report              |
| GET    | `/api/reports`               | List generated reports            |

---

## Sample Topology

The simulator ships with a pre-configured topology of 8 devices:

| Device          | Model       | Role                | Zone         | IP              |
|-----------------|-------------|---------------------|--------------|-----------------|
| `pa-fw-01`      | PA-5250     | Edge Firewall (HA Active) | untrust / trust / dmz / mgmt | 203.0.113.2 |
| `pa-fw-02`      | PA-5250     | HA Passive Peer     | untrust / trust / dmz / mgmt | 203.0.113.3 |
| `pa-fw-dc`      | PA-3410     | DC Firewall         | trust / dmz  | 172.16.0.1      |
| `core-sw-01`    | Cisco 9500  | Core Switch (VSS)   | —            | 10.0.0.254      |
| `core-sw-02`    | Cisco 9500  | Core Switch (VSS)   | —            | 10.0.1.254      |
| `isp-router-01` | Simulated   | ISP BGP Router      | —            | 203.0.113.1     |
| `dmz-sw-01`     | Cisco 3850  | DMZ Switch          | —            | 172.16.100.254  |
| `panorama-01`   | Panorama    | Management          | —            | 192.168.200.10  |

### Security Zones

| Zone    | Network          | Purpose     |
|---------|------------------|-------------|
| untrust | 0.0.0.0/0        | Internet    |
| trust   | 10.0.0.0/8       | Internal    |
| dmz     | 172.16.100.0/24  | DMZ servers |
| mgmt    | 192.168.200.0/24 | Management  |

### Pre-configured Security Policies

1. **allow-outbound-web** — trust → untrust (HTTP/S)
2. **inbound-https-dmz** — untrust → dmz (HTTPS to 172.16.100.10)
3. **dmz-to-trust-db** — dmz → trust (MySQL to 10.10.10.20)
4. **mgmt-access** — mgmt → any (SSH, ping)
5. **deny-untrust-to-trust** — explicit block
6. **default-deny** — implicit deny-all

---

## Palo Alto Config Format

Batfish parses Palo Alto PAN-OS **set-format** configurations. Example:

```
set deviceconfig system hostname pa-fw-01
set zone untrust network layer3
set zone untrust network layer3 ethernet1/1
set network interface ethernet ethernet1/1 layer3 ip 203.0.113.2/30
set rulebase security rules "allow-outbound-web" from trust to untrust source 10.0.0.0/8 destination any application [ web-browsing ssl ] service application-default action allow log-end yes
set network virtual-router default protocol bgp enable yes
set network virtual-router default protocol bgp local-as 65002
```

Cisco devices use standard IOS format (`.cfg` files).

---

## How Batfish Integration Works

The Node.js backend cannot call pybatfish directly (Python library), so it uses a hybrid approach:

1. **Node.js** collects configs from MongoDB and writes them to a temp snapshot directory
2. **Node.js** spawns a Python child process running `batfish_runner.py`
3. **Python/pybatfish** connects to the Batfish container, uploads the snapshot, runs all queries
4. **Python** outputs results as JSON to stdout
5. **Node.js** parses the JSON, builds structured checks, and stores everything in MongoDB

The Batfish container runs with a 2 GB JVM heap (`-Xmx2g`) to fit within typical VM memory constraints.

---

## Deployment

### Prerequisites
- Docker and Docker Compose v2+
- At least 4 GB available RAM (Batfish needs ~2 GB)

### Start

```bash
cd ~/paloalto-batfish-simulator
docker compose up -d
```

First boot pulls the Batfish image (~500 MB) and builds the backend/frontend images. Batfish takes ~90 seconds to initialize.

### Stop

```bash
docker compose down
```

### Rebuild After Code Changes

```bash
docker compose build frontend backend
docker compose up -d frontend backend
```

### View Logs

```bash
docker compose logs -f backend
docker compose logs -f batfish
```

### Seed Sample Configs

```bash
for f in backend/snapshots/sample/configs/*; do
  NAME=$(basename "$f" | sed 's/\.[^.]*$//')
  FMT="set"
  echo "$f" | grep -q '\.cfg' && FMT="ios"
  CONTENT=$(cat "$f" | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")
  curl -s -X PUT "http://localhost:8090/api/configs/$NAME" \
    -H "Content-Type: application/json" \
    -d "{\"content\":$CONTENT,\"format\":\"$FMT\"}"
done
```

---

## Environment Variables

| Variable              | Default          | Purpose                        |
|-----------------------|------------------|--------------------------------|
| `HOST_PORT`           | `8090`           | Frontend port on host          |
| `BACKEND_PORT`        | `3001`           | Backend internal port          |
| `MONGO_HOST`          | `database`       | MongoDB hostname (container)   |
| `MONGO_APP_DB`        | `batfish_simulator` | Database name               |
| `MONGO_APP_USER`      | `batfishuser`    | App database user              |
| `MONGO_APP_PASSWORD`  | `batfishpass123`  | App database password         |
| `MONGO_ROOT_USERNAME` | `root`           | MongoDB root user              |
| `MONGO_ROOT_PASSWORD` | `rootsecret`     | MongoDB root password          |
| `BATFISH_HOST`        | `batfish`        | Batfish hostname (container)   |
| `BATFISH_PORT`        | `9996`           | Batfish coordinator port       |

---

## Step-by-Step Build Process

This section documents exactly what was done to build the project from scratch.

### Step 1 — Environment Discovery

Connected to the VM via SSH. Found Ubuntu 24.04 with Docker Compose v5.1.1. Discovered an existing `docker-learning-stack` project (React + FastAPI + MongoDB) that served as the reference pattern for Docker service structure, Nginx proxy config, and MongoDB initialization.

### Step 2 — Research

Launched parallel research agents to find:
- **Batfish REST API** — learned that `batfish/allinone` exposes a coordinator on port 9996 and pybatfish is the official Python client
- **Palo Alto config format** — Batfish parses PAN-OS set-format CLI exports in a `configs/` subdirectory
- **Docker Compose patterns** — found production examples with health checks, volume mounts, and service dependencies

### Step 3 — Docker Infrastructure

Created `docker-compose.yml` with 4 services following the existing project pattern:
- Frontend (Nginx serving Vite build, proxying `/api` to backend)
- Backend (Node.js 22-slim with Python 3 and pybatfish installed in a venv)
- MongoDB 7 with init script seeding 8 network devices
- Batfish allinone with 2 GB heap limit and Python-based health check

### Step 4 — Backend Development

Built the Express API server with:
- **3 Mongoose models** — Device, Config (versioned), Analysis (with raw Batfish data and structured checks)
- **4 route modules** — devices CRUD, configs with validation, analysis with async execution and polling, reports with HTML generation
- **Batfish service** — writes configs to a temp snapshot directory, spawns Python runner, parses JSON results, builds pass/fail/warn checks
- **Python runner** (`batfish_runner.py`) — connects to Batfish via pybatfish, uploads snapshot, runs 8 analysis queries, outputs JSON

### Step 5 — Sample Configs

Created 5 Batfish-compatible network configs:
- 2 Palo Alto firewalls (`.set` format) with zones, interfaces, security policies, NAT rules, BGP, and HA
- 1 simulated ISP router (Cisco IOS with BGP)
- 2 core switches (Cisco IOS with static routes)

### Step 6 — Frontend Development

Converted the provided HTML theme into a React 19 application:
- Extracted all CSS variables and rules into `theme.css`
- Built 6 components matching the original dark terminal aesthetic
- Added API integration with polling for analysis status
- Added interactive config editor with validation feedback
- Later added: device creation form, drag-and-drop file upload, button styling fixes

### Step 7 — Deployment

Transferred the project to the VM via rsync. Built Docker images. Fixed two issues during deployment:
- **Batfish health check** — the container lacks `curl`, switched to Python socket-based check
- **pybatfish property spec** — removed invalid `interfaceProperties` parameter format

### Step 8 — Verification

Confirmed all 4 containers running and healthy. Seeded sample configs into MongoDB. Ran a full Batfish analysis (completed in 4 seconds) producing 2 passes, 3 fails, 1 warning. Generated an HTML report and verified it was accessible.

---

## Technologies Used

| Layer     | Technology                                      |
|-----------|------------------------------------------------|
| Frontend  | React 19, Vite 6, Nginx 1.27                   |
| Backend   | Node.js 22, Express 5, Mongoose 8              |
| Analysis  | Batfish (Java), pybatfish (Python 3), pandas    |
| Database  | MongoDB 7                                       |
| Container | Docker, Docker Compose v5                       |
| Theme     | Custom CSS (dark terminal / monospace aesthetic) |
