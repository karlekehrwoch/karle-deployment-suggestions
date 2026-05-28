# 003: docker-compose.yml Optimierungen

## Aktuelle Konfiguration
```yaml
services:
  gateway:
    build: .
    image: hermes-agent
    container_name: hermes
    restart: unless-stopped
    network_mode: host
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - HERMES_UID=${HERMES_UID:-10000}
      - HERMES_GID=${HERMES_GID:-10000}
    command: ["gateway", "run"]

  dashboard:
    image: hermes-agent
    container_name: hermes-dashboard
    restart: unless-stopped
    network_mode: host
    depends_on:
      - gateway
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - HERMES_UID=${HERMES_UID:-10000}
      - HERMES_GID=${HERMES_GID:-10000}
    command: ["dashboard", "--host", "127.0.0.1", "--no-open"]
```

## Vorschläge

### 1. Volume-Benennung statt Tilde
Die Tilde (`~/.hermes`) wird zur Host-UID des ausführenden Users expandiert.
Explizite Pfad-Angabe ist robuster:

```yaml
volumes:
  - /srv/hermes-stack/data:/opt/data     # Expliziter Host-Pfad
  - /srv/hermes-stack/vault:/vault:ro     # Vault read-only für Agent
```

### 2. Healthcheck ergänzen
```yaml
gateway:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9119/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 30s
```

### 3. Resource Limits
```yaml
gateway:
  deploy:
    resources:
      limits:
        memory: 4G
        cpus: "2.0"
      reservations:
        memory: 1G
        cpus: "0.5"
```

### 4. Litellm-Proxy als eigener Service
Wenn litellm nicht bereits separat läuft, sollte er als eigener Service ergänzt werden:

```yaml
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    container_name: litellm
    restart: unless-stopped
    ports:
      - "4000:4000"
    volumes:
      - /srv/hermes-stack/litellm/config.yaml:/app/config.yaml
    environment:
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
    command: ["--config", "/app/config.yaml", "--port", "4000"]
```

### 5. Log-Driver begrenzen
```yaml
gateway:
  logging:
    driver: json-file
    options:
      max-size: "50m"
      max-file: "3"
```

### 6. Dashboard als Same-Container (Alternative)
Der Dashboard-Service nutzt dasselbe Image und Volume. Er könnte auch als
Separat-Process im Gateway-Container laufen (spart Ressourcen):

```yaml
# Alternative: Dashboard im selben Container starten
gateway:
  command: ["sh", "-c", "gateway run & dashboard --host 127.0.0.1 --no-open & wait -n"]
```

Oder per Supervisor/Tini. Die Separate-Container-Variante ist aber sauberer für Updates.
