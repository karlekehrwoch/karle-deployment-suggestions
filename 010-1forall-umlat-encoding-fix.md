# 010 – Umlaut-Encoding-Fix: 1forall MCP OpenAI-kompatibler Endpunkt

**Status:** Investigation abgeschlossen, Fix erforderlich  
**Priorität:** Hoch (blocking für deutschsprachige Nutzung von 1forall-Modellen via LiteLLM)  
**Erstellt:** 05.06.2026  
**Betroffene Komponenten:** oneforall-ai-mcp Server, LiteLLM Proxy

---

## Problem

Wenn ein 1forall-LLM-Modell über LiteLLM als Hermes-Chatmodell verwendet wird, erscheinen deutsche Umlaute (ä, ö, ü, ß, Ä, Ö, Ü) sowie andere Nicht-ASCII-Zeichen im Telegram-Chat als `?`.

**Beispiel:**
- Erwartet: `Der Satz heißt "Die schönen Mädchen müssen über die Brücke gehen."`
- Tatsächlich: `Der Satz hei?t "Die sch?nen M?dchen m?ssen ?ber die Br?cke gehen."`

---

## Architektur & Datenfluss

```
Hermes Agent
    ↓ (OpenAI API)
LiteLLM Proxy (:4000)
    ↓ (OpenAI API, model: "openai/anthropic/claude-3.5-haiku")
oneforall-ai-mcp (:8751/v1)   ← OpenAI-kompatibler Proxy
    ↓ (1forall Async API)
api.1forall.ai               ← 1forall Backend
```

---

## Beweise & Testergebnisse

| Testfall | Pfad | Umlaute korrekt? |
|---|---|---|
| 1forall API direkt (MCP Tool `oneforall_llm_send_request`) | Hermes → 1forall API | ✅ Ja |
| 1forall-claude-3.5-haiku via LiteLLM | Hermes → LiteLLM → oneforall-mcp → 1forall API | ❌ `?` |
| 1forall-gpt-5-mini via LiteLLM | Hermes → LiteLLM → oneforall-mcp → 1forall API | ❌ `?` |
| glm5.1-agent (Ollama) via LiteLLM | Hermes → LiteLLM → Ollama | ✅ Ja |
| deepseek-v4pro (Ollama) via LiteLLM | Hermes → LiteLLM → Ollama | ✅ Ja |

### Rohbytes-Analyse

LiteLLM-Response auf Umlaut-Anfrage enthält **0 Non-ASCII-Bytes**. Die Umlaute werden bereits VOR der LiteLLM-Verarbeitung als `?` ersetzt – das Problem liegt im **oneforall-ai-mcp OpenAI-kompatiblen Endpunkt**.

### 1forall API Direkt-Test

```
Model: claude_haiku → "Häßliche Vögel fliegen über große Überlandstraßen." ✅
Model: gpt-5-mini → "Ärztin Öztürk und Ünal grüßen süße Mädchen auf der Straße schön." ✅
```

### LiteLLM Proxy-Test

```
Model: 1forall-claude-3.5-haiku → "Hier sind die von Ihnen genannten Umlaute: ? ? ? ? ? ? ?" ❌
Model: 1forall-gpt-5-mini → "Der Satz lautet: ?Die sch?nen M?dchen m?ssen ?ber die Br?cke gehen.?" ❌
```

---

## Root-Cause-Analyse

Das Problem liegt im **oneforall-ai-mcp Server** in der OpenAI-kompatiblen Chat-Completions-Implementierung. Mögliche Ursachen (nach Wahrscheinlichkeit):

### 1. Latin-1/ISO-8859-1 Encoding in HTTP-Response (am wahrscheinlichsten)
Der Server sendet den Response-Body mit `charset=ISO-8859-1` statt `charset=UTF-8`, oder das Framework kodiert den Body in Latin-1. UTF-8-Multibyte-Zeichen (Umlaute = 2 Bytes) werden dabei zu `?` weil sie außerhalb des Latin-1-Bereichs liegen – allerdings nur, wenn die Bytesequenz nicht gültig Latin-1 ist.

