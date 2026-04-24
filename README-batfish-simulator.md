# Batfish Network Simulation Platform

Single-file browser-based network simulation platform for multi-vendor infrastructure analysis using [Batfish](https://www.batfish.org/).

**File:** `batfish-simulator_ok.html` — open directly in any browser. No server, no build, no dependencies.

---

## Quick Start

1. Download `batfish-simulator_ok.html`
2. Open in Chrome / Edge / Firefox
3. Select a topology from the **TOPO** dropdown
4. Run simulations from the **SIMULATIONS** tab

Optional: connect to a live Batfish instance for real analysis.

---

## Features

### Multi-Vendor Support

| Vendor | Device Types | Config Syntax |
|---|---|---|
| **Cisco** IOS/ASA | Router, Switch, Firewall | `hostname`, `interface`, `router bgp/ospf`, `access-list` |
| **Fortinet** FortiGate | Firewall, VPN | `config system`, `config firewall policy`, `config vpn ipsec` |
| **F5** BIG-IP | Load Balancer | `ltm pool`, `ltm virtual`, `ltm profile client-ssl` |
| **Palo Alto** NGFW | Firewall | `set zone`, `set rulebase security rules`, `set network ike` |
| **AWS** | VPC, Transit Gateway | `[VPC]`, `[SecurityGroup]`, `[TransitGatewayRouteTable]` |

### Topology Management

- **4 pre-built topologies** ready to use
- **Create** new empty topologies
- **Switch** between topologies (devices, links, configs swap independently)
- **Rename** and **delete** topologies
- Each topology maintains its own device inventory, link map, and config store

### Pre-Built Topologies

| Topology | Devices | Links | Description |
|---|---|---|---|
| **Default** | 13 | 14 | Mixed vendor datacenter with F5 HA, FortiGate edge, Cisco core, Palo Alto segmentation, AWS cloud |
| **Enterprise Hybrid Cloud** | 15 | 17 | Multi-tier DC: Cisco BGP routers → FortiGate perimeter → F5 SSL offload → Palo Alto microsegmentation → AWS TGW |
| **Multi-Site Enterprise** | 14 | 14 | Hub-spoke with FortiGate branch VPN, Cisco hub routers, Palo Alto DC firewalls, F5 load balancing, AWS DR |
| **Financial Zero Trust** | 16 | 16 | Per-tier Palo Alto segmentation (web/app/data zones), FortiGate perimeter+partner VPN, F5 WAF+API gateway, AWS PCI |

### Interactive Topology View

- **Drag nodes** to reposition devices in the SVG canvas
- **Link Mode** (`🔗 LINK MODE` button) — click two devices to create a connection
- **Delete links** — hover a link line (turns red) and click to remove
- **Delete devices** — hover the `🗑` button in the sidebar

### Configuration Editor

- Full config editor per device with vendor syntax
- **Validate** configs (checks for placeholders, missing interfaces)
- **Format**, **Copy**, **Insert Template** helpers
- Configs persist per topology

### Network Simulations

| Simulation | Description |
|---|---|
| **Reachability** | BFS path analysis between any two devices through the link topology |
| **Traceroute** | Hop-by-hop path display |
| **ACL Trace** | Filter/policy lookup on a specific device |
| **Show Routes** | Routing table display for a device |
| **BGP Analysis** | BGP neighbor, AS-path, local preference analysis |
| **OSPF Analysis** | OSPF neighbor state, LSDB, SPF run count (uses actual topology links) |
| **Multipath (ECMP)** | Equal-cost multipath check across all routers/firewalls |

All simulations dynamically use the **current topology's** devices and links.

### HTML Report Generation

Every simulation auto-downloads a standalone HTML report to your Downloads folder.

**Report contents:**

- Simulation result with full analysis output
- Device inventory table (name, vendor, role, IP, site, status)
- Link topology table
- Routing tables per device (parsed from configs)
- BGP summary (AS numbers, neighbors, remote-AS)
- OSPF summary (process ID, router-id, area, networks)
- ACL/Policy summary (permit/deny rules per device)
- Interface summary per device

**Report filename:** `batfish-report-{topology}-{sim-type}-{timestamp}.html`

Use the **📄 EXPORT REPORT** button in the topbar for a cumulative report of all simulation results.

Reports use the same dark theme as the simulator.

### Diff Analysis

Side-by-side config comparison between baseline and candidate configurations with added/removed/unchanged line counts.

### Batfish Integration

Connect to a live Batfish instance for real network analysis:

```bash
docker run --name batfish -v bf-vol:/data \
  -p 9996:9996 -p 9997:9997 batfish/allinone
```

1. Set the Batfish host IP and port in the topbar
2. Click **CONNECT**
3. Click **⬆ PUSH SNAPSHOT** to upload all device configs
4. Run simulations — results come from Batfish instead of mock data

Without Batfish, all simulations run in **mock mode** with realistic generated output.

---

## UI Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ BATFISH·SIM  TOPO:[dropdown] ✎＋✕  [host][port] CONNECT  ...   │
├──────────┬──────────────────────────────────┬────────────────────┤
│          │                                  │                    │
│ DEVICE   │     MAIN CONTENT                 │  SIMULATION        │
│ LIST     │  ┌────────────────────────┐       │  RESULTS           │
│          │  │ TOPOLOGY / CONFIG /    │       │  & LOGS            │
│ (by      │  │ SIMULATION / DIFF     │       │                    │
│  vendor) │  │                        │       │                    │
│          │  │                        │       │                    │
│ LINKS    │  └────────────────────────┘       │                    │
│ LIST     │                                  │                    │
│          │                                  │                    │
│ [+ADD]   │                                  │                    │
└──────────┴──────────────────────────────────┴────────────────────┘
```

---

## Tech Stack

- **Zero dependencies** — vanilla HTML/CSS/JS in a single file
- SVG topology rendering with drag-and-drop
- Blob API for report file downloads
- Google Fonts (JetBrains Mono + Rajdhani)
- Optional: Batfish REST API integration

---

## License

Educational / lab use.
