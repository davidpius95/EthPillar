# EthPillar Architecture & Flow

This document explains what happens when you run `ethpillar`, how the app is structured, which files handle which responsibilities, and how they interact. It also includes pictorial diagrams of the end-to-end flows.

## High‑Level Overview

EthPillar is a terminal UI (TUI) for installing and managing Ethereum nodes, validators, and related plugins. It drives two kinds of runtimes:
- Systemd‑managed clients: Execution, Consensus, Validator, MEV‑Boost
- Docker‑managed plugins: e.g., Aztec Sequencer, Dora, Sentinel, etc.

Key entry points and responsibilities:
- `ethpillar.sh:1`: Main CLI/TUI. Orchestrates flows, menus, status, and integrations.
- `functions.sh:176`: Shared helpers (run scripts, network detection, UI helpers, status, exports).
- `install-node.sh:1`: Wrapper to install a node using a specific Python deployer.
- `deploy-*.py`: Python deployers that install selected EL/CL stacks and write systemd service files.
- `plugins/*`: Optional plugins (some are Dockerized, others add extra systemd services).

Paths used at runtime:
- `/etc/systemd/system/*.service`: Systemd service units for EL/CL/VC/MEV‑Boost and some plugins.
- `/opt/ethpillar/<plugin>`: Plugin install roots (env + docker-compose, data mounts).

---

## First Run Flow

What happens when you type `ethpillar` for the first time:

1) Main entry and environment load
- `ethpillar.sh:1` starts and sources helpers/env.
- `ethpillar.sh:1650` calls: `checkV1StakingSetup`, `setWhiptailColors`, `installNode`, `applyPatches`, `checkDiskSpace`, `checkCPULoad`, `setNodeMode`, `initializeNetwork`, then `menuMain()`.

2) Node installation workflow (if nothing installed)
- `installNode()` at `ethpillar.sh:1577` runs when no EL/CL/VC services and no Aztec are present.
- Displays a combo picker (EL/CL preset or Aztec). If you select a client combo:
  - `install-node.sh` is invoked with a matching deployer, e.g. `deploy-nimbus-nethermind.py`.
  - The deployer downloads binaries and writes systemd units:
    - Execution: `/etc/systemd/system/execution.service`
    - Consensus: `/etc/systemd/system/consensus.service`
    - Validator: `/etc/systemd/system/validator.service` (when selected)
    - MEV‑Boost: `/etc/systemd/system/mevboost.service` (when selected)
    - See examples around: `deploy-nimbus-nethermind.py:567`, `:709`, `:781`.

3) Main menu appears
- `menuMain()` at `ethpillar.sh:71` builds the menu dynamically based on installed services and plugins.
- Node mode banner and network endpoints are computed from `setNodeMode()` at `ethpillar.sh:1619` and `initializeNetwork()` at `ethpillar.sh:37`.

---

## Main Menu & Controls

The main TUI groups operations for services and plugins:

- “Start/Stop/Restart all clients”
  - Systemd services: `testAndServiceCommand()` → `sudo service ... start/stop/restart` (`ethpillar.sh:78`).
  - Plugins: `testAndPluginCommand()`
    - `start` → `docker compose up -d`
    - `stop`  → `docker compose stop`
    - `restart` → `docker compose restart`
    - See `ethpillar.sh:84`.

- Execution / Consensus / Validator / MEV‑Boost submenus
  - `ethpillar.sh:330`, `:396`, `:462`, `:566` handle service‑specific logs, start/stop/restart, resync/update.

- Plugins submenu
  - `submenuPlugins()` at `ethpillar.sh:1336` lists optional plugins and runs their installers/menus via `runScript()` (`functions.sh:176`).

- Logging & Monitoring
  - Log dashboard (tmux panes) is launched via `view_logs.sh` (`ethpillar.sh:238`).
  - “Rolling Consolidated Logs” tails systemd logs; includes Aztec docker logs when present (`ethpillar.sh:241`).
  - “Monitoring” manages Prometheus/Grafana/Exporter services (`ethpillar.sh:778`).

---

## Deployers and Services

When you install an EL/CL combo, `install-node.sh` executes a `deploy-*.py` script which:
- Downloads/installs clients (e.g., Nethermind, Nimbus, Lighthouse, Teku, Besu, Reth).
- Writes systemd unit files for EL/CL/VC/MEV‑Boost.
- Optionally prompts to start and enable autostart.

Systemd service examples (created by deployers):
- Execution: `/etc/systemd/system/execution.service` (see `deploy-nimbus-nethermind.py:567`)
- Consensus: `/etc/systemd/system/consensus.service` (see `deploy-nimbus-nethermind.py:709`)
- Validator: `/etc/systemd/system/validator.service` (see `deploy-nimbus-nethermind.py:781`)
- MEV‑Boost: `/etc/systemd/system/mevboost.service` (around `deploy-nimbus-nethermind.py:416`)