**Korrektur:** Response-Header `Content-Type: application/json; charset=utf-8` explizit setzen und sicherstellen, dass der Body als UTF-8 kodiert wird.

### 2. ASCII-unsafe JSON-Serializer
Einige JSON-Bibliotheken (z.B. Pythons `json.dumps(ensure_ascii=True)` als Default) ersetzen Nicht-ASCII-Zeichen durch `\uXXXX`-Escapesequenzen. Wenn diese Escape-Sequenzen anschließend durch eine ASCII-only-Schicht verarbeitet werden, können sie zu `?` werden.

**Korrektur:** `json.dumps(ensure_ascii=False)` verwenden oder den JSON-Output explizit als UTF-8 schreiben.

### 3. Stream-Chunk-Encoding-Fehler
Der 1forall-API ist asynchron: Der oneforall-ai-mcp Server muss pollend auf das Ergebnis warten und dann den Response zusammenbauen. Wenn Chunks oder der finale Response nicht als UTF-8 deklariert/kodiert werden, kann es zu Encoding-Problemen kommen.

### 4. Environment-Locale
Der Docker-Container hat möglicherweise keine UTF-8 Locale konfiguriert (`LANG=C` oder `LC_ALL=C`), was Python/Node.js dazu veranlasst, Non-ASCII-Zeichen zu verwerfen.

**Korrektur:** In Dockerfile: `ENV LANG=en_US.UTF-8` oder `ENV LC_ALL=C.UTF-8`

---

## Empfohlener Fix

### Sofortmaßnahme: Dockerfile Locale
Im `Dockerfile.oneforall-ai-mcp`:
```dockerfile
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
```

### Fix im oneforall-ai-mcp Server-Code

Die OpenAI-kompatiblen Chat-Completions-Handler müssen:

1. **HTTP Response Headers:**
   ```
   Content-Type: application/json; charset=utf-8
   ```

2. **JSON Encoding:**
   - Python: `json.dumps(response, ensure_ascii=False).encode('utf-8')`
   - Node.js: `JSON.stringify(response)` (standardmäßig UTF-8, aber prüfen)

3. **1forall API Response-Verarbeitung:**
   - Response explizit als UTF-8 dekodieren: `response.decode('utf-8')` bzw. `Buffer.from(response, 'utf-8')`

### Diagnose-Befehle

Um das Problem im oneforall-ai-mcp Container zu diagnostizieren:

```bash
# Container-Locale prüfen
docker exec oneforall-ai-mcp env | grep -E "LANG|LC_"

# Direkter API-Test
curl -s http://oneforall-ai-mcp:8751/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_COMPAT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude_haiku","messages":[{"role":"user","content":"Sag: äöü"}],"max_tokens":20}'

# Response-Headers prüfen
curl -sI http://oneforall-ai-mcp:8751/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_COMPAT_TOKEN"
```

---

## Workaround (bis Fix deployed)

1. **1forall-Modelle nicht als Default-Hermes-Modell nutzen** – aktuelle Config mit `glm5.1-agent` als Default ist korrekt.

2. **Bei Bedarf: 1forall API direkt über MCP-Tools** nutzen (ohne LiteLLM-Passthrough):
   - `oneforall_llm_send_request` → poll → Ergebnis korrekt als UTF-8
   - Nachteil: Kein nahtloser Chat-Ersatz, da asynchron

3. **Alternativ: LiteLLM Custom Provider mit direkter 1forall API-Anbindung** umgehen den oneforall-mcp OpenAI-compat Layer, falls LiteLLM einen 1forall-Provider unterstützt.

---

## Referenzen

- 1forall API: https://api.1forall.ai
- LiteLLM Proxy Config: `config/litellm/config.yaml`
- oneforall-ai-mcp Dockerfile: `docker/Dockerfile.oneforall-ai-mcp`
- oneforall-ai-mcp Docker-Compose: `docker/docker-compose.oneforall-ai-mcp.yml`
- LiteLLM Health (1forall-Modelle): `api_base: http://oneforall-ai-mcp:8751/v1`, `model: openai/anthropic/claude-3.5-haiku`