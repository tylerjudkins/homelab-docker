# Homelab GitOps Workflow: Git -> GitHub -> Portainer (Dev + Prod)

Guide for `homelab-docker`. Covers creating the files, pushing to GitHub, deploying prod and dev stacks in Portainer, and the daily change workflow.

**Scope:** the **data-layer** stack (PostgreSQL + InfluxDB) gets a permanent dev stack because it's cheap and teaches the whole workflow. **Ignition** gets a short-lived test stack instead (see Section 10).

> **Adapt, don't paste blindly.** The compose file below is a template. Your repo already has a reviewed `stacks/data-layer/docker-compose.yml`. Compare the two and merge in only the changes you want (mainly the `${VAR}` ports and network name).

---

## 1. Mental model

| Concept | What it is | Where it lives |
|---|---|---|
| **Compose file** | Desired state: what runs | Git (committed) |
| **Env vars / secrets** | Values that make a deployment unique | **Portainer only** (never git) |
| **`main` branch** | Production state | GitHub `origin/main` |
| **`dev` branch** | Staging state (long-lived, never deleted) | GitHub `origin/dev` |
| **Feature branch** | Short-lived sandbox for one change | Local + GitHub, deleted after merge |
| **Prod stack** | Deployment tracking `main` | Portainer |
| **Dev stack** | Deployment tracking `dev` | Portainer |
| **Named volumes** | Persistent data (databases) | Docker host, survives redeploys |

```
feature branch --PR--> dev --PR--> main
                        |           |
                   dev stack    prod stack
                   (Portainer)  (Portainer)
```

**Key safety property:** pushing a branch deploys nothing. Only a merge into `dev` or `main` changes a running stack.

**Key vocabulary:** `origin` is the **nickname for your GitHub repo's URL**. `main` is a **branch**. `origin/main` is your laptop's cached snapshot of GitHub's `main`, updated by `git fetch`.

---

## 2. Pre-flight checks

Run from inside your local clone:

```bash
git remote -v          # origin should show git@github.com:tylerjudkins/homelab-docker.git
git status             # should report a clean tree, on branch main
ssh -T git@github.com  # should greet you by GitHub username
```

**Why:** confirms Git knows where GitHub is and that your SSH key works before you build anything on top.

---

## 3. Create the git files

Target layout:

```
homelab-docker/
├── .gitignore
├── .env.example
├── PROGRESS.md
├── docs/
│   └── GITOPS-WORKFLOW.md      <- this file
├── configs/
└── stacks/
    └── data-layer/
        ├── docker-compose.yml
        └── .env.example
```

### 3.1 `.gitignore` (verify these lines exist)

```gitignore
# Secrets: never commit real values
.env
.env.*
!.env.example
*.pem
*.key

# Local data and OS junk
data/
.DS_Store
```

**Why the order matters:** `.env.*` ignores every env variant, then `!.env.example` re-includes only the placeholder file. Reverse the order and the exception stops working.

### 3.2 `stacks/data-layer/docker-compose.yml`

```yaml
# Do NOT add a top-level "name:" key.
# Portainer sets the Compose project name from the STACK NAME.
# That prefix is what keeps dev and prod volumes separate.

services:
  postgres:
    image: postgres:15
    restart: unless-stopped
    # No container_name: a fixed name can only exist once per host
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      # ":?" aborts the deploy with a message if the variable is unset
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}
      POSTGRES_DB: ${POSTGRES_DB:-homelab}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"    # host port varies per stack
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app_net

  influxdb:
    image: influxdb:2.7
    restart: unless-stopped
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUX_USER}
      DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUX_PASSWORD:?INFLUX_PASSWORD is required}
      DOCKER_INFLUXDB_INIT_ORG: ${INFLUX_ORG:-homelab}
      DOCKER_INFLUXDB_INIT_BUCKET: ${INFLUX_BUCKET:-plant}
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUX_TOKEN:?INFLUX_TOKEN is required}
    ports:
      - "${INFLUX_PORT:-8086}:8086"
    volumes:
      - influxdata:/var/lib/influxdb2
      - influxconfig:/etc/influxdb2
    networks:
      - app_net

networks:
  app_net:
    name: ${NETWORK_NAME:-homelab_net}   # varies per stack
    external: true                        # must exist before deploy (Section 5.1)

volumes:
  pgdata:
  influxdata:
  influxconfig:
```

