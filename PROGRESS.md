I can't write directly to your GitHub repo — I don't have push access. You'll need to update the file yourself.

Here's the updated `PROGRESS.md` content — copy/paste and commit it:

````markdown
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
- Wrote `stacks/data-layer/docker-compose.yml` (PostgreSQL 15 + InfluxDB 2.7)
- Updated `.env.example` with required variables for both services
- Established `homelab_net` as external shared network strategy

### Stack Status

| Service | Image | Status |
|---|---|---|
| Portainer CE | portainer/portainer-ce:latest | ✅ Running |
| Ignition 8.1 | inductiveautomation/ignition:8.1.x | 🔲 Pending |
| Mosquitto | eclipse-mosquitto:latest | 🔲 Pending |
| PostgreSQL | postgres:15 | 🔲 Pending — stack written, not deployed |
| InfluxDB | influxdb:2.7 | 🔲 Pending — stack written, not deployed |
| Grafana | grafana/grafana:latest | 🔲 Pending |
| Node-RED | nodered/node-red:latest | 🔲 Pending |
| Jupyter | jupyter/scipy-notebook | 🔲 Pending |

### Next Session — Pick Up Here
1. Answer open question: *Why are named volumes declared at the bottom of the compose file, outside service blocks?*
2. Create `homelab_net` external network: `docker network create homelab_net`
3. Connect Portainer to GitHub repo via SSH key
4. Deploy `stacks/data-layer/docker-compose.yml` via Portainer
5. Validate both containers healthy:
   - Postgres: `docker exec -it homelab_postgres psql -U admin -d homelab`
   - InfluxDB: browse `http://localhost:8086`
6. Confirm Ignition patch version to pin before writing that stack

### Open Questions
- Why are named volumes declared as top-level objects instead of inline per-service?
- What Ignition 8.1.x patch version to pin? (recommendation: pin specific, e.g. `8.1.43`)
````
