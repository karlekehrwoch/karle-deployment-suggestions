# 006: 1forall.ai MCP Server (1forall-mcp)

## Status: Proposal

## Zusammenfassung

Docker-basierter MCP-Server, der die 1forall.ai Async-REST-API als native Hermes-Tools verfügbar macht. Unterstützt **Multi-Account-Rotation** für erweiterte Credit-Kapazität. Stellt TTS mit Voice Cloning, Sound/Musik-Generierung, Bild- und Video-Generierung als First-Class-Hermes-Tools bereit. Der Server abstrahiert das Async-Polling-Modell vollständig — Tools blockieren bis Ergebnisse bereit sind.

## Problem

1forall.ai Advance (Lifetime) bietet 24K Credits/Mo mit TTS, Voice Cloning, Musik, Bild- und Video-Generierung. Aktuell ist 1forall.ai **nicht** in Hermes integriert — alle Features müssen manuell via curl/Python mit manuellem Polling aufgerufen werden.

### Was fehlt

- **Kein nativer Tool-Zugriff** auf TTS + Voice Cloning (Killer-Feature)
- **Keine Musik/Sound-Generierung** als Tool
- **Keine Bild-/Video-Generierung** aus Hermes heraus
- **Kein PDF→Audiobook** Pipeline
- **Kein Excel→Speech/Batch-TTS** als Tool
- **Kein Credit-Tracking** — 24K Credits/Mo sind nicht sichtbar
- **Kein Multi-Account-Rotation** — bei Erschöpfung bis nächsten Monat warten
- **Async-Polling** muss manuell gemacht werden (code_ref → check-status → result)

## Lösung

### 1. Docker-Service: `1forall-mcp`

Python-basierter MCP-Server (FastAPI + MCP SDK), der die 1forall.ai API wrapt und das Async-Polling **intern** abstrahiert.

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8745
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8745"]
```

### 2. Async-Polling Abstraktion (Kern-Feature)

1forall.ai nutzt ein asynchrones Job-Modell: POST → `code_ref` → poll → result. Der MCP-Server verbirgt dieses Polling komplett — der Tool-Aufruf blockiert bis das Ergebnis bereit ist.

```python
class AsyncJobHandler:
    """Abstracts 1forall.ai's async polling into synchronous MCP tool calls."""

    MAX_POLL_ATTEMPTS = 120  # 120 * 5s = 10 min max wait
    POLL_INTERVAL = 5  # seconds

    async def submit_and_wait(self, endpoint: str, payload: dict, api_key: str) -> dict:
        """Submit job, poll until complete, return result."""
        # 1. Submit
        code_ref = await self._submit(endpoint, payload, api_key)

        # 2. Poll
        for attempt in range(self.MAX_POLL_ATTEMPTS):
            await asyncio.sleep(self.POLL_INTERVAL)
            status = await self._check_status(code_ref, api_key)
            if status["status"] == "completed":
                # 3. Get result
                return await self._get_result(status["id"], api_key)
            elif status["status"] == "failed":
                raise McpError(f"Job failed: {status.get('error', 'unknown')}")

        raise McpError(f"Job timeout after {self.MAX_POLL_ATTEMPTS * self.POLL_INTERVAL}s")
```

### 3. Multi-Account-Rotation

Identisches System wie 1min-mcp, aber angepasst an 1forall.ai's Credit-Modell:

```python
class OneForAllKeyManager(MultiKeyManager):
    """1forall.ai specific key manager with credit cost tables."""

    # Credit costs per action type (from 1forall.ai pricing)
    CREDIT_COSTS = {
        "tts_standard": 1,        # per character (approx)
        "tts_neural": 3,           # per character (approx)
        "voice_cloning": 50,      # per clone operation
        "text_to_image": 10,      # per image
        "text_to_video": 100,     # per video
        "sound_generation": 14,    # per second (elevenlabs-sound)
        "music_generation": 14,   # per second
        "llm_chat": 1,            # per request (approx)
    }

    def __init__(self, api_keys: list[str], monthly_credits: int = 24000,
                 strategy: str = "failover"):
        self.accounts = [
            OneForAllAccount(key, monthly_credits) for key in api_keys
        ]
        self.strategy = strategy