The TUI then controls these with `sudo service ...` start/stop/restart.

---

## Plugins Architecture

Plugins live under `plugins/` and are integrated through the main menu:
- `ethpillar.sh:110`: Adds plugin tiles when their install dirs exist (e.g., `/opt/ethpillar/aztec`).
- `ethpillar.sh:1336`: “Plugins” submenu handles install/open of each plugin’s own menu.
- `functions.sh:176`: `runScript()` runs plugin scripts.

Common plugin patterns:
- Dockerized plugin (e.g., Aztec, Dora, Sentinel):
  - Installer writes to `/opt/ethpillar/<plugin>` with `.env` and `docker-compose.yml`.
  - Start/Stop is via `docker compose ...` (from the plugin menu and main menu bulk actions).
- Systemd plugin (e.g., CSM Nimbus Validator):
  - Installer writes `/etc/systemd/system/csm_nimbusvalidator.service`.
  - Managed like other services from the main menu.

Examples:
- Aztec Sequencer
  - Installer: `plugins/aztec/plugin_aztec.sh:108`
  - Menu: `plugins/aztec/menu.sh:366`
  - Runtime definition: `plugins/aztec/docker-compose.yml.example:1`
  - Config template: `plugins/aztec/.env.example:1`

- Dora (lightweight explorer)
  - Installer: `plugins/dora/plugin_dora.sh:1`
  - Systemd unit example template: `plugins/dora/dora.service.example:1`

- Node Checker (security/health)
  - CLI: `plugins/node-checker/run.sh:801` (invoked via menus under Security & Node Checks).

- CSM Sentinel (Telegram alerts; docker)
  - Installer/menu: `plugins/sentinel/*`

- Client Stats / Contributoor / eth-validator-cli
  - Similar plugin scaffolds under `plugins/`.

---

## Logging, Monitoring, and Utilities

- Log dashboard (tmux panes): `view_logs.sh:1`. Adapts layout to your node mode and includes docker logs for plugins.
- Export logs to files (journald services): `export_logs()` at `functions.sh:1285`.
- Monitoring suite (Prometheus, Grafana, exporters): controlled from the Monitoring submenu (`ethpillar.sh:778`).
- Toolbox & Security menus: UFW, Fail2Ban, 2FA, swappiness, noatime, speed test, etc. (all wired via `runScript()` helpers in `functions.sh`).

---

## File Responsibilities Map

- `ethpillar.sh:1` — Main TUI: menus, orchestration, status banner, plugin integration.
- `functions.sh:1` — Shared utilities: environment, UI helpers, script runner (`runScript()` at `functions.sh:176`), network detection, status dialogs, exports, checks.
- `install-node.sh:1` — Wrapper around Python deployers; sets up Python env/tools, runs `deploy-*.py`.
- `deploy-*.py` — Client installers; write systemd unit files (execution/consensus/validator/mevboost), download binaries, prompt for start/enable.
- `view_logs.sh:1` — Tmux log board; shows EL/CL/VC and plugin logs in panes.
- `update_execution.sh:1`, `update_consensus.sh:1`, `resync_*` — Update/maintenance helpers for clients.
- `plugins/*` — Plugin installers and menus; write `/opt/ethpillar/<plugin>`, docker compose configs or systemd units.

---

## Pictorial Diagrams

### Unified Architecture Overview

```mermaid
flowchart TD
  classDef node fill:#1f2937,stroke:#475569,rx:6,ry:6,color:#fff;
  classDef file fill:#0ea5e9,stroke:#0369a1,rx:6,ry:6,color:#fff;
  classDef svc fill:#16a34a,stroke:#065f46,rx:6,ry:6,color:#fff;
  classDef docker fill:#2563eb,stroke:#1e40af,rx:6,ry:6,color:#fff;
  classDef store fill:#334155,stroke:#64748b,rx:6,ry:6,color:#fff;

  subgraph CLI[EthPillar CLI]
    E[ethpillar.sh]:::file
    F[functions.sh]:::file
  end
  E --> F
  E --> INIT[initializeNetwork • setNodeMode • applyPatches]:::node
  E --> MENU[menuMain]:::node

  MENU -->|Services| SM[Service Menus - Execution, Consensus, Validator, MEV-Boost]:::node
  MENU -->|Plugins| PM[Plugins Menu]:::node
  MENU --> LOGS[Log Dashboard - view_logs.sh]:::file
  MENU --> MON[Monitoring]:::node

  E -->|First run| INSTALL[installNode]:::node
  INSTALL --> CHOICE{Pick Stack}:::node

  %% Systemd path
  CHOICE -->|EL/CL preset| SH[install-node.sh]:::file
  SH --> DP[deploy-*.py]:::file
  DP --> SYS[(systemd units: execution, consensus, validator, mevboost)]:::svc
  SYS --> RUNTIME1[Clients running as services]:::svc

  %% Plugin path (example: Aztec)
  CHOICE -->|Aztec plugin| AZI[plugins/aztec/plugin_aztec.sh -i]:::file
  AZI --> AZDIR[/opt/ethpillar/aztec - .env + docker-compose.yml/]:::store
  AZDIR --> DC[docker compose up -d]:::docker
  DC --> RUNTIME2[Aztec container]:::docker

  %% Bulk actions
  MENU --> ALL[Start / Stop / Restart all]:::node
  ALL -->|systemd| SYS
  ALL -->|docker| DC

  %% Logs & Monitoring
  LOGS --> J[(journald)]:::store
  LOGS --> DCL[Docker logs]:::docker
  MON --> G[(Grafana, Prometheus, Exporters)]:::svc

```

