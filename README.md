# Zeno — local container dashboard

A dashboard for your dev stack: container status, start/stop/restart (role-based), live logs, Docker CLI terminal, and provisioning for databases, Ubuntu dev servers, and web servers.

- **User guide:** [docs/USER.md](docs/USER.md)
- **Developer guide:** [docs/DEVELOPER.md](docs/DEVELOPER.md)

## Quick start

```bash
docker compose -f docker-compose.yml -f backend/docker-compose.mongo.yml up -d --build
```

Open **http://localhost:9090** — login **admin** / **admin** (change in production).

## Features

- MongoDB-backed users with **admin** and **user** roles
- Per-user **Core / Pro / Elite** edition tier (admin assigns in Manage Users; new registrations default to **Core**)
- **Historical metrics** — CPU, RAM, and disk I/O trends per container and host
- **System timeline** — unified view of operations, state changes, and alerts (last 24h)
- **Central logs viewer** — search and filter logs across up to 3 containers
- **Alert system** — CPU above 90%, published port failures
- **Activity audit log** — container creates, deletes, and operations per user
- Normal users: **Core Apps are view only**; manage all other containers; admins: full control
- Create Database, Ubuntu Server, Web Server from the sidebar

## Docker socket warning

The dashboard mounts `/var/run/docker.sock` and has effective root-level Docker access. Use auth, keep the port local or behind a reverse proxy, and treat credentials like host root access.

See [docs/DEVELOPER.md](docs/DEVELOPER.md) for API reference and configuration.