```

**Strategien:** `failover`, `round-robin`, `least-used` (identisch zu 005)

**1forall.ai Besonderheit:**
- Credits laufen **nie ab** und können bis zu 6 Monate akkumuliert werden
- Advance-Plan gibt **10% Rabatt** auf Zusatz-Credits
- Nachkauf-Option als Fallback vor Multi-Accounting

### 4. MCP-Tools

| Tool | 1forall.ai Endpoint | Beschreibung |
|------|---------------------|-------------|
| `text_to_speech` | /speech/text-to-speech/ | Text→Audio mit Stimmauswahl |
| `list_voices` | /speech/voices/ | Verfügbare Stimmen anzeigen |
| `list_languages` | /speech/languages/ | Unterstützte Sprachen |
| `list_cloned_voices` | /speech/cloned-voices/ | Daniels geklonte Stimmen |
| `clone_voice` | /speech/... (TBD) | Neue Stimme klonen |
| `text_to_image` | /image/text-to-image/ | Text→Bild |
| `image_to_image` | /image/image-to-image/ | Bild→Bild-Transformation |
| `text_to_video` | /video/text-to-video/ | Text→Video |
| `image_to_video` | /video/image-to-video/ | Bild→Video-Animation |
| `generate_sound` | /audio/convert-to-audio/ | Sound-Effekte generieren |
| `generate_music` | /audio/convert-to-audio/ | Musik generieren |
| `list_audio_models` | /audio/models/ | Audio-Modelle anzeigen |
| `list_image_models` | /image/models/ | Bild-Modelle anzeigen |
| `list_video_models` | /video/models/ | Video-Modelle anzeigen |
| `llm_request` | /llm/send-request/ | LLM-Anfrage |
| `credit_status` | — | Credit-Stand aller Accounts |
| `account_status` | — | Alle Accounts + Rotation-Status |

### 5. Docker-Compose Integration

```yaml
# docker-compose.yml (addition)
services:
  1forall-mcp:
    build: ./1forall-mcp
    container_name: 1forall-mcp
    ports:
      - "8745:8745"
    environment:
      - ONEFORALL_API_KEYS=${ONEFORALL_API_KEYS}  # comma-separated
      - ONEFORALL_ROTATION_STRATEGY=failover
      - ONEFORALL_CREDIT_THRESHOLD=10
      - ONEFORALL_MONTHLY_CREDITS=24000  # Advance plan
      - ONEFORALL_POLL_INTERVAL=5
      - ONEFORALL_MAX_POLL_ATTEMPTS=120  # 10 min max for video
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8745/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

### 6. Hermes Config

```yaml
# config.yaml
mcp_servers:
  1forall:
    url: "http://1forall-mcp:8745/mcp"
    tools:
      include:
        - text_to_speech
        - list_voices
        - list_cloned_voices
        - text_to_image
        - text_to_video
        - image_to_video
        - generate_sound
        - generate_music
        - credit_status
        # ... alle 16+ Tools
    prompts: false
    resources: false
    timeout: 300  # Video-Generierung kann mehrere Minuten dauern
```

### 7. .env (Secrets)

```bash
# Multi-Account: comma-separated API keys
# Account A = Daniels Haupt-Account (Advance, 24K Credits/Mo)
# Account B = Zweiter Account (falls vorhanden)
ONEFORALL_API_KEYS=key-account-a,key-account-b
```

## Daniel's Use Cases

1. **TTS mit Voice Cloning**: Daniels Stimme klonen für hochwertige TTS (Ergänzung zu lokalem Piper)
2. **Musik/Sound-Generierung**: Hintergrundmusik für Videos/Podcasts, Sound-Effekte für Tonie-Audio
3. **Text→Video**: Kurze Erklärvideos für Etsy-Produkte oder Social Media
4. **Bild→Video**: Statische Produktfotos animieren für Etsy-Listings
5. **PDF→Audiobook**: KDP-Bücher als Hörbuch-Version
6. **Excel→Speech**: Batch-TTS für große Mengen (z.B. Vokabeltrainer)
7. **Multi-Account-Rotation**: Wenn 24K Credits/Mo nicht reichen → nächsten Account nutzen