### 3.3 `stacks/data-layer/.env.example` (placeholders only)

```bash
# Copy these names into Portainer's stack environment variables.
# NEVER put real values in this file.
POSTGRES_USER=changeme
POSTGRES_PASSWORD=changeme
POSTGRES_DB=homelab
POSTGRES_PORT=5432

INFLUX_USER=changeme
INFLUX_PASSWORD=changeme-min-8-chars
INFLUX_TOKEN=changeme
INFLUX_ORG=homelab
INFLUX_BUCKET=plant
INFLUX_PORT=8086

NETWORK_NAME=homelab_net
```

### Why this design

- **Compose file = structure. Portainer env vars = values.** Git keeps history forever, which is exactly wrong for secrets. Variables are the seam between the two.
- **Every environment difference is a variable** (ports, network name, passwords). The same file then works in dev and prod with no edits.
- **Gotcha:** `POSTGRES_PASSWORD` and the `DOCKER_INFLUXDB_INIT_*` values are read **only on the first start with an empty volume**. Changing them in Portainer later does **not** change the password inside an existing database. Changing it means an in-database command, or deleting the volume (which destroys the data).

---

## 4. Commit and push to GitHub

### 4.1 Verify nothing secret is staged

```bash
git status
git check-ignore -v .env          # prints a rule if .env is ignored (good)
git add .gitignore stacks/ docs/
git diff --cached                 # READ this. Look for real passwords or tokens.
```

### 4.2 Commit and push `main`

```bash
git commit -m "Add data-layer stack with variable-driven config"
git push origin main
```

`git push origin main` means: send my local `main` branch to the remote nicknamed `origin`.

### 4.3 Create and push the long-lived `dev` branch

```bash
git checkout -b dev               # create dev from current main and switch to it
git push -u origin dev            # publish it; -u links local dev to origin/dev
```

**Why `-u`:** it sets the upstream, so later `git pull` and `git push` on `dev` need no extra arguments.

**Do not delete `dev` after merging.** A Portainer stack tracking a deleted branch breaks.

### 4.4 Turn on GitHub protections (in the repo's Settings)

| Setting | Effect |
|---|---|
| **Branch protection / ruleset on `main`**: require a pull request | Nothing (human or AI tool) can push straight to production |
| **Secret scanning + push protection** | Blocks pushes containing recognized key patterns |

Availability of some options depends on repo visibility and GitHub plan. Check what your repo offers.

**If a secret is ever committed:** rotate it immediately. `git rm` does not remove it from history, especially in a public repo.

---

## 5. Deploy the stacks in Portainer

### 5.1 Create one network per environment (Terminal, once)

```bash
docker network create homelab_net
docker network create homelab_net_dev
```

**Why two:** if dev and prod share a network, both `postgres` services answer to the DNS name `postgres`, and a dev container could talk to prod's database. Separate networks enforce isolation.

### 5.2 Prod stack

In Portainer: **Stacks -> Add stack -> Repository**.

| Field | Value |
|---|---|
| **Name** | `data-layer` |
| **Repository URL** | your GitHub repo URL |
| **Repository reference** | `refs/heads/main` |
| **Compose path** | `stacks/data-layer/docker-compose.yml` |
| **Environment variables** | Enter each name from `.env.example` with **real values** (unique password and token) |

### 5.3 Dev stack

Same form, different values:

| Field | Prod | Dev |
|---|---|---|
| **Name** | `data-layer` | `data-layer-dev` |
| **Repository reference** | `refs/heads/main` | `refs/heads/dev` |
| `POSTGRES_PORT` | `5432` | `5433` |
| `INFLUX_PORT` | `8086` | `8087` |
| `NETWORK_NAME` | `homelab_net` | `homelab_net_dev` |
| Passwords / token | Unique | **Different** from prod |

**Why the stack name matters:** it becomes the Compose project name, which Docker prefixes onto volume names. Prod gets `data-layer_pgdata` and dev gets `data-layer-dev_pgdata`. They are separate volumes, so dev cannot touch prod data.

