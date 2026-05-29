# 007: ChatPlayground.ai MCP Server Integration

## Hintergrund

ChatPlayground.ai ist eine Plattform zum **Vergleich von 40+ AI-Modellen** in einer GUI. Der Hauptwert: Mehrere Modelle parallel befragen und Antworten vergleichen.

**Was es bietet:**
- 40+ AI-Modelle (Claude, GPT, Gemini, DeepSeek, Llama, Perplexity etc.)
- Parallel-Modus: Bis zu 6 Modelle gleichzeitig befragen
- Chat mit Bildern und PDFs
- Prompt Library & Chat History
- Bildgenerierung und Code-Modelle

**Was fehlt:** Keine offizielle API – nur Web-GUI.

**🔑 Account: Unlimited Lifetime** – einmalig bezahlt, unbegrenzte Queries für immer, null Margenkosten pro Anfrage.

## Warum der Lifetime-Account alles ändert

Ohne Lifetime wäre ChatPlayground nur ein bequemer Modell-Vergleich, den Karle auch über LiteLLM abbilden kann. **Mit Unlimited Lifetime** wird es zu einem hochwertigen Asset:

| Aspekt | LiteLLM (aktuell) | ChatPlayground Lifetime |
|--------|-------------------|-------------------------|
| Kosten pro Query | Credits/Token-basiert | **0€ – unlimited** |
| Modelle | 15 konfiguriert | **40+** |
| Neue Modelle | Manuell in LiteLLM adden | **Automatisch verfügbar** |
| Parallel-Vergleich | Karle fragt nacheinander | **Echt parallel** |
| Bilder/PDFs chatten | Nicht über LiteLLM | **Ja** |

**Kern-Einsicht:** Jede Query die über ChatPlayground läuft statt über LiteLLM **spart Credits**. Bei häufiger Nutzung (Cron-Jobs, Automatisierungen) summiert sich das signifikant.

## Das Problem: Keine API

Im Gegensatz zu 1min.ai und 1forall.ai bietet ChatPlayground **keine REST-API**. Alle Interaktionen laufen über die Web-GUI.

## Integrations-Optionen (neu bewertet)

### Option A: Browser-Automation (Playwright) ⚠️ Machbar aber fragil

**Ansatz:** Headless Browser steuert die GUI, loggt sich ein, sendet Prompts, parsed Antworten aus dem DOM.

```python
from playwright.async_api import async_playwright

async def chat_multi_model(prompt: str, models: list[str]) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto("https://www.chatplayground.ai")
        # Login mit gespeicherten Credentials
        await page.fill('[data-testid="prompt-input"]', prompt)
        for model in models:
            await page.click(f'[data-model="{model}"]')
        await page.click('[data-testid="send"]')
        # ... auf Antworten warten, DOM parsen ...
```

**Vorteile:**
- Nutzt den Lifetime-Account direkt – null Kosten pro Query
- Kein Reverse-Engineering nötig
- Funktioniert auch wenn interne APIs unzugänglich sind

**Nachteile:**
- **Fragil:** UI-Updates können den Wrapper brechen
- **Ressourcen:** Headless Chrome ~500MB RAM im Docker-Stack
- **Langsam:** Browser-Overhead + Rendering + Wartezeiten
- **Session-Management:** Login-Sessions expiren, müssen erneuert werden
- **Anti-Bot-Schutz:** Cloudflare/Captchas möglich

**Mitigation:**
- Playwright mit `stealth`-Plugin nutzen
- Session-Cookies persistieren (nur 1x einloggen)
- DOM-Selektor-basiert mit Fallbacks
- Healthcheck + automatischer Re-Login

### Option B: Netzwerk-Traffic Reverse-Engineering ✅ Empfohlen als erster Versuch

**Ansatz:** ChatPlayground ruft intern APIs auf. Diese Endpunkte identifizieren und direkt ansprechen.

**Schritt 1: Traffic analysieren**
1. In ChatPlayground einloggen
2. Browser DevTools → Network Tab
3. Prompt absenden
4. Alle XHR/Fetch/WebSocket-Calls identifizieren
5. Request/Response-Format dokumentieren
6. Auth-Mechanismus verstehen (Cookie? Bearer Token? Session?)

**Schritt 2: Client bauen**
```python
import httpx

class ChatPlaygroundClient:
    def __init__(self, session_token: str):
        self.client = httpx.AsyncClient(
            base_url="https://www.chatplayground.ai",
            cookies={"session": session_token}
        )

    async def chat(self, prompt: str, model: str) -> str:
        r = await self.client.post("/api/chat", json={
            "prompt": prompt,
            "model": model
        })
        return r.json()["response"]

    async def chat_parallel(self, prompt: str, models: list[str]) -> dict:
        tasks = [self.chat(prompt, m) for m in models]
        results = await asyncio.gather(*tasks)
        return dict(zip(models, results))
```

**Schritt 3: Als MCP-Server wrappen** (wie 005/006)

**Vorteile:**
- **Schnell und ressourcenschonend** – kein Headless Browser nötig
- Stabiler als DOM-Scraping
- Session-Token statt Login-Flow

**Nachteile:**
- **Inoffiziell:** Endpunkte können sich ändern
- **Token-Erneuerung:** Session-Tokens expiren periodisch
- **TOS-Risiko:** Königte gegen Nutzungsbedingungen verstoßen
- **Aufwand für Traffic-Analyse:** Einmalig ~1-2h

