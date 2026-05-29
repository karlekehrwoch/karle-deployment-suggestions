# 006: 1forall.ai MCP Server Integration

## Hintergrund

1forall.ai bietet eine Subscription-basierte AI-Plattform mit REST-API (Swagger/OpenAPI 3.0). Im Vergleich zu 1min.ai ist der Fokus stärker auf **Audio, Speech und Video**, mit einem **asynchronen Job-Modell** (start→check-status→retrieve).

**API Base URL:** `https://api.1forall.ai/`
**Auth:** `Authorization: Api-Key <key>` (Header)
**Swagger UI:** https://api.1forall.ai/v1/external/swagger-ui/

## API-Kategorien & Endpunkte

### 🔊 Audio
| Endpunkt | Methode | Beschreibung |
|----------|---------|--------------|
| `/v1/external/audio/convert-to-audio/` | POST | Text→Sound, Text→Music, Music→Music (asynchron) |
| `/v1/external/audio/check-status/{code_ref}/` | GET | Job-Status prüfen |
| `/v1/external/audio/conversions/{id}/` | GET | Fertiges Ergebnis abrufen |
| `/v1/external/audio/models/` | GET | Verfügbare Audio-Modelle (u.a. `elevenlabs-sound`) |

**Audio-Modi:** Sound-Generierung, Musik-Generierung, Music-to-Music
**Parameter:** `title`, `text`, `model`, `seconds` (0.5-Schritte)
**Credits:** z.B. 14 Credits/Sekunde (elevenlabs-sound)

### 🖼️ Image
| Endpunkt | Methode | Beschreibung |
|----------|---------|--------------|
| `/v1/external/image/text-to-image/` | POST | Text→Bild Generierung |
| `/v1/external/image/image-to-image/` | POST | Bild→Bild Transformation |
| `/v1/external/image/check-status/{code_ref}/` | GET | Job-Status prüfen |
| `/v1/external/image/conversions/{id}/` | GET | Fertiges Ergebnis abrufen |
| `/v1/external/image/models/` | GET | Verfügbare Bild-Modelle |

### 🤖 LLM
| Endpunkt | Methode | Beschreibung |
|----------|---------|--------------|
| `/v1/external/llm/send-request/` | POST | LLM-Anfrage senden |
| `/v1/external/llm/check-status/{code_ref}/` | GET | Job-Status prüfen |
| `/v1/external/llm/conversions/{id}/` | GET | Fertiges Ergebnis abrufen |
| `/v1/external/llm/models/` | GET | Verfügbare LLM-Modelle |

### 🗣️ Speech (TTS mit Voice Cloning!)
| Endpunkt | Methode | Beschreibung |
|----------|---------|--------------|
| `/v1/external/speech/text-to-speech/` | POST | Text→Sprache generieren |
| `/v1/external/speech/cloned-voices/` | GET | Eigene geclonte Stimmen auflisten |
| `/v1/external/speech/voices/` | GET | Verfügbare Stimmen auflisten |
| `/v1/external/speech/languages/` | GET | Unterstützte Sprachen |
| `/v1/external/speech/regions/` | GET | Verfügbare Regionen |
| `/v1/external/speech/check-status/{code_ref}/` | GET | Job-Status prüfen |
| `/v1/external/speech/conversions/{id}/` | GET | Fertiges Ergebnis abrufen |

### 🎬 Video
| Endpunkt | Methode | Beschreibung |
|----------|---------|--------------|
| `/v1/external/video/text-to-video/` | POST | Text→Video Generierung |
| `/v1/external/video/image-to-video/` | POST | Bild→Video Transformation |
| `/v1/external/video/check-status/{code_ref}/` | GET | Job-Status prüfen |
| `/v1/external/video/conversions/{id}/` | GET | Fertiges Ergebnis abrufen |
| `/v1/external/video/models/` | GET | Verfügbare Video-Modelle |

## Besonderheiten im Vergleich zu 1min.ai

| Aspekt | 1min.ai | 1forall.ai |
|--------|---------|------------|
| API-Modell | Synchron (streaming/non-streaming) | **Asynchron** (start→poll→retrieve) |
| Auth | `API-KEY` Header | `Api-Key` Header |
| Speech/TTS | ❌ Nicht verfügbar | ✅ **Mit Voice Cloning** |
| Audio-Gen | Begrenzt | ✅ Sound + Musik + Music-to-Music |
| Video | Analyse | ✅ **Text→Video + Image→Video** |
| Bild-Tools | 15+ Features (Remover, Upscaler, etc.) | Text→Image + Image→Image (simpler) |
| LLM | Eigener Chat-Endpunkt | Eigener LLM-Endpunkt |

## Vorschlag

### 1. Eigenen Python MCP-Server bauen

Neues Repo: `karlekehrwoch/1forall-mcp-server`

