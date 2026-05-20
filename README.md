# Heimdall

Unified DDI management interface for BIND, Kea DHCP, and NetBox.

Heimdall is an open-source web application that provides a single control plane for managing DNS (BIND), DHCP (Kea), and IPAM (NetBox). It does not replace your existing infrastructure — it sits on top of it, talking to each engine via their native APIs.

## Philosophy

- **Bring your own engines** — Heimdall is a management layer, not a server. BIND, Kea, and NetBox run independently; Heimdall just talks to them.
- **Modular by design** — each integration is a self-contained Go plugin (`.so`). Drop a new module into `/modules`, restart the service, and it's available.
- **API-first** — the core exposes a REST API; the frontend is just one possible consumer.

## Tech Stack

| Layer | Technology |
| --- | --- |
| Core / Engine | Go 1.22+ |
| Module System | Go plugin system (`.so` shared libraries) |
| Frontend | React + Vite (Node.js) |
| API | REST (JSON) |
| Config | YAML + environment variables |

## Project Structure

```text
heimdall/
│
├── core/                         # Go backend — engine & REST API
│   ├── cmd/
│   │   └── heimdall/
│   │       └── main.go           # Entrypoint
│   ├── internal/
│   │   ├── api/                  # HTTP router & handlers (e.g. chi/fiber)
│   │   │   ├── router.go
│   │   │   └── handlers/
│   │   ├── loader/               # Plugin discovery & dynamic loading
│   │   │   └── loader.go         # Scans /modules, dlopen *.so, registers routes
│   │   ├── middleware/           # Auth, logging, CORS, error handling
│   │   ├── config/               # YAML + env config parsing
│   │   └── models/               # Shared structs & interfaces
│   │       └── module.go         # Module interface every plugin must implement
│   ├── go.mod
│   └── go.sum
│
├── frontend/                     # React web UI (Node.js)
│   ├── public/
│   ├── src/
│   │   ├── components/           # Reusable UI components
│   │   ├── pages/                # Route-level pages
│   │   ├── store/                # State management (Zustand / Redux)
│   │   └── api/                  # API client (axios / fetch)
│   ├── package.json
│   └── vite.config.ts
│
├── modules/                      # DDI integration modules (Go plugins)
│   ├── bind/                     # BIND DNS module
│   │   ├── manifest.json         # Module metadata & capabilities
│   │   ├── bind.go               # Implements core Module interface
│   │   ├── client.go             # BIND rndc/API client
│   │   ├── handlers.go           # HTTP handlers for this module
│   │   └── README.md
│   ├── kea/                      # Kea DHCP module
│   │   ├── manifest.json
│   │   ├── kea.go                # Implements core Module interface
│   │   ├── client.go             # Kea REST API client
│   │   ├── handlers.go
│   │   └── README.md
│   └── netbox/                   # NetBox IPAM module
│       ├── manifest.json
│       ├── netbox.go             # Implements core Module interface
│       ├── client.go             # NetBox REST/GraphQL client
│       ├── handlers.go
│       └── README.md
│
├── docs/                         # Documentation
│   ├── architecture.md
│   ├── module-development.md     # Guide to writing new modules
│   └── api-reference.md
│
├── Makefile                      # build, build-modules, run, docker
├── docker-compose.yml
├── .env.example
└── README.md
```

## Module System

Modules are compiled as Go shared libraries (`.so`) and loaded dynamically at startup via the Go plugin system. Each module must implement the `Module` interface defined in `core/internal/models/module.go`:

```go
// core/internal/models/module.go
type Module interface {
    Name()         string
    Version()      string
    Capabilities() []string
    RegisterRoutes(router Router)
    Init(cfg Config) error
    Shutdown() error
}
```

Each module directory also contains a `manifest.json` used by the loader before opening the `.so`:

```json
{
  "name": "kea",
  "version": "1.0.0",
  "description": "Kea DHCP integration",
  "capabilities": ["dhcp.leases", "dhcp.reservations", "dhcp.subnets"],
  "entry": "kea.so",
  "config_schema": {
    "host": { "type": "string", "required": true },
    "port": { "type": "integer", "default": 8000 },
    "api_key": { "type": "string", "required": false }
  }
}
```

On startup, the loader:

1. Scans `/modules` for directories containing a `manifest.json`
2. Reads the manifest to validate metadata
3. Opens the `.so` via `plugin.Open()`
4. Looks up the exported `NewModule` symbol
5. Calls `Init()` with the module config
6. Calls `RegisterRoutes()` to mount the module's endpoints on the main router

No changes to core are needed — drop a new `.so` + `manifest.json` into `/modules` and restart.

## Built-in Modules

| Module | Engine | Capabilities |
| --- | --- | --- |
| bind | BIND 9 | Zones, records, RNDC control |
| kea | Kea DHCP | Leases, reservations, subnets, statistics |
| netbox | NetBox | IP addresses, prefixes, VLANs, devices |

## Building

```bash
# Build the core
make build

# Build all modules as .so
make build-modules

# Build a single module
make build-module MODULE=kea

# Run locally
make run
```

## Getting Started

### Requirements

- Go 1.22+
- Node.js 20+
- Access to a running BIND, Kea, and/or NetBox instance

### Quick Start

```bash
git clone https://github.com/youruser/heimdall.git
cd heimdall

cp .env.example .env
# Edit .env with your connection details

# Start with Docker
docker compose up -d

# Or manually
make build && make build-modules
./bin/heimdall

cd frontend && npm install && npm run dev
```

### Configuration

```env
# Core
HEIMDALL_PORT=8080
HEIMDALL_MODULES_DIR=./modules

# Kea
KEA_HOST=192.168.1.1
KEA_PORT=8000

# BIND
BIND_RNDC_HOST=192.168.1.2
BIND_RNDC_PORT=953
BIND_RNDC_KEY=/etc/rndc.key

# NetBox
NETBOX_HOST=https://netbox.local
NETBOX_TOKEN=your_api_token
```

## Writing a Module

See `docs/module-development.md` for the full guide. The short version:

1. Create a folder under `/modules/your-module/`
2. Add a `manifest.json`
3. Implement the `Module` interface from `core/internal/models/module.go`
4. Export a `NewModule() models.Module` symbol
5. Compile with `go build -buildmode=plugin -o your-module.so .`
6. Restart Heimdall — the module loads automatically

## Caveats on Go Plugins

Go plugins have a few constraints worth knowing:

- The plugin and the core must be compiled with the same Go version and same dependencies
- Only supported on Linux (and macOS with limitations) — not on Windows
- Once loaded, a plugin cannot be unloaded — a restart is required to reload
- All shared packages (e.g. `core/internal/models`) must be identical between core and plugin

These are acceptable tradeoffs for a server-side DDI tool running on Linux.

## Roadmap

- [x] Core API & module loader
- [x] Kea DHCP module
- [x] BIND module
- [x] NetBox module
- [ ] Web UI (React)
- [ ] Authentication (local + LDAP)
- [ ] Audit log
- [ ] Webhook/event system for cross-module sync

## License

MIT
