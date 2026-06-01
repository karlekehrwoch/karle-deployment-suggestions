# 005: 1min.ai MCP Server (1min-mcp)

## Status: Proposal

## Zusammenfassung

Docker-basierter MCP-Server, der die 1min.ai REST API als native Hermes-Tools verfügbar macht. Unterstützt **Multi-Account-Rotation** für erweiterte Credit-Kapazität. Stellt 15+ Bild-Tools, Chat mit Web-Suche, Audio/Video-Verarbeitung und Code-Generierung als First-Class-Hermes-Tools bereit — nicht nur als Chat-Endpoint über LiteLLM.

## Problem

1min.ai Advanced Business (Lifetime) bietet 4M+450K Credits/Mo mit 15+ Bild-Tools, Chat, Audio, Video und Code. Aktuell ist 1min.ai **nicht** in Hermes integriert — alle Features müssen manuell via curl/Python aufgerufen werden. LiteLLM unterstützt nur Chat-Endpoints und lässt 90% der 1min.ai-Funktionalität ungenutzt.

### Was fehlt

- **Kein nativer Tool-Zugriff** auf 15+ Bild-Features (Background Remover, Upscaler, Face Swap, etc.)
- **Kein Chat mit Web-Suche** als Hermes-Tool
- **Kein Audio/Video/Code-Zugriff** aus Hermes heraus
- **Kein Credit-Tracking** — Daniel weiß nicht wie viel von den 4M Credits verbraucht sind
- **Kein Multi-Account-Rotation** — bei Credit-Erschöpfung ist Schluss bis zum nächsten Monat

## Lösung

### 1. Docker-Service: `1min-mcp`

Python-basierter MCP-Server (FastAPI + MCP SDK), der die 1min.ai API wrapt.

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8744
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8744"]
```

### 2. Multi-Account-Rotation

Der MCP-Server unterstützt mehrere API-Keys mit automatischer Rotation:

**Strategien:**
- `round-robin`: Gleichmäßig verteilen über alle Accounts
- `failover`: Haupt-Account bis Credit-Threshold, dann nächsten
- `least-used`: Account mit wenigstem Verbrauch zuerst

**Credit-Tracking:**
- Jeder API-Key hat einen Credit-Counter (geschätzt aus Request-Typen)
- Bei Unterschreitung des Thresholds (default: 10%) → Auto-Switch
- Monthly-Reset-Erkennung (1. des Monats → Counter zurücksetzen)

```python
# Key Manager (vereinfacht)
class MultiKeyManager:
    def __init__(self, api_keys: list[str], strategy: str = "failover"):
        self.accounts = [ApiKeyAccount(key) for key in api_keys]
        self.strategy = strategy
        self.current_index = 0

    def get_active_key(self) -> str:
        """Return best available API key based on strategy."""
        if self.strategy == "failover":
            for acc in self.accounts:
                if acc.credits_remaining_pct > self.threshold:
                    return acc.api_key
            # All depleted - use least-depleted
            return min(self.accounts, key=lambda a: a.credits_used).api_key
        elif self.strategy == "round-robin":
            acc = self.accounts[self.current_index]
            self.current_index = (self.current_index + 1) % len(self.accounts)
            return acc.api_key

    def report_usage(self, api_key: str, feature_type: str):
        """Track credit consumption per feature type."""
        costs = {
            "IMAGE_GENERATOR": 50, "BACKGROUND_REMOVER": 20,
            "IMAGE_UPSCALER": 30, "CHAT": 5, "VIDEO": 500,
            # Based on 1min.ai credit costs
        }
        acc = next(a for a in self.accounts if a.api_key == api_key)
        acc.credits_used += costs.get(feature_type, 10)
