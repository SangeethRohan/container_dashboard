# Zeno — Developer Guide

Architecture, setup, configuration, and API reference for the Zeno container dashboard.

## Project layout

```
container-dashboard/
├── docker-compose.yml              # Main compose (dashboard service)
├── backend/
│   ├── app.py                      # Flask API + Docker integration
│   ├── db.py                       # MongoDB users, tiers, activity, layouts, features
│   ├── docker-compose.mongo.yml    # MongoDB overlay
│   ├── Dockerfile
│   ├── requirements.txt
│   └── static/                     # Frontend (HTML/CSS/JS)
├── docs/
│   ├── USER.md
│   └── DEVELOPER.md
└── README.md
```

## Running locally

```bash
docker compose -f docker-compose.yml -f backend/docker-compose.mongo.yml up -d --build
```

Dashboard: **http://localhost:9090**

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `DASHBOARD_USER` | `admin` | Seed admin username (marked `is_primary`) |
| `DASHBOARD_PASS` | `admin` | Seed admin password |
| `SECRET_KEY` | (dev key) | Flask session signing |
| `MONGO_URI` | `mongodb://zeno:zenopass@zeno_mongo:27017/zeno?authSource=admin` | MongoDB connection |
| `MONGO_DB` | `zeno` | Database name |
| `APP_TIER` | `Core` | Default tier for new users / migration |
| `METRICS_ENABLED` | `true` | Enable background metrics and alert collector |
| `MONITOR_INTERVAL_SEC` | `60` | Collector interval in seconds |

## Architecture

```
Browser → Flask (zeno_dashboard) → Docker socket
                ↓
            MongoDB (zeno_mongo)
              - users (auth, role, tier, is_primary)
              - settings (default_tier, tier_features)
              - activity_log (audit trail)
              - metrics_history (time-series snapshots, 7-day TTL)
              - alerts (CPU / port failure events)
              - group_layouts (per-user dashboard grouping)
```

Background collector (daemon thread in `app.py`):

```
Every MONITOR_INTERVAL_SEC (default 60s):
  → Sample host + running container stats → metrics_history
  → Detect container status changes → activity_log (state_change)
  → Evaluate CPU > 90% and port reachability → alerts
```

### Container labeling

User-created resources are tagged for grouping:

| Label `zeno.app.kind` | Group | Prefix |
|----------------------|-------|--------|
| `user-db` | My Databases | `zeno_userdb_` |
| `user-server` | My Servers | `zeno_ubuntu_` |
| `user-web` | My Web Servers | `zeno_web_` |

`zeno.created_by` stores the username that created the container.

### Authorization model

- **Admin** (`role: admin`): all container actions, user management, tier features, activity APIs.
- **User** (`role: user`): `can_manage` is false when `group === "Core Apps"` (view only); true for all other groups.
- **Feature gating**: non-admin users need tier-enabled features for create flows (`create_database`, `create_ubuntu`, `create_web_server`). Admins bypass feature checks.

Enforced in:

- `can_manage_container_group()` / `can_manage_container_obj()` in `app.py`
- `@require_feature(...)` on create endpoints
- `can_manage` flag on each item from `GET /api/v1/containers`
- Frontend hides actions, terminal, and nav items when not permitted

### Primary admin

The seed user from `DASHBOARD_USER` is stored with `is_primary: true` in MongoDB. This account:

- Cannot be deleted (`delete_user` raises)
- Cannot have `tier` or `role` changed via `update_user`
- Is excluded from bulk tier changes (`bulk_set_tier`)
- Can still have password reset via admin or self-service password change

### Per-user group layouts

Each user has a document in `group_layouts`:

```json
{
  "username": "alice",
  "layout": {
    "groups": [
      { "id": "core-apps", "name": "Core Apps", "order": 0, "locked": true },
      { "id": "my-databases", "name": "My Databases", "order": 1 }
    ],
    "assignments": { "zeno_userdb_app": "my-databases" },
    "container_order": { "my-databases": ["zeno_userdb_app"] }
  }
}
```

Core Apps (`core-apps`) is always locked; core container assignments are forced on save.

### Tier features

Stored in `settings` with key `tier_features`:

```json
{
  "Core": { "create_database": true, "create_ubuntu": false, "create_web_server": false },
  "Pro": { "create_database": true, "create_ubuntu": true, "create_web_server": false },
  "Elite": { "create_database": true, "create_ubuntu": true, "create_web_server": true }
}
```

