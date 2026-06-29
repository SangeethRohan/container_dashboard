# Zeno — User Guide

Zeno is a local container dashboard for managing Docker workloads from your browser.

## Getting started

1. Start the stack (see [Developer Guide](DEVELOPER.md) for setup).
2. Open **http://localhost:9090**
3. Sign in or use **Create account** on the login page to register a normal user account.

Default admin credentials (first install): **admin** / **admin** — change these in production.

## Roles

| Role | What you can do |
|------|-----------------|
| **Admin** | Full container control, user management, tier features, activity logs, edition settings |
| **User** | Manage your own containers; **Core Apps are view only**; create resources allowed by your tier |

### Core Apps (view only for normal users)

Containers in **Core Apps** (e.g. `zeno_dashboard`, `zeno_mongo`) are **view only** for normal users — you can see logs and stats but cannot start, stop, restart, delete, or use the Docker CLI on them.

All other containers (your databases, Ubuntu servers, web servers, and the rest of the stack) can be **started, stopped, restarted, deleted**, and controlled via **Docker CLI** when permitted.

## Dashboard

- **Dashboard** — All containers grouped by type or your custom groups. The top-bar **Alerts** badge shows the count of active alerts you have enabled in Settings; grey when zero, red when one or more. Critical enabled alerts also appear as a banner.
- **Host** — Host CPU, memory, and disk graphs, load average, and detailed system information.
- **Timeline** — What happened in the last 24 hours: operations, container state changes, and alerts.
- **Central Logs** — Your activity log and container log search.
- **Alerts** — Container CPU, memory, crash loop, and port failure alerts.
- **Create Database** — Postgres, MySQL, MongoDB, or Redis with optional schema (if enabled for your tier).
- **Create Ubuntu Server** — Dev sandbox with optional language toolchains and samples in `/workspace` (if enabled for your tier).
- **Create Web Server** — Nginx, Apache, Caddy, or Traefik (if enabled for your tier).
- **Manage Groups** — Drag containers between groups and reorder groups; layout is saved per user.

Create menu items you do not have access to are hidden automatically based on your edition tier.

Click a container row to expand **logs**, **live stats**, **24h metric charts** (CPU, RAM, disk I/O), and (if permitted) **Docker CLI**.

### Historical metrics

Expanded container rows show sparkline charts for the last 24 hours:

- **CPU** — percentage usage over time
- **RAM** — memory usage percentage
- **Disk I/O** — block read rate (Docker reports block I/O, not filesystem usage)

The **Host** view shows similar trends for system CPU, memory, and disk usage.

### Central logs

Under **Central Logs** in the sidebar:

- **Your activity log** — permanent record of your container actions (cannot be deleted)
- **Container logs** — pick a group and container from your saved layout, search lines, and refresh

### Alerts

Under **Alerts** in the sidebar:

| Alert | Condition |
|-------|-----------|
| CPU high | Container CPU above threshold (default 90%) for two checks |
| Memory high | Container memory above threshold (default 90%) for two checks |
| Crash loop | Container stuck in a restart loop |
| Port failure | Published host port unreachable while running |

Set CPU and memory thresholds on the Alerts page. Each alert records the CPU and memory % at the time it fired. Only **container** alerts appear here (host threshold events are tracked separately).

The topbar **Alerts** badge opens this page when active alerts exist.

### Docker CLI terminal

When allowed, click **Open terminal** to run shell commands inside a running container.

## Profile menu

- **Profile** — Your username, role, and edition tier.
- **Settings** — App version, environment info, and **change password** (requires current password).
- **Manage Users** — Admin only. User dashboard, bulk tier/delete, per-user activity logs.
- **Logout**

Admins also see **Tier Features** in the sidebar to enable or disable create flows per edition.

## Editions (Core / Pro / Elite)

Each user has an edition tier shown next to **Zeno** in the sidebar. New registrations receive the **Core** edition by default. Admins assign tiers per user under **Manage Users** (row dropdown or bulk **Apply tier**).

