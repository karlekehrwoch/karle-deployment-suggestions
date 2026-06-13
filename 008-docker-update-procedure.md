# 008: Docker Update Procedure — Avoid `/update` in Container

**Status:** Draft  
**Area:** Deployment, Documentation  
**Priority:** Medium  
**Author:** Karle (Agent)

---

## Problem

`hermes update` and the gateway `/update` command **do not work correctly inside the Docker container**:

1. `/opt/hermes` has no `.git` directory — the source is baked into the image via `COPY . .` in the Dockerfile
2. The fallback pip-only update updates the Python wheel in `.venv/` but **not** the editable install at `/opt/hermes/`, leading to inconsistent source vs. package versions
3. Users who run `/update` from Telegram get a false sense of success while the source code remains stale

## Suggestion

### 1. Add `UPDATE.md` to the deployment repo

Document the correct update path prominently:

```bash
# On the host (VPS 95.156.227.157):
cd /srv/hermes-stack
git pull origin dev
docker compose build
docker compose up -d
```

### 2. Add a startup warning in `entrypoint.sh`

Detect that we're in a container without `.git` and print a warning:

```bash
if [ ! -d /opt/hermes/.git ]; then
  echo "⚠️  Running from Docker image (no .git). Use 'docker compose build && docker compose up -d' on the host to update — do NOT use /update."
fi
```

### 3. Block or warn on `/update` in Docker

**Option A:** Patch `hermes update` to detect the Docker/no-git scenario and refuse with a helpful message.

**Option B:** Add a gateway hook that warns the user when `/update` is invoked in a container:

> ⚠️ You're running inside a Docker container. `/update` only updates the Python wheel, not the source code. Rebuild on the host: `cd /srv/hermes-stack && git pull && docker compose build && docker compose up -d`

### 4. Document persistent vs ephemeral paths

| Path | Persistent? | Notes |
|------|-------------|-------|
| `/opt/data/` | ✅ | Config, sessions, skills, memory, `.env` |
| `/vault/` | ✅ | Obsidian vault (read-only) |
| `/opt/data/home/.agent-browser/` | ✅ | Chrome binaries |
| `/opt/hermes/.venv/` | ❌ | Manually pip-installed packages — add to Dockerfile |
| `/opt/hermes/.playwright/` | ❌ | Build-time browsers |
| Everything under `/opt/hermes/` | ❌ | Container overlay — lost on recreate |

### 5. Move manual pip installs into Dockerfile

Any package installed via `pip install` inside the running container (e.g., `faster-whisper`, `piper-tts`) will be lost on the next `docker compose up -d`. Add them to the Dockerfile's `uv sync` step or a `requirements.txt`:

```dockerfile
# In Dockerfile, after uv sync:
RUN /opt/hermes/.venv/bin/pip install faster-whisper piper-tts
```

## References

- Skill: `hermes-agent` → `references/deployment-optimization.md`
- Current Hermes version: `0.14.0`