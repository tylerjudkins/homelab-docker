# Homelab Docker — Progress Log

## Session 1 — [DATE: Replace with today's date]

### Environment
- Host: MacBook Apple Silicon (M-series)
- Docker Desktop: local dev
- GitHub user: tylerjudkins
- Repo: https://github.com/tylerjudkins/homelab-docker

### Completed
- [x] Installed and recovered access to Portainer CE
- [x] Generated SSH key on MacBook and added to GitHub
- [x] Created GitHub repo `homelab-docker`
- [x] Built folder scaffold (stacks/, configs/, .env.example, .gitignore)
- [x] Configured git remote to use SSH
- [x] Successfully pushed scaffold to GitHub main branch

### Decisions Made
- Using SSH (not HTTPS) for all GitHub auth
- Each service will be an isolated stack in Portainer
- .env files are gitignored — .env.example documents required variables
- Platform flag `linux/amd64` required for Ignition on Apple Silicon

### Planned Stack
| Service | Image | Status |
|---|---|---|
| Portainer CE | portainer/portainer-ce:latest | ✅ Running |
| Ignition 8.1 | inductiveautomation/ignition:8.1.x | 🔲 Pending |
| Mosquitto | eclipse-mosquitto:latest | 🔲 Pending |
| PostgreSQL | postgres:15 | 🔲 Pending |
| InfluxDB | influxdb:2.7 | 🔲 Pending |
| Grafana | grafana/grafana:latest | 🔲 Pending |
| Node-RED | nodered/node-red:latest | 🔲 Pending |
| Jupyter | jupyter/scipy-notebook | 🔲 Pending |

### Next Session — Pick Up Here
1. Connect Portainer to GitHub repo via SSH
2. Write and deploy first stack: PostgreSQL + InfluxDB
3. Validate containers are healthy in Portainer UI

### Key Concepts Covered
- .gitignore purpose and why custom beats presets
- SSH vs HTTPS for GitHub auth
- ARM64 vs AMD64 Docker image compatibility on M-series Macs
- MIT license
- PAT (Personal Access Token) — why it exists, why SSH is preferred
