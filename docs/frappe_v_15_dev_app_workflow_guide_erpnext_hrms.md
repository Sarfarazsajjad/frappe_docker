# Frappe v15 Development Guide (with ERPNext v15 + HRMS v15)

A practical, end‑to‑end guide to: set up a local Frappe dev environment (v15), install ERPNext & HRMS, open the Desk (web GUI), create your own app, iterate with hot‑reload, version your app with Git/GitHub, and finally install your app on any ERPNext/HRMS v15 deployment.

---

## 1) Prerequisites

- **OS**: macOS, Windows, or Linux
- **Docker Desktop** (allocate ≥ 4 GB RAM to Docker)
- **VS Code** with **Dev Containers** extension
- **Git** + a GitHub account (SSH keys set up recommended)
- **Ports**: 8000 (web), 9000–9005 (webpack/socket) should be free locally

> **Tip:** We’ll use the host/domain `development.localhost`. On macOS/Windows it should resolve automatically. On Linux, add `127.0.0.1 development.localhost` to `/etc/hosts` if needed.

---

## 2) Clone the frappe_docker repo and prepare the Dev Container

```bash
git clone https://github.com/frappe/frappe_docker.git
cd frappe_docker

# Copy the example devcontainer + VS Code settings
cp -R devcontainer-example .devcontainer
cp -R development/vscode-example development/.vscode
```

Open the folder in **VS Code** → Command Palette → **Dev Containers: Reopen in Container**.

This boots a development container and support services (Redis, MariaDB) defined in `.devcontainer/docker-compose.yml`.

---

## 3) Create a v15 bench inside the Dev Container

Open a terminal **inside** the Dev Container (user `frappe`).

```bash
cd /workspace/development
# IMPORTANT: pin Frappe major version to 15
bench init --frappe-branch version-15 --skip-redis-config-generation frappe-bench
cd frappe-bench
```

Point bench to the dockerized services:

```bash
bench set-config -g db_host mariadb
bench set-config -g redis_cache    redis://redis-cache:6379
bench set-config -g redis_queue    redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-queue:6379
```

Create your dev site:

```bash
bench new-site --mariadb-user-host-login-scope=% development.localhost
# Set the Administrator password when prompted (e.g., "admin" for convenience in dev)
```

Enable developer mode:

```bash
bench --site development.localhost set-config developer_mode 1
bench --site development.localhost clear-cache
```

---

## 4) Install ERPNext v15 and HRMS v15

**Always** fetch the correct major branch for each app.

```bash
# ERPNext v15
bench get-app --branch version-15 erpnext --resolve-deps
bench --site development.localhost install-app erpnext

# HRMS v15
bench get-app --branch version-15 hrms --resolve-deps
bench --site development.localhost install-app hrms
```

> If you ever accidentally installed Frappe 16-dev earlier and get version conflicts, see **Troubleshooting → Version mismatch** below.

---

## 5) Start the dev stack & open the Desk (web GUI)

```bash
bench start
```

Open: **http://development.localhost:8000**

Login as **Administrator** with the password you set while creating the site.

---

## 6) Create your app and install it on the site

```bash
cd /workspace/development/frappe-bench
bench new-app my_app
# Answer the prompts for title/description/etc.

bench --site development.localhost install-app my_app
```

Your app’s code lives at:

```
/workspace/development/frappe-bench/apps/my_app
```

Key locations:

- `my_app/hooks.py` — hooks (fixtures, scheduler events, overrides, etc.)
- `my_app/my_app/doctype/` — DocTypes, server-side controllers
- `my_app/public/` — static assets (JS/CSS)
- `my_app/config/desktop.py` — Desk module/Workspace entries

### Common dev loop

- Code changes (Python/Doctype) → hot reload usually picks them up
- If in doubt, run:
  ```bash
  bench --site development.localhost migrate
  bench --site development.localhost clear-cache
  ```
- Frontend assets during development:
  ```bash
  bench watch
  ```
- Build production assets when adding new bundles:
  ```bash
  bench build
  ```

---

## 7) Version control: Git & GitHub (clean app-only repo)

You typically want only **your app** under version control, **not the entire bench**.

```bash
cd /workspace/development/frappe-bench/apps/my_app

git init
git add .
git commit -m "feat: initial commit"

# Keep your default branch aligned with the Frappe/ERPNext major version
git branch -M version-15

git remote add origin git@github.com:<your-username>/<your-repo>.git
git push -u origin version-15
```

> Commit frequently. Consider a conventional commit style (feat:, fix:, chore:, etc.). Tag versions when you cut releases.

---

## 8) Ship your app: install on any ERPNext/HRMS v15 deployment

### Option A — Classic bench (bare-metal/VM)

On the target machine (already running Frappe/ERPNext v15):

```bash
cd /path/to/frappe-bench

bench get-app --branch version-15 https://github.com/<your-username>/<your-repo>.git
bench --site <target-site> install-app my_app
bench --site <target-site> migrate
```

### Option B — frappe_docker (production) using a **custom image**

1) Create an `apps.json` that includes ERPNext, HRMS, and your app:

```json
[
  {
    "url": "https://github.com/frappe/frappe",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/frappe/hrms",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/<your-username>/<your-repo>.git",
    "branch": "version-15"
  }
]
```

2) Build a custom image using the frappe_docker build tooling that consumes `apps.json`.

3) Use that image for your `backend`, `worker`, `scheduler` services in your production `docker-compose.yml`.

4) On your production site (new or existing):

```bash
bench --site <site> install-app my_app
bench --site <site> migrate
```

> **Note:** Baking apps into the image gives deterministic, repeatable deploys. Avoid ad‑hoc `get-app` inside long‑running containers for production.

---

## 9) Data portability: fixtures (for clean installs)

If your app defines roles, workspaces, custom fields, property setters, server/client scripts, etc., export them as fixtures so fresh installs get a working baseline.

In `my_app/hooks.py`:

```python
fixtures = [
    "Custom Field",
    "Property Setter",
    "Client Script",
    "Server Script",
    "Role",
    "Workspace"
]
```

Export and commit:

```bash
bench --site development.localhost export-fixtures
cd apps/my_app
git add .
git commit -m "chore: export fixtures"
```

---

## 10) Troubleshooting

### Version mismatch (Frappe 16-dev with ERPNext 15)

**Symptoms:**
- During `get-app`/`install-app erpnext`, you see errors like:
  - *"Installed frappe-dependency 'frappe' version '16.0.0-dev' does not satisfy required version '>=15.40.4,<16.0.0'"*
  - pip resolver conflicts (`pycountry`, `requests-oauthlib`, etc.)

**Fix A — Re-init bench on v15 (cleanest):**

```bash
cd /workspace/development
rm -rf frappe-bench
bench init --frappe-branch version-15 --skip-redis-config-generation frappe-bench
cd frappe-bench
# re-run config, site, and app install steps
```

**Fix B — Downgrade Frappe in-place:**

```bash
cd /workspace/development/frappe-bench
bench stop || true

cd apps/frappe
git fetch --all
git checkout version-15
cd -

# Align all apps
bench switch-to-branch version-15 --upgrade

# Reinstall requirements
bench setup requirements --python
bench setup requirements --node

# Now install apps
bench get-app --branch version-15 erpnext --resolve-deps
bench --site development.localhost install-app erpnext
bench get-app --branch version-15 hrms --resolve-deps
bench --site development.localhost install-app hrms

bench migrate
bench start
```

If pip still nags, pin expected deps in the bench venv then rerun requirements:

```bash
./env/bin/pip install -U "pycountry==24.6.1" "requests-oauthlib==2.0.0"
bench setup requirements --python
```

### Desk won’t open

- Ensure `bench start` is running in the terminal
- Visit `http://development.localhost:8000`
- On Linux, add to `/etc/hosts`:
  ```
  127.0.0.1 development.localhost
  ```
- Check logs: `frappe-bench/logs/web.error.log`, `web.log`

### Forgot Administrator password (dev)

```bash
bench --site development.localhost set-admin-password admin
```

### New assets not loading

```bash
bench build
bench --site development.localhost clear-cache
```

### Aligning branches later

```bash
bench version
bench switch-to-branch version-15 --upgrade
```

---

## 11) Quick command cheat‑sheet

```bash
# Create v15 bench
a) bench init --frappe-branch version-15 --skip-redis-config-generation frappe-bench
b) bench set-config -g db_host mariadb
   bench set-config -g redis_cache redis://redis-cache:6379
   bench set-config -g redis_queue redis://redis-queue:6379
   bench set-config -g redis_socketio redis://redis-queue:6379

# Site
bench new-site --mariadb-user-host-login-scope=% development.localhost
bench --site development.localhost set-config developer_mode 1
bench --site development.localhost clear-cache

# Apps
bench get-app --branch version-15 erpnext --resolve-deps
bench --site development.localhost install-app erpnext
bench get-app --branch version-15 hrms --resolve-deps
bench --site development.localhost install-app hrms

# App development
bench new-app my_app
bench --site development.localhost install-app my_app
bench start
bench watch
bench build
bench --site development.localhost migrate

# Password reset (dev)
bench --site development.localhost set-admin-password admin

# Git (app only)
cd apps/my_app
git init && git add . && git commit -m "feat: initial commit"
git branch -M version-15
git remote add origin git@github.com:<you>/<repo>.git
git push -u origin version-15
```

---

## 12) Best practices recap

- **Always** pin branches: `--frappe-branch version-15` and `--branch version-15` for all apps
- Use a `.localhost` domain for dev (avoids CORS/CSRF headaches)
- Keep only **your app** in Git; don’t commit the whole bench
- Export **fixtures** to ship roles, custom fields, scripts, and workspaces reliably
- For production, prefer **custom images** (with `apps.json`) over ad‑hoc changes in running containers

---

### You’re set!

This workflow gets you from zero → dev bench with ERPNext + HRMS → your own app → GitHub → clean production installs. If you need an apps.json sample or docker build snippet tailored to your existing compose, add your current files and we’ll slot it in.