### 5.4 Repository authentication (verify in your version)

Portainer needs read access to the repo if it's private. Confirm in your Portainer version's form which methods are offered (SSH key vs. username + personal access token). A **fine-grained, read-only, single-repo token** is the least-privilege choice if tokens are supported. That credential is a secret: it lives in Portainer only.

### 5.5 Auto-updates

Enable **GitOps updates** on each stack. Portainer then checks the tracked branch on a **polling interval** (simplest for a home network) and redeploys when it changes. A webhook is faster but requires Portainer to be reachable from GitHub. Confirm the options in your version's UI.

### 5.6 Verify isolation

```bash
docker ps                          # postgres/influx containers for both stacks
docker volume ls                   # expect data-layer_pgdata AND data-layer-dev_pgdata
docker network ls                  # expect homelab_net AND homelab_net_dev
```

Then connect to Postgres on `localhost:5432` (prod) and `localhost:5433` (dev) and confirm they are separate databases.

---

## 6. Daily change workflow

```bash
# 0. Sync first (every session, every machine)
git fetch origin
git checkout dev
git pull origin dev

# 1. Branch for one change
git checkout -b add-grafana-stack

# 2. Edit files (you or Claude Code), then:
git add stacks/grafana/
git commit -m "Add Grafana stack"
git push -u origin add-grafana-stack     # inert: nothing deploys
```

Then on GitHub:

1. **Open a PR:** `add-grafana-stack` into **`dev`**. Read the diff and merge.
2. **Dev stack redeploys.** Test it (logs, UI, connections).
3. **Open a PR:** `dev` into **`main`**. Merge using **"Create a merge commit"**, not squash.
4. **Prod stack redeploys.**

```bash
# 5. Cleanup and resync
git checkout dev
git fetch origin
git merge origin/main                     # bring dev level with main
git push origin dev
git branch -d add-grafana-stack           # optional cleanup
```

**Why not squash `dev` -> `main`:** squashing rewrites the change as a new commit, so `dev` and `main` disagree about history and you get repeat conflicts. A merge commit keeps them consistent.

**Merging does not delete branches.** Deleting a merged feature branch is optional cleanup. `dev` and `main` are never deleted.

---

## 7. Working from multiple computers

Each computer has its own clone. GitHub (`origin`) is the shared copy. Nothing syncs automatically.

**Rule: pull before you start, push before you leave.**

```bash
git pull origin dev        # start of session
git push origin dev        # end of session
```

If a push is rejected (**non-fast-forward**), the other machine pushed first. Run `git pull origin <branch>`, resolve any conflict, and push again.

To pick up a feature branch that exists only on GitHub:

```bash
git fetch origin
git checkout add-grafana-stack     # creates a local branch tracking origin
```

---

## 8. Rollback

| Situation | Action |
|---|---|
| Bad change on a feature branch | Delete the branch. Nothing was deployed. |
| Bad merge into `dev` or `main` | `git revert -m 1 <merge-commit-sha>` then push. Portainer redeploys the previous config. |
| Bad local commit not yet pushed | `git reset --soft HEAD~1` (keeps your edits, undoes the commit) |

`-m 1` is required when reverting a **merge commit**: it tells Git to keep the first parent (the branch you merged into) as the mainline.

**Limit:** revert fixes **config**. It does not repair data damaged inside a volume. Back up databases separately.

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Deploy fails: `POSTGRES_PASSWORD is required` | Variable missing in the stack's env vars | Add it in Portainer, redeploy |
| `network homelab_net_dev ... could not be found` | External network not created | Run the `docker network create` commands (5.1) |
| `port is already allocated` | Two stacks bind the same host port | Give each stack a different `*_PORT` value |
| `container name already in use` | A fixed `container_name:` in the compose file | Remove it |
| Dev and prod volumes collide | Top-level `name:` in compose, or identical stack names | Remove `name:`, use distinct stack names |
| Changed DB password in Portainer, old one still works | Init variables apply only to an empty volume | Change it inside the database, or recreate the volume (data loss) |
| Stack won't update after a merge | Wrong branch reference, deleted branch, or polling disabled | Check **Repository reference** and GitOps updates |
| Portainer can't clone the repo | Auth misconfigured | Recheck 5.4 |
| `git push` rejected | Remote has commits you lack | `git pull origin <branch>`, then push |
| Pushed a secret | It is now in history | **Rotate it.** Do not rely on deleting the commit. |

