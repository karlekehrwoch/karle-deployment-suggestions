# Karle Deployment Suggestions

Optimierungsvorschläge für das Hermes Agent Deployment. Jede Datei ist ein eigenständiger Vorschlag, den Codex lesen und auf `karle-deployment-readonly` anwenden kann.

## Dateien

| # | Datei | Thema |
|---|-------|-------|
| 001 | [Browser-Persistenz](001-browser-persistenz.md) | Chrome/Playwright bei Container-Neubau erhalten |
| 002 | [Browser-Init Automatisierung](002-browser-init-automatisierung.md) | Automatischer Chrome-Download beim Container-Start |
| 003 | [Docker-Compose Optimierungen](003-docker-compose-optimierungen.md) | Healthcheck, Resource Limits, Litellm-Service, Log-Rotation |
| 004 | [Config & Security Hardening](004-config-security-hardening.md) | Tokens in .env, Tirith aktivieren, Secret Redaction |

## Bezug zum Haupt-Repo

- **Quelle:** `karle-deployment-readonly` (dev-Branch)
- **Dockerfile:** `/opt/hermes/Dockerfile`
- **Compose:** `/opt/hermes/docker-compose.yml`
- **Config:** `/opt/data/config.yaml` (persistentes Volume)
- **Host-Pfad:** `/srv/hermes-stack/` auf VPS 95.156.227.157

## Anwendung mit Codex

```bash
# Codex-Prompt für jede Datei:
" Lies <dateiname>.md und wende die empfohlenen Änderungen auf das
  karle-deployment-readonly Repo (dev-Branch) an. Commit mit beschreibender
  Nachricht. "
```