**Mitigation:**
- Token-Erneuerung: Fallback auf Browser-Login wenn Token abgelaufen
- Endpunkt-Versionierung: Konfigurierbare API-Pfade
- Monitoring: Healthcheck erkennt wenn Endpunkte nicht mehr reagieren

### Option C: Hybrid (B mit A als Fallback) ✅✅ Best of both worlds

**Ansatz:** Option B als primäre Methode (schnell, ressourcenschonend). Option A als Fallback wenn Token expired oder Endpunkte sich ändern.

```
ChatPlayground MCP Server
├── Primär: Direkte API-Calls (Option B)
│   └── Schnell, ressourcenschonend
├── Fallback: Browser-Automation (Option A)
│   └── Funktioniert immer, auch nach UI-Updates
└── Auto-Recovery
    └── Wenn B fehlschlägt → A startet → extrahiert neuen Token → B funktioniert wieder
```

**Vorteile:**
- Bestmögliche Stabilität durch Fallback-Kette
- Schnell im Normalbetrieb (Option B)
- Selbstheilend bei Token-Expiry
- Nur ~50MB RAM im Normalbetrieb (Headless Chrome nur bei Bedarf)

### Option D: API beim Support anfragen ✅ Langfristig ergänzen

Auch mit funktionierendem B/C-Wrapper: Offizielle API anfragen. Wenn sie eine freischalten, können wir den fragilen Teil ersetzen.

> "Hi ChatPlayground team, I'm a lifetime subscriber and power user. I've automated my workflows and would love official API access. I'd pay extra for it. My use case: programmatic multi-model comparison from my automation stack. 1min.ai and 1forall.ai already offer APIs – would love to see the same from ChatPlayground."

## Empfehlung

| Option | Aufwand | Stabilität | Nutzen | Kosten/Query |
|--------|---------|------------|--------|-------------|
| A: Browser-Only | Hoch | Niedrig | Hoch | 0€ |
| B: Reverse-Engineering | Mittel | Mittel | Hoch | 0€ |
| **C: Hybrid B+A** | **Mittel-Hoch** | **Hoch** | **Hoch** | **0€** |
| D: API anfragen | Niedrig | - | Langfristig | 0€ |

**Umsetzung:**

1. **Sofort:** Traffic-Analyse (Option B) – du loggst dich in ChatPlayground ein, öffnest DevTools, schickst einen Prompt, und teilst mir die Network-Calls
2. **Dann:** MCP-Server bauen basierend auf den gefundenen Endpunkten
3. **Falls B nicht funktioniert:** Browser-Fallback (Option A) adden
4. **Parallel:** API beim Support anfragen (Option D)

## Konkrete ChatPlayground MCP-Tools

```yaml
mcp_servers:
  chatplayground:
    url: http://chatplayground-mcp:8746/mcp
    headers:
      Authorization: Bearer ${CP_MCP_TOKEN}
    tools:
      include:
        - chat_single          # Ein Modell befragen
        - chat_parallel         # Mehrere Modelle parallel befragen
        - chat_compare          # Parallel + Vergleichs-Zusammenfassung
        - list_models           # Verfügbare Modelle auflisten
        - chat_with_image       # Bild + Prompt
        - chat_with_pdf         # PDF + Prompt
    prompts: false
    resources: false
    timeout: 120
```

**Beispiel-Nutzung:**
```
"Karle, frag ChatPlayground: Was ist der beste Bewässerungsplan für Tomaten?"
  → chat_compare mit GPT-5.5, Claude, Gemini
  → Automatischer Vergleich der 3 Antworten
  → Kosten: 0€ (Lifetime Account)
```

**Kostenersparnis-Beispiel:**
- Cron-Job der 10x/Tag 3 Modelle parallel fragt = 30 Queries/Tag
- LiteLLM-Kosten: ~30 × $0.002 = $0.06/Tag = ~$22/Jahr
- ChatPlayground: $0/Jahr (Lifetime bereits bezahlt)

## Docker-Integration

```yaml
# docker-compose.yml Ergänzung
  chatplayground-mcp:
    build: ./chatplayground-mcp-server
    container_name: chatplayground-mcp
    restart: unless-stopped
    environment:
      - CP_SESSION_TOKEN=${CP_SESSION_TOKEN}
      - CP_EMAIL=${CP_EMAIL}
      - CP_PASSWORD=${CP_PASSWORD}
    ports:
      - "8746:8746"
    networks:
      - hermes-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8746/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## Aufwandsschätzung

- Traffic-Analyse (DevTools): ~1-2h (Daniel muss mithelfen)
- MCP-Server (Python/FastMCP): ~2-3h
- Browser-Fallback (falls nötig): ~2h
- Docker-Integration: ~30min
- Testing: ~1h
- **Gesamt: ~4-6h** (ohne Browser-Fallback: ~4h)

## Nächster Schritt

**Daniel muss den Traffic analysieren:**
1. ChatPlayground.ai einloggen
2. F12 → Network Tab
3. Prompt absenden
4. Screenshots der API-Calls (URLs, Headers, Payloads) an Karle schicken

Dann kann ich den MCP-Server bauen.