## Multi-Account-Kosten

| Szenario | StackSocial-Kosten | Credits/Mo | Begründung |
|---|---|---|---|
| 1 Account Advance (aktuell) | $0 | 24K | Bereits vorhanden |
| +1 Account Advance | +$89.99 | 48K | 2x Advance |
| +1 Account Pro (Alternative) | +$49.99 | 36K | 1x Advance + 1x Pro |

**1forall.ai Besonderheit:** „Available to BOTH new and existing users" — StackSocial-Kauf ist unkompliziert. Credits verfallen nie und akkumulieren bis zu 6 Monate.

**Alternative: Pay-as-you-go Nachkauf** (kein Multi-Account-Risiko):
- Advance-Plan gibt 10% Rabatt auf Zusatz-Credits
- Credits verfallen nie
- Kein ToS-Risiko bei Multi-Accounting
- Empfehlung: Erst Pay-as-you-go testen, dann Multi-Account wenn teurer

## Synergie mit 1min.ai (Proposal 005)

Die beiden MCP-Server ergänzen sich perfekt:

| | 1min.ai (005) | 1forall.ai (006) |
|---|---|---|
| **Stärke** | Bild-Bearbeitung (15+ Features) | Audio/Sprache/Video |
| **Bild-Gen** | ✅ 15+ Features (BG Remove, Upscale, Face Swap...) | ✅ Text→Image, Image→Image |
| **TTS** | ⚠️ Basic | ✅ Killer-Feature + Voice Cloning |
| **Video** | ⚠️ Basic | ✅ Text→Video, Image→Video |
| **Chat** | ✅ Mit Web-Suche | ⚠️ LLM nur als Fallback |
| **Musik/Sound** | ❌ | ✅ Einzigartig |
| **Batch** | ❌ | ✅ Excel→Speech, PDF→Speech |

**Zusammen als MCP:** 46+ native AI-Tools in Hermes ohne manuelle API-Verwaltung.

## Abhängigkeiten

- Docker + Docker Compose (vorhanden)
- Python 3.12 + FastAPI + MCP SDK
- 1forall.ai API-Keys (vorhanden, erweiterbar)
- Netzwerk: 1forall-mcp Container muss api.1forall.ai erreichen
- Proposal 005 (1min-mcp) für gemeinsame Multi-Key-Library

## Implementation Steps

1. `1forall-mcp/` Verzeichnis im Deployment-Repo erstellen
2. Async-Polling-Handler implementieren (Kern-Feature!)
3. Multi-Key-Manager (geteilt mit 005, oder separate Implementierung)
4. TTS-Tools als erste Priority (Killer-Feature)
5. Voice Cloning Tool implementieren
6. Bild-/Video-/Audio-Tools wrappen
7. Docker-Compose Service hinzufügen
8. Hermes config.yaml erweitern
9. Health-Check + Credit-Monitoring Endpoint
10. Tests mit echten API-Calls (TTS, Image, Video)
11. Skill `1forall-ai` aktualisieren

## Risiken & Mitigation

| Risiko | Wahrscheinlichkeit | Mitigation |
|--------|-------------------|------------|
| Async-Job timeout | Mittel | Konfigurierbares Timeout (default 10 Min), Retry-Logic |
| 1forall.ai API ändert sich | Mittel | Swagger-Endpoint für Auto-Discovery nutzen |
| Multi-Account blockiert | Niedrig | 1forall.ai ToS weniger restriktiv; Pay-as-you-go als Fallback |
| Video-Generierung dauert sehr lange | Hoch | Progress-Reporting via MCP, Timeout=300s |
| Voice Cloning Qualität | Mittel | Test mit Daniels Piper-Stimme, Vergleich |