Admins can set their own personal edition under **Settings → Admin edition** (primary admin tier is fixed).

### Tier features (admin)

Under **Tier Features**, admins can turn features on or off per tier:

| Feature | Description |
|---------|-------------|
| Create Database | Access to the database provisioning flow |
| Create Ubuntu Server | Access to Ubuntu sandbox creation |
| Create Web Server | Access to web server creation |

Disabled features are hidden from the sidebar and blocked on the API for users on that tier. Admins always have all features.

## Manage Groups

Each user has their own dashboard group layout:

- **Drag containers** between groups (drop on a group zone or on another container to insert before it).
- **Reorder containers** within a group by dropping on another chip in the same group.
- **Reorder groups** using the **⋮⋮** handle on group headers.
- **Core Apps** is locked — containers there cannot be moved.
- Click **Save layout** to persist changes to your account.

## Manage Users (admin)

Open from **Profile → Manage Users**. The page includes:

- **Summary cards** — Total users, containers created/deleted, operations logged.
- **Create user** — Username, password, role, and tier.
- **User table** — Select multiple users with checkboxes (primary admin excluded).
- **Bulk actions** — Apply tier to selection or delete selected users.
- **Per-user tier** — Change tier from the row dropdown (primary admin tier is fixed).
- **Password** — Reset any user’s password, including the primary admin.
- **Activity log** — Click a username or row to see timestamped operations: creates, deletes, start/stop/restart, and CLI commands.

### Primary admin account

The seed admin account (from `DASHBOARD_USER`, usually **admin**) is marked **primary**:

- Cannot be deleted (including bulk delete).
- Tier and role cannot be changed.
- Only **Password** reset is available in the user table.
- Shown with a **primary** badge and no selection checkbox.

## Settings

All users can change their own password under **Settings → Change password**. You must enter your current password; new passwords must be at least 4 characters.

The **Application** card shows product version, host, MongoDB status, your edition, role, and dashboard URL.

Under **Notifications**, toggle each alert type on or off for the **dashboard** badge and banner. Disabled types still appear on the **Alerts** page. Preferences are saved per user when you flip a switch.

**Admins** (non-primary) can set their personal edition under **Admin edition** — Core, Pro, or Elite. The hint notes that **Core** is the default tier for newly registered users; your selection only changes your own admin edition. The primary admin account cannot change tier from Settings.

## Creating resources

### Database

Pick an engine, name, port (1024–65535), credentials, and optional tables/collections. Containers appear under **My Databases**.

### Ubuntu server

Optional languages install toolchains and sample files under `/workspace/samples`. Use **Open terminal** on the container row to run commands inside the sandbox.

### Web server

Choose type and port (or leave port empty for auto-assignment). Use **Open ↗** on the dashboard when the server is running.

## Security notes

- The dashboard mounts the Docker socket and can control the host. Do not expose port 9090 to the internet without strong authentication.
- Use unique passwords; registration creates **user** role accounts only.
- Change the default admin password after install.
- Session cookies are used for login; use HTTPS in production.

## Troubleshooting

| Issue | What to try |
|-------|-------------|
| Port in use | Pick another host port or stop the conflicting service |
| Cannot stop/restart Core App | Normal users: Core Apps are view only; ask an admin |
| Terminal unavailable | Container must be running; Core Apps have no CLI for normal users |
| Create menu missing | Your tier may not include that feature; ask an admin |
| MongoDB connecting | Ensure `zeno_mongo` is healthy (`docker compose ps`) |
| Wrong password on change | Settings requires the correct current password |
| CPU alert firing | Check container load; alert clears when CPU drops below 80% |
| Port failure alert | Verify the service inside the container is listening on the published port |
| No metric charts yet | Historical data collects every 60s; wait a few minutes after first start |
| Logs search empty | Broaden search or increase tail size; only matching lines are shown |

For technical setup and API details, see [Developer Guide](DEVELOPER.md).
