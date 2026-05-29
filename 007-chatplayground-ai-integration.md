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

**Preis:** $17.50/Monat (12-Monats-Plan), unbegrenzte Queries

## Das Problem: Keine API

Im Gegensatz zu 1min.ai und 1forall.ai bietet ChatPlayground **keine REST-API**. Alle Interaktionen laufen über die Web-GUI. Das macht eine Integration deutlich komplexer und fragiler.

## Integrations-Optionen

### Option A: Browser-Automation (Playwright/Puppeteer) ❌ Nicht empfohlen

**Ansatz:** Headless Browser steuert die GUI, loggt sich ein, sendet Prompts, parsed Antworten aus dem DOM.

```python
# Konzeptionell
from playwright.async_api import async_playwright

async def chat_multi_model(prompt: str, models: list[str]) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto("https://www.chatplayground.ai")
        await page.fill('[data-testid="prompt-input"]', prompt)
        for model in models:
            await page.click(f'[data-model="{model}"]')
        await page.click('[data-testid="send"]')
        # ... auf Antworten warten, DOM parsen ...
```

**Nachteile:**
- **Extrem fragil:** Jedes UI-Update bricht den Wrapper
- **Captchas/Cloudflare-Schutz:** Wahrscheinlich vorhanden
- **Session-Management:** Login-Sessions expiren regelmäßig
- **Keine saubere Datenstruktur:** Antworten aus DOM scrapen ist unzuverlässig
- **Langsam:** Browser-Overhead + Rendering + Wartezeiten
- **Browser-Ressourcen:** Headless Chrome braucht ~500MB RAM extra im Docker-Stack

### Option B: Netzwerk-Traffic-Reverse-Engineering ⚠️ Fragil aber machbar

**Ansatz:** ChatPlayground ruft intern APIs auf (für die AI-Provider). Diese internen Endpunkte abfangen und direkt ansprechen.

1. Browser DevTools → Network Tab beim Chatten
2. API-Calls identifizieren (wahrscheinlich WebSocket oder REST)
3. Request/Response-Format dokumentieren
4. Eigenen Client schreiben, der diese internen Endpunkte direkt anspricht

**Vorteile:**
- Schneller als Browser-Automation
- Stabiler als DOM-Scraping
- Weniger Ressourcen

**Nachteile:**
- **Inoffiziell:** Endpunkte können sich jederzeit ändern
- **Auth-Token:** Session-Tokens expiren, müssen erneuert werden
- **Rate-Limiting:** Unbekannt, ob Server-seitig geschützt
- **TOS-Risiko:** Könnte gegen die Nutzungsbedingungen verstoßen

### Option C: ChatPlayground nicht als MCP integrieren ✅ Empfohlen

**Ansatz:** ChatPlayground.ai hat einen anderen Use-Case als 1min.ai/1forall.ai. Der Wert ist der **Vergleich**, nicht die einzelne Abfrage. Das passt besser als gelegentliches manuelles Tool denn als automatisierter MCP-Server.

**Begründung:**
1. **Du hast schon 15+ Modelle via LiteLLM** – die meisten ChatPlayground-Modelle sind bei dir bereits konfiguriert (GPT, Claude, Gemini, DeepSeek, Mistral, Qwen)
2. **Parallel-Vergleich kann Karle selbst:** Ich kann mehrere LiteLLM-Modelle parallel abfragen und Ergebnisse vergleichen – kein MCP nötig
3. **Keine API = dauerhafte Wartungsfalle** – jeder UI-Change bricht den Wrapper
4. **Kosten-Nutzen:** $17.50/Monat für etwas, das du schon hast (bis auf den Bequeme-Vergleichs-Modus)

### Option D: API beim Support anfragen ✅ Langfristig beste Option

**Ansatz:** ChatPlayground kontaktieren und API-Zugang anfragen.

**Argumente für API-Zugang:**
- "Ich nutze ChatPlayground produktiv und möchte es in meinen Workflow automatisieren"
- "Ich bin bereit, für API-Zugang extra zu zahlen"
- "Viele Power-User brauchen API-Zugang"
- Referenz: 1min.ai und 1forall.ai bieten APIs – Wettbewerb drückt

**Vorteile:**
- Wenn sie eine API freischalten → saubere Integration wie 005/006
- Offiziell unterstützt, kein Reverse-Engineering-Risiko
- Feedback an das Produkt hilft beiden Seiten

## Empfehlung

| Option | Aufwand | Stabilität | Nutzen |
|--------|---------|------------|--------|
| A: Browser-Automation | Hoch | Sehr niedrig | Mittel |
| B: Reverse-Engineering | Mittel | Niedrig | Mittel |
| **C: Nicht integrieren** | **Keiner** | **-** | **Hoch** |
| D: API anfragen | Niedrig | - | Langfristig hoch |

**Schritt 1:** Option C – ChatPlayground manuell nutzen für Modell-Vergleiche, Karle macht Parallel-Abfragen via LiteLLM wenn du vergleichst

**Schritt 2:** Option D – Support kontaktieren:
> "Hi ChatPlayground team, I'm a subscriber and love the platform. I automate my workflows with AI tools and would love API access. I'd pay extra for it. Any plans for an API? My use case: programmatic multi-model comparison from my automation stack."

**Schritt 3:** Wenn API kommt → wie 005/006 als MCP-Server integrieren

## Was Karle schon jetzt kann (ohne MCP)

Daniel kann mich bitten, mehrere Modelle parallel abzufragen:

```
"Karle, frag GPT-5.5, Claude und Gemini zum Thema X und vergleiche die Antworten"
```

Ich frage dann nacheinander über LiteLLM ab und erstelle einen Vergleich. Das deckt 80% des ChatPlayground-Use-Cases ab, ohne dass wir eine fragile GUI-Wrapper bauen müssen.