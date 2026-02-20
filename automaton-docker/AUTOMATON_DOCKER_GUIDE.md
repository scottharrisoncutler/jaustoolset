# Conway Automaton — Docker Containerization Guide

This guide evaluates the [Conway-Research/automaton](https://github.com/Conway-Research/automaton) project and provides step-by-step instructions for building and running it inside a Docker container so that it cannot compromise any other part of your computer.

---

## 1. Repository Evaluation

### What It Is

The **automaton** is a self-improving, self-replicating, autonomous AI agent runtime written in TypeScript/Node.js. On first boot it:

1. Generates an Ethereum wallet (on Base L2).
2. Provisions itself a Conway Cloud API key via Sign-In With Ethereum (SIWE).
3. Enters a continuous **Think → Act → Observe** loop driven by a "genesis prompt."

It has the ability to:

- Execute arbitrary shell commands on its host.
- Read and write files, including its own source code.
- Make HTTP/HTTPS requests to external services.
- Perform on-chain cryptocurrency transactions.
- Spawn child automaton instances ("self-replication").

### Technology Stack

| Component | Detail |
|---|---|
| Language | TypeScript (ES2022, NodeNext modules) |
| Runtime | Node.js ≥ 20 |
| Package manager | pnpm (npm also works for initial install) |
| Database | SQLite via `better-sqlite3` |
| Crypto | `viem` (Ethereum), `siwe` (Sign-In With Ethereum) |
| Build tool | `tsc` (TypeScript compiler) |

### Security Considerations

Because the automaton is **designed** to run arbitrary code, modify its own files, and interact with the network, running it directly on your host machine is risky. Specific concerns include:

| Risk | Description |
|---|---|
| **Arbitrary code execution** | The agent loop can run any shell command the host user can. |
| **File-system access** | It reads/writes files — including its own source — and could access host files. |
| **Network access** | It makes outbound HTTP calls and on-chain transactions. |
| **Self-modification** | It can install packages, edit code, and change its own behavior at runtime. |
| **Self-replication** | It can spin up child processes or VMs if given access to Conway Cloud. |
| **Cryptocurrency wallet** | A private key is generated and stored locally; leaking it means losing funds. |

**Running in a Docker container addresses all of these** by providing process isolation, filesystem sandboxing, and controllable network access.

---

## 2. Prerequisites

| Tool | Minimum Version | Install |
|---|---|---|
| Docker Engine | 24+ | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |
| Docker Compose | v2 (bundled with Docker Desktop) | Included with Docker Desktop or install the plugin |
| Git | any | For cloning this repo |

Verify your installation:

```bash
docker --version        # Docker version 24.x or later
docker compose version  # Docker Compose version v2.x
```

---

## 3. Quick Start

```bash
# 1. Clone this repository (if you haven't already)
git clone https://github.com/scottharrisoncutler/jaustoolset.git
cd jaustoolset/automaton-docker

# 2. Build the container image
docker compose build

# 3. Start the automaton interactively (first-run setup wizard)
docker compose run --rm automaton --run
```

The first-run wizard will prompt you for a name, genesis prompt, and creator address. After setup, the agent loop starts automatically.

To run in the background after initial setup:

```bash
docker compose up -d
docker compose logs -f   # follow logs
```

---

## 4. File Overview

```
automaton-docker/
├── Dockerfile            # Multi-stage build: compile + slim runtime
├── docker-compose.yml    # Orchestration with security constraints
├── .env.example          # Template for secrets (copy to .env)
└── AUTOMATON_DOCKER_GUIDE.md   # This file
```

---

## 5. What the Dockerfile Does

The Dockerfile uses a **multi-stage build**:

### Stage 1 — Builder

- Starts from `node:20-slim`.
- Installs build tools (`git`, `python3`, `make`, `g++`) required by native Node.js modules like `better-sqlite3`.
- Clones the automaton repo at `--depth 1` (minimal history).
- Runs `pnpm install` and `pnpm build` to compile TypeScript.

### Stage 2 — Runtime

- Starts from a fresh `node:20-slim` (build tools are **not** carried over).
- Copies only the compiled application from the builder stage.
- Creates a dedicated non-root user (`automaton`) so the process cannot escalate privileges.
- Declares a `VOLUME` for persistent state (`~/.automaton/`).
- Runs as user `automaton` — **never root**.

---

## 6. Security Hardening (docker-compose.yml)

The compose file applies multiple layers of defense:

| Setting | Purpose |
|---|---|
| `cap_drop: ALL` | Drops every Linux capability (no raw sockets, no `chown`, no `ptrace`, etc.). |
| `no-new-privileges` | Prevents the process from gaining privileges via `setuid` binaries or other means. |
| `read_only: true` | Root filesystem is read-only; writes are only possible to explicitly mounted volumes and tmpfs. |
| `mem_limit: 1g` | Caps memory at 1 GB — prevents the container from consuming all host RAM. |
| `cpus: 1.0` | Limits to 1 CPU core — prevents the container from starving other processes. |
| `tmpfs` mounts | Size-limited RAM-backed temporary directories for Node.js caches. |
| Named volume | Persistent state is stored in a Docker-managed volume, **not** on your host filesystem. |
| Ports commented out | No ports are exposed by default; uncomment only if you need the automaton to serve HTTP. |

### Optional: Restrict Network Access

To prevent the automaton from reaching the internet entirely (useful for offline testing):

```bash
# Create an isolated network with no external access
docker network create --internal automaton-net

# Run the container on that network
docker compose run --rm --network automaton-net automaton --run
```

To allow only specific outbound destinations, use Docker's built-in firewall rules or a reverse proxy.

---

## 7. Managing Secrets

1. Copy the template:

   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and fill in any values you want to pre-configure.

3. Uncomment the `env_file` line in `docker-compose.yml`:

   ```yaml
   env_file:
     - .env
   ```

4. **Never** commit `.env` to version control. The `.gitignore` in this directory covers it.

---

## 8. Inspecting and Controlling the Container

```bash
# View logs
docker compose logs -f

# Open a shell inside the running container
docker compose exec automaton /bin/bash

# Stop the automaton
docker compose down

# Stop and remove all data (wallet, SOUL.md, database)
docker compose down -v
```

---

## 9. Updating the Automaton

To pull the latest automaton source and rebuild:

```bash
docker compose build --no-cache
docker compose up -d
```

The persistent state volume is preserved across rebuilds.

---

## 10. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `pnpm install` fails in build | Network issue or lockfile mismatch | The Dockerfile falls back to `pnpm install` (without `--frozen-lockfile`) automatically. Retry the build. |
| `better-sqlite3` build error | Missing native build tools | Ensure the builder stage has `python3`, `make`, and `g++` (already included in the Dockerfile). |
| Container exits immediately | First-run wizard needs interactive input | Use `docker compose run --rm automaton --run` (not `up -d`) for the initial setup. |
| Permission denied on `/home/automaton/.automaton` | Volume ownership mismatch | Run `docker compose down -v` and start fresh, or fix ownership with a one-time `docker compose run --rm -u root automaton chown -R automaton:automaton /home/automaton/.automaton`. |
| Out of memory | Container hit 1 GB limit | Increase `mem_limit` in `docker-compose.yml`. |

---

## 11. Summary of Isolation Guarantees

By running the automaton in this Docker setup, you get:

- **Process isolation** — the automaton's processes cannot see or interact with host processes.
- **Filesystem sandboxing** — only the Docker-managed volume is writable; your host files are inaccessible.
- **Resource limits** — CPU and memory caps prevent denial-of-service against the host.
- **Privilege restriction** — the container runs as a non-root user with all Linux capabilities dropped.
- **Network control** — ports are not exposed by default, and you can further restrict network access with Docker's `--internal` networks.
- **Clean teardown** — `docker compose down -v` removes all container state, leaving no trace on the host.