---

## 10. Ignition: test stack, not a permanent dev stack

Ignition 8.1 runs under emulation on Apple Silicon. A second permanent gateway roughly doubles the RAM and CPU cost. Instead:

1. Branch and edit the Ignition compose file (keep `platform: linux/amd64` on the service).
2. Deploy a **temporary** stack (`ignition-test`) from that branch, with its own ports and volumes.
3. Verify, then **delete the stack** in Portainer.
4. Merge the branch.

Deleting a stack removes its containers. Delete its volumes too only if you're sure you don't need the data.

---

## 11. Optional: Claude Code guardrails

If you use Claude Code in your local clone, a `CLAUDE.md` in the repo root encodes your rules once. Starter:

```markdown
# homelab-docker rules
- Never edit or commit `.env` files. Only `.env.example` (placeholders).
- All secrets and environment differences use `${VAR}` in compose files.
- No top-level `name:` and no `container_name:` in compose files.
- Ignition services must declare `platform: linux/amd64`.
- Work on a feature branch. Never commit directly to `main` or `dev`.
- Show the diff before any commit or push.
```

Review every diff before you push or merge. Merging to `dev` or `main` changes running stacks.

---

## 12. Real production services (Immich, Jellyfin)

Sections 1–11 describe the **lab pattern**: disposable data, a permanent dev stack, no cost to breaking it. Immich and Jellyfin are different — real, irreplaceable data with nightly backups (photos to a local restic repo and Backblaze B2; see `~/Documents/immich-homelab-runbook.md` on the host for the full backup/recovery architecture). A few deliberate departures from the lab pattern:

- **No dev stack.** Spinning up a second Postgres + photo library for a single-user service adds cost without teaching anything new. Changes go through a feature-branch PR into `dev` for review, then a PR into `main` — same review discipline, but only one live deployment. There is no `data-layer-dev`-style parallel copy.
- **`container_name:` is intentionally kept** in `stacks/immich/docker-compose.yml`, unlike the "no container_name" rule in Section 3.2 and the CLAUDE.md starter above. That rule exists to let dev and prod run side by side on one host; since there's no parallel Immich deployment, fixed names are fine and match the containers' existing identity.
- **Host is rootful Podman**, not Docker — `docker network create` / `docker ps` become `sudo podman network create` / `sudo podman ps`. Portainer talks to the rootful socket.
- **Bind mounts, not named volumes, for the stateful paths** (`/mnt/immich/photos`, `/mnt/immich/postgres`). Unlike the lab stack's Docker-managed named volumes, these paths must stay byte-identical across any stack recreation or the running containers lose their data. `immich_model-cache` is the one named volume, already `external: true`.
- **Jellyfin is not a Portainer stack.** It runs as a systemd quadlet (`/etc/containers/systemd/jellyfin.container`) with an `ExecStartPre` guard that refuses to start if its media mount is missing — plain compose has no equivalent, so it stays on the quadlet rather than moving into git-triggered GitOps. The quadlet file itself may still get tracked in git for documentation/backup, separate from deployment.
- **Password rotation on a live database is not a `git revert`.** Section 8's rollback table assumes config-only changes. A Postgres password lives inside the database itself — reverting the compose/env change in git does not undo a password change; that always needs an explicit `ALTER USER` (or equivalent) against the running database.

---

## Quick reference

| Command | Purpose |
|---|---|
| `git fetch origin` | Update `origin/*` snapshots (no file changes) |
| `git pull origin <branch>` | Fetch + merge into your current branch |
| `git checkout -b <name>` | Create and switch to a branch |
| `git push -u origin <name>` | Publish a branch and set its upstream |
| `git status` | Ahead/behind/dirty state (relative to last fetch) |
| `git diff --cached` | Review staged changes before committing |
| `git revert -m 1 <sha>` | Undo a merge commit |
| `git branch -d <name>` | Delete a merged local branch |