Feature keys: `create_database`, `create_ubuntu`, `create_web_server`.

`GET /api/v1/me` returns `features` map for the current user. Admins receive all `true`.

### Activity logging

`db.log_activity()` writes to `activity_log`:

```json
{
  "username": "alice",
  "action": "create|delete|start|stop|restart|exec|state_change",
  "container": "zeno_ubuntu_dev",
  "container_image": "ubuntu:24.04",
  "details": "ubuntu:python,go",
  "ts": "2026-06-29T14:00:00+00:00"
}
```

Logged from container lifecycle endpoints, create-server/database flows, and the monitoring collector (`state_change`).

### Metrics history

`metrics_history` documents (TTL 7 days):

```json
{
  "container": "zeno_userdb_app",
  "ts": "2026-06-29T14:00:00+00:00",
  "cpu": 12.5,
  "mem_used_mb": 128,
  "mem_limit_mb": 512,
  "block_read_bytes": 0,
  "block_write_bytes": 0
}
```

Host snapshots use `container: "__host__"` with disk fields stored in `block_read_bytes` / `block_write_bytes`.

### Alerts

`alerts` collection:

```json
{
  "rule": "cpu_high|port_failure",
  "container": "zeno_web_app",
  "message": "CPU above 90% (92.1%) on zeno_web_app",
  "severity": "warning|critical",
  "ts": "2026-06-29T14:00:00+00:00",
  "resolved": false,
  "resolved_at": null
}
```

Rules: CPU > 90% for 2 consecutive checks; published host port TCP connect failure while running. Dedup window: 15 minutes per rule+container while unresolved.

### Docker CLI session cwd

Per `(username, container)` working directory is tracked in `_terminal_cwd` in `app.py`. `cd` and `pwd` are handled server-side; other commands run with `workdir=cwd`.

## API reference

Base path: `/api/v1`  
Auth: session cookie (`credentials: include` on fetch). Unauthenticated requests return `401`.

### Auth

| Method | Path | Description |
|--------|------|-------------|
| POST | `/login` | `{ username, password }` |
| POST | `/logout` | Clear session |
| POST | `/register` | Public signup → `role: user`, default tier |
| GET | `/me` | Current user, tier, `is_admin`, `features`, `feature_labels` |
| POST | `/account/password` | `{ current_password, new_password }` — self password change |
| PUT | `/account/tier` | Admin: set own personal edition `{ tier }` — blocked for primary admin |

### Containers

| Method | Path | Description |
|--------|------|-------------|
| GET | `/containers` | List all; includes `can_manage`, `group`, `group_id`, `created_by` |
| POST | `/containers/<name>/<action>` | `start`, `stop`, `restart` (permission checked) |
| DELETE | `/containers/<name>` | Remove stopped container |
| GET | `/containers/<name>/logs` | Tail logs (`?tail=200`) |
| GET | `/containers/<name>/stats` | CPU, memory, network, block I/O |
| POST | `/containers/<name>/exec` | `{ command }` — CLI exec; returns `cwd` |

### Observability

| Method | Path | Description |
|--------|------|-------------|
| GET | `/metrics/history` | `?container=<name>&hours=24` — time-series points |
| GET | `/timeline` | `?hours=24&limit=200` — merged activity, state, alerts |
| GET | `/activity/me` | Current user's activity log (read-only) |
| GET | `/alerts` | `?hours=24&containers_only=true` — container alerts with thresholds |
| GET | `/alerts/thresholds` | CPU and memory % thresholds |
| PUT | `/alerts/thresholds` | `{ cpu_percent, mem_percent }` |
| GET | `/logs/central` | `?containers=a,b&search=error&tail=300` — up to 3 containers |

### Host

| Method | Path | Description |
|--------|------|-------------|
| GET | `/host/stats` | CPU %, memory %, disk %, load average |
| GET | `/host/details` | OS, kernel, disk, Docker version, etc. |

### Provisioning

Requires `@require_feature` unless caller is admin.

| Method | Path | Body highlights |
|--------|------|-----------------|
| POST | `/databases` | `engine`, `name`, `host_port`, credentials, `tables`, `persistent` |
| DELETE | `/databases/<name>` | `?remove_volume=true` optional |
| POST | `/servers/ubuntu` | `name`, `languages[]`, `persistent` |
| POST | `/servers/web` | `name`, `type`, `host_port?`, `persistent` |