```

### 3. MCP-Tools (30+)

| Tool | 1min.ai Feature | Beschreibung |
|------|-----------------|--------------|
| `image_generate` | IMAGE_GENERATOR | Text→Bild |
| `image_upscale` | IMAGE_UPSCALER | Bild vergrößern |
| `image_vary` | IMAGE_VARIATOR | Variationen erstellen |
| `image_extend` | IMAGE_EXTENDER | Bild erweitern |
| `bg_remove` | BACKGROUND_REMOVER | Hintergrund entfernen |
| `bg_replace` | BACKGROUND_REPLACER | Hintergrund ersetzen |
| `face_swap` | FACE_SWAPPER | Gesichter tauschen |
| `text_remove` | TEXT_REMOVER | Text aus Bild entfernen |
| `object_remove` | OBJECT_REMOVER | Objekt aus Bild entfernen |
| `search_replace` | SEARCH_AND_REPLACE | Element suchen & ersetzen |
| `sketch_to_image` | SKETCH_TO_IMAGE | Skizze→Bild |
| `image_3d` | 3D_IMAGE_GENERATOR | 3D-Bild generieren |
| `image_to_prompt` | IMAGE_TO_PROMPT | Bild→Prompt (Reverse Engineering) |
| `chat` | UNIFY_CHAT_WITH_AI | Chat mit optionalem Web-Search |
| `chat_create` | /conversations | Neue Konversation starten |
| `credit_status` | — | Credit-Stand aller Accounts anzeigen |
| `account_status` | — | Alle Accounts + Rotation-Status |

### 4. Docker-Compose Integration

```yaml
# docker-compose.yml (addition)
services:
  1min-mcp:
    build: ./1min-mcp
    container_name: 1min-mcp
    ports:
      - "8744:8744"
    environment:
      - ONEMIN_API_KEYS=${ONEMIN_API_KEYS}  # comma-separated
      - ONEMIN_ROTATION_STRATEGY=failover
      - ONEMIN_CREDIT_THRESHOLD=10
      - ONEMIN_MONTHLY_CREDITS=4450000  # 4M + 450K
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8744/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

### 5. Hermes Config

```yaml
# config.yaml
mcp_servers:
  1min:
    url: "http://1min-mcp:8744/mcp"
    tools:
      include:
        - image_generate
        - image_upscale
        - bg_remove
        - bg_replace
        - face_swap
        - chat
        - credit_status
        # ... alle 30+ Tools
    prompts: false
    resources: false
    timeout: 120  # Bild/Video-Generierung kann dauern
```

### 6. .env (Secrets)

```bash
# Multi-Account: comma-separated API keys
# Account A = Daniels Haupt-Account (Advanced Business)
# Account B = Zweiter Account (falls vorhanden)
ONEMIN_API_KEYS=key-account-a,key-account-b
```

## Daniel's Use Cases

1. **Etsy PDF Illustrationen**: `image_generate` + `bg_remove` + `image_upscale` für Cover-Art
2. **Produktfotos bearbeiten**: `bg_remove` → `bg_replace` für Etsy-Listings
3. **Skizze→Bild Pipeline**: `sketch_to_image` für Print-Qualität-Illustrationen
4. **Bildvariationen**: `image_vary` für mehrere Design-Varianten
5. **Recherche**: `chat` mit `webSearch: true` als Ergänzung zu Hermes LLMs
6. **Multi-Account-Rotation**: Wenn 4M Credits/Mo nicht reichen → zweiten Account aktivieren

## Multi-Account-Kosten

| Szenario | StackSocial-Kosten | Credits/Mo | Begründung |
|---|---|---|---|
| 1 Account (aktuell) | $0 | 4M + 450K | Bereits vorhanden |
| +1 Account | +$59.97 | 8M + 900K | 2x Advanced Business |
| +2 Accounts | +$119.94 | 12M + 1.35M | 3x Advanced Business |

**Hinweis:** StackSocial-Codes sind nicht stapelbar auf einen Account, aber jeder Code kann auf einen neuen Account redeemt werden. „New users only" gilt für den 1min.ai-Account, nicht für StackSocial-Käufe.

## Abhängigkeiten

- Docker + Docker Compose (vorhanden)
- Python 3.12 + FastAPI + MCP SDK
- 1min.ai API-Keys (vorhanden, erweiterbar)
- Netzwerk: 1min-mcp Container muss api.1min.ai erreichen

## Implementation Steps

1. `1min-mcp/` Verzeichnis im Deployment-Repo erstellen
2. MCP-Server mit FastAPI + MCP Python SDK implementieren
3. Multi-Key-Manager mit Credit-Tracking implementieren
4. Alle 15+ Bild-Tools als MCP-Tools wrappen
5. Chat-Tool mit Web-Search-Support implementieren
6. Docker-Compose Service hinzufügen
7. Hermes config.yaml erweitern
8. Health-Check + Credit-Monitoring Endpoint
9. Tests mit echten API-Calls
10. Dokumentation im Skill `1min-ai` aktualisieren

## Risiken & Mitigation

| Risiko | Wahrscheinlichkeit | Mitigation |
|--------|-------------------|------------|
| 1min.ai API ändert sich | Mittel | Feature-Type-Mapping konfigurierbar halten |
| Multi-Account von 1min.ai blockiert | Niedrig | ToS verbietet Account-Sharing zwischen Personen, nicht mehrere eigene Accounts |
| Credit-Schätzung ungenau | Mittel | Konservative Schätzung + Dashboard für manuellen Check |
| Rate-Limit bei Multi-Key | Niedrig | 180 req/min per Key → Multi-Key erhöht Gesamt-Rate |