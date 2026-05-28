# 004: Config & Security Hardening

## Aktuelle Config (Auszug)

```yaml
model:
  provider: custom:litellm
  default: glm5.1-agent
custom_providers:
  - name: litellm
    base_url: http://litellm:4000/v1
    key_env: HERMES_LITELLM_API_KEY
    api_mode: chat_completions
    models:
      glm5.1-agent: {}
      # ... 14 models

mcp_servers:
  github:
    url: "https://api.githubcopilot.com/mcp/"
    headers:
      Authorization: "Bearer github...j8d6"  # ← Token im Klartext!
  static_webapp_deploy:
    headers:
      Authorization: "Bearer 3a7cde7b..."   # ← Token im Klartext!

# security: commented out (default behavior)
```

## Probleme
1. **API-Tokens im Klartext** in `config.yaml` statt in `.env`
2. **Security-Tirith** ist auskommentiert (obwohl im Image verfügbar)
3. **Secret Redaction** ist nicht explizit aktiviert
4. **API Server** ist nicht konfiguriert (default: localhost-only, aber ungesichert)

## Lösungsvorschläge

### 1. Tokens nach .env auslagern

```yaml
# config.yaml — nur Referenzen
mcp_servers:
  github:
    url: "https://api.githubcopilot.com/mcp/"
    headers:
      Authorization: "Bearer ${COPILOT_GITHUB_TOKEN}"
  static_webapp_deploy:
    url: "http://static-webapp-deploy-api:8743/mcp"
    headers:
      Authorization: "Bearer ${WEBAPP_DEPLOY_TOKEN}"
```

```bash
# .env
COPILOT_GITHUB_TOKEN=github_p_...j8d6
WEBAPP_DEPLOY_TOKEN=3a7cde7b9001cb6a24d2e4172bf8a6f4d8af7734bca4bd1ab6ef4c6833dec620
HERMES_LITELLM_API_KEY=sk-...
```

### 2. Security explizit aktivieren

```yaml
security:
  redact_secrets: true
  tirith_enabled: true
  tirith_path: "tirith"
  tirith_timeout: 5
  tirith_fail_open: true
```

### 3. API Server (falls benötigt)

```yaml
# In config.yaml:
# API_SERVER_KEY muss gesetzt sein für Remote-Zugriff
# Aktuell: localhost-only → sicher

# In .env:
API_SERVER_KEY=${HERMES_API_KEY}
# API_SERVER_HOST=0.0.0.0  # NUR mit SSH-Tunnel, niemals direkt!
```

### 4. .gitignore für .env sicherstellen

```gitignore
# .gitignore
.env
.env.*
*.key
*.pem
```

### 5. File Permissions

```bash
chmod 600 /srv/hermes-stack/data/.env
chmod 600 /srv/hermes-stack/data/config.yaml
```
