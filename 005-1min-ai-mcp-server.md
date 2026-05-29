# 005: 1min.AI MCP Server Integration

## Hintergrund

1min.AI bietet eine Subscription-basierte AI-Plattform mit umfangreicher REST-API, die deutlich über reine Chat-Endpunkte hinausgeht. Die API deckt 6 Kategorien ab:

- **Chat mit AI** – Unified Chat mit Web-Suche, Bild-Anhänge, Memory, Brand Voice
- **AI for Image** – 15+ Features: Image Generator, Upscaler, Background Remover/Replacer, Face Swap, Sketch→Image, 3D Image, Text/Object Remover, Image Variator, Extender, Image-to-Prompt, Mask Editor, Text Editor, Search & Replace
- **AI for Audio** – TTS, Transcription, Audio-Verarbeitung
- **AI for Video** – Video-Analyse und -Verarbeitung
- **AI for Code** – Code-Generierung/Analyse
- **AI for Writing** – Texterstellung, Rewriting, Paraphrasing

**API-Specs:** 180 Requests/Min, Credits-System, Base URL `https://api.1min.ai`

## Warum MCP statt LiteLLM-Provider?

Ein LiteLLM-Provider würde **nur Chat-Endpunkte** abbilden – 90% der Features gingen verloren. Als MCP-Server werden **alle 1min.AI-Features als eigenständige Tools** in Hermes registriert, analog zum bestehenden `static_webapp_deploy` MCP.

Vergleich:

| Ansatz | Chat | Bilder | Audio | Video | Code | Writing |
|--------|------|--------|-------|-------|------|---------|
| LiteLLM Provider | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| MCP Server | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## Vorschlag

### 1. Eigenen Python MCP-Server bauen

Neues Repo: `karlekehrwoch/1min-mcp-server`

- **Framework:** FastMCP (Python) – leichtgewichtig, gut dokumentiert
- **Architektur:** Wrapt die 1min.AI REST-API und exponiert alle Features als MCP-Tools
- **Naming:** `mcp_1min_{feature}` – z.B. `mcp_1min_image_generator`, `mcp_1min_background_remover`

Beispiel-Tool-Definition:

```python
from fastmcp import FastMCP

mcp = FastMCP("1min-ai")

@mcp.tool()
async def image_generator(prompt: str, model: str = "flux-pro") -> str:
    """Generate an image from a text prompt using 1min.AI"""
    # POST https://api.1min.ai/api/features
    # type: IMAGE_GENERATOR
    ...

@mcp.tool()
async def background_remover(image_url: str) -> str:
    """Remove background from an image using 1min.AI"""
    # POST https://api.1min.ai/api/features
    # type: BACKGROUND_REMOVER
    ...

@mcp.tool()
async def chat_with_ai(prompt: str, model: str = "gpt-4o", web_search: bool = False) -> str:
    """Chat with AI, optionally with web search"""
    # POST https://api.1min.ai/api/chat-with-ai
    ...
```

### 2. Docker-Service im Stack

```yaml
# docker-compose.yml Ergänzung
  1min-mcp:
    build: ./1min-mcp-server
    container_name: 1min-mcp
    restart: unless-stopped
    environment:
      - ONEMIN_API_KEY=${ONEMIN_API_KEY}
    ports:
      - "8744:8744"  # MCP HTTP endpoint
    networks:
      - hermes-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8744/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 3. Hermes config.yaml Eintrag

```yaml
mcp_servers:
  # ... bestehende Einträge ...
  1min:
    url: http://1min-mcp:8744/mcp
    headers:
      Authorization: Bearer ${ONEMIN_MCP_TOKEN}
    tools:
      include:
        - image_generator
        - image_upscaler
        - image_variator
        - background_remover
        - background_replacer
        - face_swapper
        - sketch_to_image
        - image_to_prompt
        - object_remover
        - text_remover
        - chat_with_ai
        - audio_transcription
        - video_analysis
    prompts: false
    resources: false
```

### 4. .env Ergänzung

```bash
ONEMIN_API_KEY=<api-key-von-1min.ai>
ONEMIN_MCP_TOKEN=<generiertes-token-für-mcp-auth>
```

## Konkrete Use-Cases

1. **Etsy-PDF-Produkte:** Cover-Illustrationen generieren (z.B. „Tomaten pflanzen" mit professionellen Pflanzenbildern)
2. **Background Remover:** Produktfotos freistellen für Etsy-Listings
3. **Image Upscaler:** Bilder für Druck-Qualität hochskalieren (wichtig für PDF)
4. **Sketch→Image:** Aus einfachen Skizzen druckfähige Illustrationen machen
5. **Chat mit Web-Suche:** Ergänzende Recherche mit Bildanalyse-Fähigkeit
6. **Face Swap / Image Variator:** Kreative Content-Variationen für Social Media

## API-Key erstellen

1. Einloggen auf https://app.1min.ai
2. Unter API-Keys einen neuen Key erstellen
3. Credits einsehen unter https://app.1min.ai/members (Team-Admin erforderlich)

## Aufwandsschätzung

- MCP-Server (Python/FastMCP): ~1-2h
- Docker-Integration: ~30min
- Testing & Tool-Feintuning: ~1h
- **Gesamt: ~3h**