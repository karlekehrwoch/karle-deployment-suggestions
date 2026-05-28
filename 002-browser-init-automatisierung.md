# 002: Browser-Initialisierung automatisieren

## Problem
Wenn der Container neu erstellt wird, muss `agent-browser install` manuell ausgeführt werden,
um Chrome herunterzuladen. Das dauert ~30 Sekunden und erfordert menschliches Eingreifen.

## Lösungsvorschlag

### Im entrypoint.sh ergänzen:

```bash
# Auto-install Chrome if not present (browser binaries on persistent volume)
if [ ! -f "/opt/data/home/.agent-browser/browsers/chrome-"*/chrome ]; then
    echo "Chrome not found, installing via agent-browser..."
    node /opt/hermes/node_modules/agent-browser/bin/agent-browser.js install 2>&1 || {
        echo "WARNING: Chrome installation failed. Browser tools may not work."
    }
fi
```

### Alternativ: Init-Container in docker-compose.yml

```yaml
services:
  browser-init:
    image: hermes-agent
    volumes:
      - ~/.hermes:/opt/data
    command: ["node", "/opt/hermes/node_modules/agent-browser/bin/agent-browser.js", "install"]
    restart: "no"

  gateway:
    # ... bestehende config ...
    depends_on:
      browser-init:
        condition: service_completed_successfully
```

## Empfehlung
Die entrypoint.sh-Lösung ist einfacher und benötigt keine Änderung an docker-compose.yml.
Der Init-Container ist sauberer, bedeutet aber einen zusätzlichen Container pro Start.
