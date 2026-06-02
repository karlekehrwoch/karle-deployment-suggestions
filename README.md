# Karle Deployment Suggestions

Optimierungsvorschläge für das Hermes Agent Deployment. Jede Datei ist ein eigenständiger Vorschlag, den Codex lesen und auf `karle-deployment-readonly` anwenden kann.

## Dateien

| # | Datei | Thema | Status |
|---|-------|-------|--------|
| 001 | [Browser-Persistenz](001-browser-persistenz.md) | Chrome/Playwright bei Container-Neubau erhalten | ✅ |
| 002 | [Browser-Init Automatisierung](002-browser-init-automatisierung.md) | Automatischer Chrome-Download beim Container-Start | ✅ |
| 003 | [Docker-Compose Optimierungen](003-docker-compose-optimierungen.md) | Healthcheck, Resource Limits, Litellm-Service, Log-Rotation | ✅ |
| 004 | [Config & Security Hardening](004-config-security-hardening.md) | Tokens in .env, Tirith aktivieren, Secret Redaction | ✅ |
| 005 | [1min.ai MCP Server](005-1min-mcp-server.md) | MCP-Server für 1min.ai API (30+ Tools, Multi-Account-Rotation) | 🆕 Proposal |
| 006 | [1forall.ai MCP Server](006-1forall-mcp-server.md) | MCP-Server für 1forall.ai API (16+ Tools, Async-Polling, Multi-Account-Rotation) | 🆕 Proposal |
| 009 | [MiniMax M3 – Ollama Cloud](009-minimax-m3-ollama-cloud.md) | 1M-Kontext Coding/Agentic Modell auf Ollama Cloud (Zero Data Retention) | 🆕 Proposal |

## Bezug zu den Repos

- **`karle-deployment`** → Daniels lokales Arbeits-Repo (Codex bearbeitet hier)
- **`karle-deployment-readonly`** → Mirror von karle-deployment (nur Lesen für Karle)
- **`karle-deployment-suggestions`** → Dieses Repo (Optimierungsvorschläge)
- **Dockerfile:** `/opt/hermes/Dockerfile`
- **Compose:** `/opt/hermes/docker-compose.yml`
- **Config:** `/opt/data/config.yaml` (persistentes Volume)
- **Host-Pfad:** `/srv/hermes-stack/` auf VPS 95.156.227.157

## Anwendung mit Codex

```bash
# Codex-Prompt für jede Datei:
" Lies <dateiname>.md und wende die empfohlenen Änderungen auf das
  karle-deployment Repo an. Commit mit beschreibender Nachricht. "
```

## MCP-Server Roadmap (005 + 006)

Zusammen ergeben die beiden MCP-Server **46+ native AI-Tools** in Hermes:

- **1min.ai** → Bild-Bearbeitungs-Powerhouse (BG Remove, Upscale, Face Swap, 15+ Features)
- **1forall.ai** → Audio/Sprache/Video-Powerhouse (TTS, Voice Cloning, Musik, Video)
- **Beide** → Multi-Account-Rotation für erweiterte Credit-Kapazität
- **Beide** → Credit-Tracking + automatischer Account-Switch