### A) End‑to‑End First Run Flow

```mermaid
flowchart TD
  U[User] --> E[ethpillar.sh:1 Main CLI]
  E --> C1[checkV1StakingSetup]
  E --> I1[installNode ethpillar.sh:1577]
  I1 --> Q{EL/CL/VC or Aztec installed?}
  Q -- No --> P1[Pick combo via whiptail]
  P1 --> D1[install-node.sh → deploy-*.py]
  D1 --> SVC[/Write systemd services\n execution/consensus/validator/mevboost/]
  P1 --> AZI[plugins/aztec/plugin_aztec.sh -i]
  AZI --> AZDIR[/Create /opt/ethpillar/aztec\n .env + docker-compose.yml/]
  E --> M[menuMain ethpillar.sh:71]
  E --> NM[setNodeMode ethpillar.sh:1619]
  E --> IN[initializeNetwork ethpillar.sh:37]
  M --> SMenus[Service Menus]
  M --> PMenu[Plugins Menu]

```

### B) Main Menu: Start/Stop/Restart All

```mermaid
sequenceDiagram
  participant U as User
  participant M as menuMain (ethpillar.sh:119)
  participant S as testAndServiceCommand
  participant P as testAndPluginCommand
  participant SYS as systemd
  participant DC as docker-compose

  U->>M: Select "Start all clients"
  M->>S: start
  S->>SYS: systemctl start execution/consensus/validator/mevboost
  M->>P: start
  P->>DC: docker compose up -d (for installed plugins)
```

### C) Plugin Install & Use (Generic)

```mermaid
sequenceDiagram
  participant U as User
  participant PM as submenuPlugins (ethpillar.sh:1336)
  participant PL as plugin installer (plugins/*/plugin_*.sh)
  participant FS as Filesystem
  participant RT as Runtime

  U->>PM: Choose a plugin
  alt Not installed
    PM->>PL: runScript plugin -i
    PL->>FS: Create /opt/ethpillar/<plugin>, write .env/compose or service unit
  end
  PM->>PL: Open plugin menu
  alt Dockerized plugin
    Note right of RT: Start/Stop via docker compose
  else Systemd plugin
    Note right of RT: Start/Stop via systemctl
  end
```

### D) Log Dashboard

```mermaid
flowchart LR
  U[User] --> LM["Logging & Monitoring → View Log Dashboard"]
  LM --> VLS[view_logs.sh]
  VLS -->|Reads| journald[systemd journal]
  VLS -->|Tails| dockerlogs[Docker logs plugins]
  VLS --> TMUX[tmux panes with ccze + btop]

```

---

## Configuration and Data

- Env and constants loaded via:
  - `env` (project env), optional `./.env.overrides`, and plugin `.env` files.
- Systemd units and data locations used by deployers:
  - Execution (e.g., Nethermind): `/var/lib/<client>`, service user `execution`.
  - Consensus (e.g., Nimbus): `/var/lib/<client>`, service user `consensus`.
  - Validator (Nimbus VC): `/var/lib/<client>`, service user `validator`.
- Plugins under `/opt/ethpillar/<plugin>` mount their own data directories (e.g., Aztec binds `/opt/ethpillar/aztec/data`).

---

## How Things Fit Together (At a Glance)

- CLI skeleton: `ethpillar.sh` calls helper functions in `functions.sh` and delegates to deployers/plugins using `runScript()` (`functions.sh:176`).
- EL/CL/VC stacks: Installed by `install-node.sh` → `deploy-*.py`, managed by systemd, controlled via ethpillar menus.
- Plugins: Installed under `/opt/ethpillar`, dockerized or systemd‑based, controlled by their menus and by main menu bulk actions.
- Logs/Monitoring: `view_logs.sh` and Monitoring submenu manage logs and Grafana/Prometheus exporters.

If you want more diagrams (e.g., per‑client stacks or specific plugin lifecycles), let me know and I’ll add them.
