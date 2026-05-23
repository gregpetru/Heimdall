# Heimdall

Unified DDI management interface for **DNS (BIND)**, **DHCP (Kea)**, and **IPAM (NetBox)**.

Heimdall is an open-source web application that provides a **single control plane** for managing DNS (BIND), DHCP (Kea), and IPAM (NetBox). It does **not** replace your existing infrastructure — it sits on top of it, talking to each engine via their native APIs.

> Status (as of **May 23, 2026**): the repository currently focuses on the **core architecture + module system**. Some operational files (e.g. `docker-compose.yml`, `.env.example`, `CONTRIBUTING.md`) may not exist yet; this README is therefore intentionally written as a **complete project reference** and “single source of truth” until those files are added.

---

## Table of contents

- [What Heimdall is (and is not)](#what-heimdall-is-and-is-not)
- [Philosophy](#philosophy)
- [Who this is for](#who-this-is-for)
- [Core concepts](#core-concepts)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Repository layout](#repository-layout)
- [Getting started (development)](#getting-started-development)
- [Configuration](#configuration)
- [Module system](#module-system)
  - [Module interface](#module-interface)
  - [manifest.json contract](#manifestjson-contract)
  - [Loader lifecycle](#loader-lifecycle)
  - [Go plugin caveats](#go-plugin-caveats)
- [Built-in modules](#built-in-modules)
- [API conventions](#api-conventions)
- [Operational guidance](#operational-guidance)
  - [Production topology (recommended)](#production-topology-recommended)
  - [Observability (logging/metrics)](#observability-loggingmetrics)
  - [Security notes](#security-notes)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [FAQ / Troubleshooting](#faq--troubleshooting)
- [License](#license)

---

## What Heimdall is (and is not)

**Heimdall is:**

- A **management layer** (control plane) for DDI.
- A system that **centralizes workflows** and provides **one API** + **one UI**.
- A **module-driven platform**: engines are integrated through **Go plugins**.

**Heimdall is not:**

- A replacement for **BIND**, **Kea**, or **NetBox**.
- A “single binary that runs DNS/DHCP/IPAM” server.
- A tool that requires you to migrate away from your current engines.

---

## Philosophy

- **Bring your own engines** — Heimdall is a management layer, not a server. BIND, Kea, and NetBox run independently; Heimdall talks to them.
- **Modular by design** — each integration is a self-contained Go plugin (`.so`). Drop a new module into `/modules`, restart the service, and it becomes available.
- **API-first** — the core exposes a REST API; the frontend is just one consumer.

---

## Who this is for

Heimdall is designed for teams that:

- run “classic” DDI stacks (BIND + Kea + NetBox) and want a **single pane of glass**;
- want to **standardize** and **automate** common operations (records, reservations, prefixes);
- need extensibility via a plugin system (new providers, custom workflows).

Typical users:

- network engineers / sysadmins
- platform / SRE teams
- datacenter operations teams

---

## Core concepts

- **Core**: Go backend that provides the REST API and loads modules.
- **Module**: a Go plugin (`.so`) implementing a standard interface.
- **Capabilities**: strings describing what a module can do (e.g. `dhcp.leases`).
- **Module config**: module-specific configuration passed at initialization (from env/YAML).
- **Frontend**: React/Vite application that consumes the API.

---

## Architecture

High-level view:

```
+-----------+         +------------------+        +---------------------------+
|  Web UI   | <-----> | Heimdall Core     | <----> | Engines (existing systems) |
| (React)   |  HTTPS  | (Go REST API)     |  APIs  | BIND / Kea / NetBox       |
+-----------+         +------------------+        +---------------------------+
                              |
                              v
                       +---------------+
                       | Modules (.so) |
                       | bind/kea/...  |
                       +---------------+
```

Key idea: **Core owns the HTTP server**. Modules register routes and act as adapters to external engines.

---

## Tech stack

| Layer | Technology |
| --- | --- |
| Core / Engine | Go 1.22+ |
| Module system | Go plugin system (`.so` shared libraries) |
| Frontend | React + Vite (Node.js) |
| API | REST (JSON) |
| Config | YAML + environment variables |

---

## Repository layout

```text
heimdall/
│
├── core/                         # Go backend — engine & REST API
│   ├── cmd/
│   │   └── heimdall/
│   │       └── main.go           # Entrypoint
│   ├── internal/
│   │   ├── api/                  # HTTP router & handlers
│   │   │   ├── router.go
│   │   │   └── handlers/
│   │   ├── loader/               # Plugin discovery & dynamic loading
│   │   │   └── loader.go
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
│   │   ├── components/
│   │   ├── pages/
│   │   ├── store/
│   │   └── api/
│   ├── package.json
│   └── vite.config.ts
│
├── modules/                      # DDI integration modules (Go plugins)
│   ├── bind/
│   │   ├── manifest.json
│   │   ├── bind.go
│   │   ├── client.go
│   │   ├── handlers.go
│   │   └── README.md
│   ├── kea/
│   │   ├── manifest.json
│   │   ├── kea.go
│   │   ├── client.go
│   │   ├── handlers.go
│   │   └── README.md
│   └── netbox/
│       ├── manifest.json
│       ├── netbox.go
│       ├── client.go
│       ├── handlers.go
│       └── README.md
│
├── docs/                         # Documentation
│   ├── architecture.md
│   ├── module-development.md
│   └── api-reference.md
│
├── Makefile                      # build, build-modules, run, docker
├── docker-compose.yml
├── .env.example
└── README.md
```

> Note: some files listed above may be planned but not present yet; this README describes the intended structure.

---

## Getting started (development)

### Requirements

- Go **1.22+**
- Node.js **20+**
- Access to at least one running engine (BIND, Kea, NetBox) — you can develop modules independently, but end-to-end testing needs a target.

### Build & run (local)

```bash
# Build the core
make build

# Build all modules as .so
make build-modules

# Run locally
make run
```

Frontend (dev server):

```bash
cd frontend
npm install
npm run dev
```

---

## Configuration

Heimdall uses **environment variables** and/or **YAML** (depending on the core config implementation).

### Suggested baseline variables

These names match the current README examples; treat them as the baseline contract:

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

### Recommended config rules

- Keep secrets (tokens/keys) out of git:
  - use `.env` locally
  - use your secrets manager in production
- Prefer **read-only** tokens where possible (NetBox).
- Run Heimdall behind a reverse proxy (TLS termination) when exposed.

---

## Module system

Modules are compiled as Go shared libraries (`.so`) and loaded dynamically at startup via the Go plugin system.

### Module interface

Each module must implement the `Module` interface (defined in `core/internal/models/module.go`):

```go
type Module interface {
    Name()         string
    Version()      string
    Capabilities() []string
    RegisterRoutes(router Router)
    Init(cfg Config) error
    Shutdown() error
}
```

**Guidance for implementers:**

- `Name()` should match `manifest.json` `name`.
- `Capabilities()` should be stable strings (treat as an API contract).
- `Init()` should validate config and verify connectivity (if appropriate).
- `RegisterRoutes()` should register module endpoints under a predictable prefix.

### `manifest.json` contract

Each module directory contains a `manifest.json` used by the loader before opening the `.so`.

Example:

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

**Recommended rules:**

- `entry` should be a filename relative to the module directory.
- `capabilities` should be namespaced (`dns.*`, `dhcp.*`, `ipam.*`).
- `config_schema` is optional but strongly recommended to support:
  - UI forms
  - config validation
  - future automation / docs generation

### Loader lifecycle

On startup, the loader typically:

1. Scans `HEIMDALL_MODULES_DIR` for module directories containing a `manifest.json`
2. Reads and validates `manifest.json`
3. Opens the `.so` via `plugin.Open()`
4. Looks up the exported `NewModule` symbol
5. Calls `Init()` with the module config
6. Calls `RegisterRoutes()` to mount the module endpoints on the main router

**Hot reload:** Go plugins cannot be unloaded; restarting the core is required to reload updated modules.

### Go plugin caveats

Go plugins have constraints worth documenting explicitly:

- Plugin and core must be compiled with the **same Go version** and compatible dependency set
- Best supported on **Linux** (Windows is not supported)
- Once loaded, a plugin cannot be unloaded (restart required)
- Shared packages (e.g. `core/internal/models`) must be identical between core and plugin builds

---

## Built-in modules

The project currently includes modules for:

| Module | Engine | Typical responsibilities |
| --- | --- | --- |
| `bind` | BIND 9 | zones/records management, RNDC actions |
| `kea` | Kea DHCP | leases, reservations, subnets, stats |
| `netbox` | NetBox | prefixes, IP addresses, VLANs, device inventory (as needed) |

> Tip: keep a `modules/<name>/README.md` documenting:
> - required config values
> - permissions required on the target engine
> - supported operations and limitations

---

## API conventions

Heimdall is API-first. Even before a complete API reference exists, it’s useful to document conventions:

### Core endpoints (recommended)

- `GET /health` — liveness/readiness
- `GET /modules` — loaded modules + versions
- `GET /capabilities` — aggregated capabilities across modules

### Module routing conventions (recommended)

Pick a stable prefix strategy, for example:

- `/api/v1/<module>/...` (e.g. `/api/v1/kea/leases`)

or

- `/api/v1/<domain>/...` (e.g. `/api/v1/dhcp/leases`), where the module is an implementation detail.

**Recommendation:** start with module prefix (faster to iterate), later evolve into domain-based routes if you want a provider-agnostic API.

---

## Operational guidance

Even if the repo doesn’t ship deployment files yet, these guidelines help adopters.

### Production topology (recommended)

- Run Heimdall Core as a service behind a reverse proxy (Nginx/Traefik/Caddy)
- TLS termination at the proxy
- Restrict outbound connectivity from Heimdall to:
  - BIND RNDC/API endpoints
  - Kea API
  - NetBox API
- Separate environments (dev/stage/prod) with separate configs and secrets

### Observability (logging/metrics)

Recommended baseline:

- Structured logs (JSON)
- Request IDs
- Per-module log namespace (e.g. `module=kea`)

Future:

- Prometheus metrics endpoint
- Tracing (OpenTelemetry)

### Security notes

Until authentication/authorization is implemented (roadmap item), treat the API as **trusted-internal**:

- Do not expose Heimdall directly to the public Internet
- Put it behind VPN / internal network
- Add a reverse proxy auth layer if needed (basic auth / OIDC gateway)
- Treat NetBox token and RNDC key material as secrets

---

## Roadmap

- [x] Core API & module loader
- [x] Kea DHCP module
- [x] BIND module
- [x] NetBox module
- [ ] Web UI (React)
- [ ] Authentication (local + LDAP)
- [ ] Audit log
- [ ] Webhook/event system for cross-module sync

Suggested next milestones (optional):

- [ ] Stable API versioning (`/api/v1`)
- [ ] Per-module configuration UI from `config_schema`
- [ ] CI build matrix for core + modules (same Go version)
- [ ] Release process (tags, checksums)

---

## Contributing

Contributions are welcome.

Until `CONTRIBUTING.md` exists, suggested workflow:

1. Fork the repo
2. Create a feature branch
3. Keep PRs small and focused
4. Include docs updates when you change behavior

If you’re adding a new module:

- Create `modules/<name>/`
- Add a `manifest.json`
- Implement the `Module` interface
- Export `NewModule() models.Module`

---

## FAQ / Troubleshooting

### “My module doesn’t load”

Common causes:

- wrong `.so` path in `manifest.json` (`entry`)
- wrong `HEIMDALL_MODULES_DIR`
- core and module compiled with different Go versions/dependencies

### “Can I hot-reload modules?”

Not safely with Go plugins: once loaded, a plugin cannot be unloaded. Restart is required.

### “Does this work on Windows?”

Go plugin support is effectively Linux-first; plan deployments on Linux.

---

## License

MIT
