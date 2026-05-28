# 001: Browser-Binärdateien persistent machen

## Problem
`agent-browser install` lädt Chrome nach `/opt/data/home/.agent-browser/browsers/` herunter.
Die Dateien **überleben** einen Container-Neustart (weil `/opt/data` ein Volume-Mount ist),
aber der Symlink `/opt/data/.agent-browser/browsers/chrome-149.0.7827.22` zeigt auf denselben Ort
und ist ebenfalls persistent.

Allerdings: `playwright` als Python-Paket liegt in `/opt/hermes/.venv/` (Container-Overlay)
und geht bei `docker compose up --force-recreate` verloren.

## Aktueller Zustand (manuell installiert)
- Chrome 149.0.7827.22 → `/opt/data/home/.agent-browser/browsers/` ✅ persistent
- Playwright Chromium → `/opt/data/.agent-browser/browsers/` ✅ persistent
- `playwright` Python-Paket → `/opt/hermes/.venv/` ❌ ephemeral

## Lösungsvorschlag

### Option A: In die Dockerfile aufnehmen (empfohlen)

```dockerfile
# Nach der bestehenden npm install + Playwright Zeile (Zeile 52-53):
RUN npx playwright install --with-deps chromium --only-shell && \
```

Die bestehende Dockerfile installiert bereits Playwright via npm (npx) und Chromium als Shell.
Das Python-Paket `playwright` fehlt aber im `uv sync` Schritt.

**Fix:** In `pyproject.toml` unter `[all]` Extras aufnehmen:

```toml
# pyproject.toml — [all] extra
playwright = { version = ">=1.40", optional = true }
```

Oder alternativ im Dockerfile nach dem `uv sync`:

```dockerfile
RUN uv pip install playwright
```

### Option B: Init-Script im entrypoint.sh

Am Ende von `entrypoint.sh`, vor dem `exec gosu`:

```bash
# Auto-install playwright Python package if missing (browser binaries persist on /opt/data)
if ! /opt/hermes/.venv/bin/python -c "import playwright" 2>/dev/null; then
    echo "Installing playwright Python package..."
    uv pip install playwright
fi
```

### Option C: Environment-Variable für Playwright-Browser-Pfad

Die Dockerfile setzt bereits `PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright`.
Die agent-browser-Binärdateien liegen aber in `/opt/data/home/.agent-browser/browsers/`.

**Fix:** Umleiten im entrypoint oder config:

```bash
export PLAYWRIGHT_BROWSERS_PATH=/opt/data/home/.agent-browser/browsers
```

Dann nutzen beide (Playwright Python + agent-browser) denselben Browser-Cache auf dem persistenten Volume.

## Empfehlung
**Option A** (Dockerfile) ist die sauberste Lösung. Option B als Fallback, falls
das Dockerfile nicht oft neu gebaut wird. Option C als Ergänzung, damit die
Pfade konsistent sind.