**Framework:** FastMCP (Python)
**Architektur:** Wrapt die 1forall.ai REST-API, handhabt das asynchrone Job-Modell automatisch

Wichtigster Design-Aspekt: Der MCP-Server muss das **Polling-Modell** abstrahieren – der User ruft ein Tool auf und bekommt das fertige Ergebnis zurück, das interne Polling passiert im MCP-Server.

```python
from fastmcp import FastMCP
import httpx
import asyncio

mcp = FastMCP("1forall-ai")

API_BASE = "https://api.1forall.ai/v1/external"

async def _poll_until_done(code_ref: str, category: str, api_key: str) -> dict:
    """Poll check-status endpoint until job completes."""
    while True:
        r = await httpx.AsyncClient().get(
            f"{API_BASE}/{category}/check-status/{code_ref}/",
            headers={"Authorization": f"Api-Key {api_key}"}
        )
        data = r.json()
        if data["status"] in ("completed", "failed"):
            return data
        await asyncio.sleep(3)  # poll every 3s

@mcp.tool()
async def text_to_speech(text: str, voice: str, language: str = "de") -> str:
    """Convert text to speech using 1forall.ai TTS with voice cloning support"""
    # POST /v1/external/speech/text-to-speech/
    # → get code_ref → poll → return url_file
    ...

@mcp.tool()
async def text_to_image(prompt: str, model: str = "flux-pro") -> str:
    """Generate an image from text using 1forall.ai"""
    ...

@mcp.tool()
async def text_to_video(prompt: str, model: str = "kling-video") -> str:
    """Generate a video from text using 1forall.ai"""
    ...

@mcp.tool()
async def convert_to_audio(text: str, model: str = "elevenlabs-sound", seconds: float = 1.0) -> str:
    """Generate sound or music from text"""
    ...

@mcp.tool()
async def image_to_video(image_url: str, model: str = "kling-video") -> str:
    """Convert an image to video"""
    ...

@mcp.tool()
async def llm_send_request(prompt: str, model: str = "gpt-4o") -> str:
    """Send a request to an LLM via 1forall.ai"""
    ...

@mcp.tool()
async def list_voices() -> str:
    """List available TTS voices"""
    ...

@mcp.tool()
async def list_cloned_voices() -> str:
    """List your cloned voices"""
    ...
```

### 2. Docker-Service im Stack

```yaml
# docker-compose.yml Ergänzung
  1forall-mcp:
    build: ./1forall-mcp-server
    container_name: 1forall-mcp
    restart: unless-stopped
    environment:
      - ONEFORALL_API_KEY=${ONEFORALL_API_KEY}
    ports:
      - "8745:8745"  # MCP HTTP endpoint
    networks:
      - hermes-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8745/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 3. Hermes config.yaml Eintrag

```yaml
mcp_servers:
  # ... bestehende Einträge ...
  1forall:
    url: http://1forall-mcp:8745/mcp
    headers:
      Authorization: Bearer ${ONEFORALL_MCP_TOKEN}
    tools:
      include:
        - text_to_speech
        - text_to_image
        - text_to_video
        - image_to_video
        - convert_to_audio
        - image_to_image
        - llm_send_request
        - list_voices
        - list_cloned_voices
    prompts: false
    resources: false
    timeout: 300  # Video-Gen kann dauern
```

### 4. .env Ergänzung

```bash
ONEFORALL_API_KEY=<api-key-von-1forall.ai>
ONEFORALL_MCP_TOKEN=<generiertes-token-für-mcp-auth>
```

## Konkrete Use-Cases

1. **TTS mit Voice Cloning:** Daniels Piper-Stimme klonen und hochwertigere TTS via 1forall.ai (Ergänzung zu lokalem Piper)
2. **Musik-Generierung:** Hintergrundmusik für Videos/Podcasts generieren
3. **Text→Video:** Kurze Erklärvideos für Etsy-Produkte oder Social Media
4. **Bild→Video:** Statische Produktfotos animieren für Listings
5. **Sound-Effekte:** Sound-Effekte für Tonie-Audio oder andere Projekte
6. **LLM als Fallback:** Zusätzlicher LLM-Zugang über anderes Credit-System

## Kombination mit 1min.ai (Vorschlag 005)

Die beiden APIs ergänzen sich ideal:
- **1min.ai** → Bild-Bearbeitung (Background Remover, Upscaler, Face Swap, Sketch→Image)
- **1forall.ai** → Audio/Speech/Video (TTS, Voice Cloning, Musik, Video-Gen)

Beide als MCP-Server → Karle hat 30+ AI-Tools nativ verfügbar, ohne API-Keys manuell verwalten zu müssen.

## Aufwandsschätzung

- MCP-Server (Python/FastMCP + Polling-Logik): ~2h
- Docker-Integration: ~30min
- Testing & Tool-Feintuning: ~1h
- **Gesamt: ~3.5h** (etwas mehr wegen asynchronem Polling-Modell)