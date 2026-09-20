# Homelab Docker — Progress Log

## Session 1

### Environment
- Host: MacBook Apple Silicon (M-series)
- Docker Desktop: local dev
- GitHub user: tylerjudkins
- Repo: https://github.com/tylerjudkins/homelab-docker

### Completed
- Installed and recovered access to Portainer CE
- Generated SSH key on MacBook and added to GitHub
- Created GitHub repo `homelab-docker`
- Built folder scaffold (stacks/, configs/, .env.example, .gitignore)
- Configured git remote to use SSH
- Successfully pushed scaffold to GitHub main branch

### Decisions Made
- Using SSH (not HTTPS) for all GitHub auth
- Each service will be an isolated stack in Portainer
- .env files are gitignored — .env.example documents required variables
- Platform flag `linux/amd64` required for Ignition on Apple Silicon

---

## Session 2 — 2026-04-03

### Concepts Learned
- `docker-compose.yml` is a service wrapper, not app config — the container doesn't know Docker exists
- **Volumes** are independent objects from containers — containers are disposable, volumes hold state
- `docker-compose down` preserves volumes; `docker-compose down -v` destroys them
- **Environment variables** (`${VAR_NAME}`) make compose files portable across environments
- **Named volumes** declared at the bottom of compose files are shared objects — not owned by any single service

### Completed
- Reviewed next session plan from Session 1
- Wrote `stacks/databases/docker-compose.yml` (PostgreSQL 15 + InfluxDB 2.7)
- Updated `.env.example` with required variables for both services
- Established `homelab_net` as external shared network strategy

---

## Session 3 — 2026-09-20

### Completed
- Added `stacks/immich/` (real production stack: Immich server, ML, Redis, Postgres w/ pgvector) and `docs/GITOPS-WORKFLOW.md`, the full dev→main GitOps guide. Merged via PR #1 into `main`.
- Established the `dev`/`main` promotion workflow: feature branch → PR into `dev` → verify → PR into `main` for prod. `dev` is long-lived and never deleted; feature branches are deleted after merge.
- Fast-forwarded `dev` to match `main` (they had diverged since the Immich PR went straight to `main`) and deleted the merged `add-immich-stack` branch.
- Filled in default configs for all remaining scaffolded stacks, following the lab pattern from `docs/GITOPS-WORKFLOW.md` (external `homelab_net` network var, `${VAR}`-driven ports, named volumes, no fixed `container_name`, since these stacks are meant to run as paired dev+prod deployments):
  - `stacks/databases/` — PostgreSQL 15 + InfluxDB 2.7 (matches the GITOPS-WORKFLOW template)
  - `stacks/mqtt/` — Eclipse Mosquitto, using `configs/mosquitto/mosquitto.conf` (also written; `allow_anonymous true` for now, internal network only)
  - `stacks/grafana/` — Grafana
  - `stacks/node-red/` — Node-RED
  - `stacks/jupyter/` — Jupyter SciPy notebook
  - `stacks/ignition/` — Ignition 8.1.43 pinned, `platform: linux/amd64`; per GITOPS-WORKFLOW Section 10 this should run as a **temporary test stack**, not a permanent dev stack

### Stack Status

| Service | Image | Status |
|---|---|---|
| Portainer CE | portainer/portainer-ce:latest | ✅ Running |
| Immich | immich-server / immich-machine-learning / redis / postgres | ✅ Merged to main — verify deployed/running in Portainer |
| PostgreSQL + InfluxDB | postgres:15 / influxdb:2.7 | 🔲 Written, not deployed |
| Mosquitto | eclipse-mosquitto:latest | 🔲 Written, not deployed |
| Grafana | grafana/grafana:latest | 🔲 Written, not deployed |
| Node-RED | nodered/node-red:latest | 🔲 Written, not deployed |
| Jupyter | jupyter/scipy-notebook | 🔲 Written, not deployed |
| Ignition 8.1.43 | inductiveautomation/ignition:8.1.43 | 🔲 Written, not deployed — temporary test stack only |

### Next Session — Pick Up Here
1. Review and merge PR for `add-default-stacks` (branched off `dev`) — read every compose file before merging, this was a batch scaffold pass, not individually deployed/tested yet.
2. Create the two external networks if not already done: `docker network create homelab_net` and `docker network create homelab_net_dev` (or `sudo podman network create ...` if targeting the rootful Podman host).
3. Deploy stacks one at a time via Portainer (dev stack first, verify, then promote), starting with `databases` since Grafana/Node-RED will want it running.
4. Decide real values for each stack's env vars in Portainer (never commit real values — `.env.example` in each stack folder lists what's needed).
5. Revisit `mosquitto.conf`'s `allow_anonymous true` once the broker needs to be reachable beyond the internal network.

### Open Questions
- Which stacks besides Immich are on the Podman host vs. the Mac Docker Desktop host? GITOPS-WORKFLOW.md Section 12 notes Immich uses rootful Podman — confirm the same or different host applies to the rest.
- Do Grafana/Node-RED need explicit `depends_on`/network wiring to reach `databases`, or is shared `homelab_net` membership sufficient?