### Group layouts (per user)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/groups/layout` | Current user layout + container list for editor |
| PUT | `/groups/layout` | Save `{ layout: { groups, assignments, container_order } }` |
| POST | `/groups` | `{ name }` — create custom group |
| DELETE | `/groups/<id>` | Delete group; reassigns containers |

### Users (admin)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users` | Users with stats; includes `is_primary` |
| GET | `/users/dashboard` | Summary + recent activity |
| POST | `/users` | Create user `{ username, password, role, tier }` |
| PATCH | `/users/<username>` | `{ tier?, role? }` — blocked for primary admin |
| PATCH | `/users/<username>/password` | Admin password reset `{ password }` |
| DELETE | `/users/<username>` | Delete user — blocked for primary admin |
| POST | `/users/bulk` | `{ action: "delete"\|"set_tier", usernames[], tier? }` |
| GET | `/activity` | `?username=&limit=&skip=` audit log |

### Settings & tier features

| Method | Path | Description |
|--------|------|-------------|
| GET | `/settings` | App info + user tier, `is_primary`, `default_tier` |
| PUT | `/account/tier` | Admin: set own personal edition (not global default) |
| PUT | `/settings/tier` | Set default tier for new registrations (admin; API only) |
| GET | `/admin/tier-features` | Feature matrix for all tiers (admin) |
| PUT | `/admin/tier-features` | `{ tier_features: { Core: { ... }, ... } }` (admin) |

## Frontend modules

| File | Role |
|------|------|
| `static/js/app.js` | Dashboard, host stats/graphs, timeline, logs, alerts, groups editor, tier features, create flows, terminal |
| `static/js/manage-users.js` | Admin user dashboard + activity log |
| `static/js/login.js` | Login + register |
| `static/js/settings.js` | Settings, password change, admin edition |
| `static/css/pages.css` | Standalone pages (settings, manage users) — forms, tables, page actions |
| `static/css/dashboard.css` | Main dashboard shell and components |

### UI patterns

- **Page actions** (`pages.css`): `.page-btn`, `.page-btn-primary`, `.page-btn-ghost`, `.page-btn-danger` for footer navigation on settings/manage-users pages.
- **Tier/role selects**: `.tier-select`, `.role-select` — themed native selects with custom arrow.
- **Activity log**: `.activity-entry.action-*` color-coded left borders.

## Extending

### Add a gated feature

1. Add key to `FEATURE_KEYS` and label in `FEATURE_LABELS` (`db.py`).
2. Add default in `default_tier_features()`.
3. Decorate the API route with `@require_feature("your_key")`.
4. Add nav item in `index.html` with `data-feature="your_key"`.
5. Call `applyFeatureNav()` from `/me` features in `app.js`.
6. Add row to Tier Features editor (auto from API `features` list).

### Add a web server type

1. Add image/port to `WEB_SERVER_DEFAULTS` in `app.py`.
2. Add UI option in `index.html` `#web-type-grid`.
3. Rebuild dashboard image.

### Change Core Apps (view-only group)

Edit `CORE_APP_NAMES` and grouping logic in `serialize()` so containers in **Core Apps** stay protected from normal-user lifecycle actions.

### Custom OPEN_LINKS

Map container names to host ports in `OPEN_LINKS` for quick **Open ↗** links.

## Rebuild after changes

```bash
docker compose -f docker-compose.yml -f backend/docker-compose.mongo.yml up -d --build dashboard
```

Static files are baked into the image (no bind mount for `static/`).

## Security checklist

- [ ] Change `DASHBOARD_PASS`, `SECRET_KEY`, Mongo credentials
- [ ] Do not publish `9090` without TLS and auth
- [ ] Docker socket = root-equivalent access
- [ ] Review activity logs for unexpected `exec` usage
- [ ] Limit admin accounts; protect primary admin credentials
- [ ] Review tier feature matrix before onboarding users

## MongoDB collections

- **users** — `username`, `password_hash`, `role`, `tier`, `is_primary`, `created_at`, `created_by`
- **settings** — `key: app_tier` (default tier), `key: tier_features` (feature matrix)
- **activity_log** — indexed on `username`, `ts`
- **group_layouts** — `username`, `layout` (groups, assignments, container_order)

End-user documentation: [USER.md](USER.md)
