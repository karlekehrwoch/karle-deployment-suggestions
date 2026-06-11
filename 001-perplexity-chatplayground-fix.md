# Fix: Perplexity Sonar Pro Timeout in ChatPlayground MCP Bridge

## Problem

When querying **Perplexity Sonar Pro** via the ChatPlayground MCP bridge (`mcp_chatplayground_ai_chatplayground_ask`), the request frequently times out after 120 seconds. Observed failure rate: ~50% on factual/web-search queries.

### Root Cause Analysis

After investigating the ChatPlayground GUI, the likely cause is that **Perplexity's response UI renders differently** from other models:

1. **Other models** (GPT-5.5, Claude, etc.): After sending a prompt, the response appears as a single streaming message block. The bridge waits for the `[data-message-author-role="assistant"]` or `[data-testid*="assistant"]` selector to appear and populate.

2. **Perplexity Sonar Pro**: After sending a prompt, Perplexity's response UI includes:
   - A **sources/citations section** that renders first (links, references)
   - A **search status indicator** (searching, analyzing)
   - The **actual answer text** that appears after the sources
   - Possibly a **different DOM structure** for the response container

This means the bridge's response selector may:
- Match the sources section before the answer text is complete
- Not find the expected response container at all (different CSS classes/attributes)
- Return early with an incomplete response (just citations, no answer)
- Wait indefinitely for a selector that never appears, hitting the 120s timeout

### Evidence

- Perplexity queries that are **pure text generation** (creative writing, no web search needed) tend to work
- Perplexity queries that **trigger web search** almost always time out
- The timeout happens even for simple factual questions where the model should respond quickly
- The GUI shows Perplexity has 3 variants now: Sonar, Sonar Pro, Sonar Pro Reasoning — all may share this issue

## Proposed Fix

### 1. Add Perplexity-specific response detection

The bridge should handle Perplexity's unique response structure:

```python
# Current approach (works for GPT/Claude/Gemini)
response_selector = "[data-message-author-role='assistant'], [data-testid*='assistant']"

# Proposed: Also check for Perplexity-specific selectors
perplexity_selectors = [
    "[data-message-author-role='assistant']",
    "[data-testid*='assistant']",
    "[class*='perplexity']",
    "[class*='sonar']",
    "[class*='citation'] + [class*='answer']",
    "[class*='source'] ~ div",
]
```

### 2. Wait for answer text, not just response container

Instead of returning as soon as a response container appears, wait for meaningful content:

```python
def wait_for_response(browser, timeout=120):
    start = time.time()
    while time.time() - start < timeout:
        response = browser.query_selector(response_selector)
        if response and response.inner_text().strip():
            text = response.inner_text().strip()
            if len(text) > 50:  # Minimum viable response
                return text
        time.sleep(2)
    raise TimeoutError("Response timeout")
```

### 3. Increase timeout for Perplexity specifically

Perplexity's web search adds latency. The bridge should allow more time:

```python
MODEL_TIMEOUTS = {
    "default": 120,
    "perplexity-sonar-pro": 180,
    "perplexity-sonar-pro-reasoning": 240,
}
```

### 4. Detect Perplexity's "searching" state and wait

```python
def is_perplexity_searching(browser):
    """Check if Perplexity is still searching/web-browsing."""
    indicators = browser.query_selector_all("[class*='loading'], [class*='searching'], [class*='spinner']")
    return len(indicators) > 0

def wait_for_perplexity_completion(browser, timeout=180):
    time.sleep(3)
    while is_perplexity_searching(browser) and time.time() - start < timeout:
        time.sleep(2)
    return wait_for_response(browser, remaining_timeout)
```

### 5. Handle Perplexity's different response DOM structure

The bridge should inspect the actual Perplexity response page and add selectors based on what it finds. Possible Perplexity response patterns:

```
Standard model:
  div.chat-message (assistant) -> p -> "Answer text here"

Perplexity:
  div.search-results -> [citations/sources]
  div.chat-message (assistant) -> [sources section] + [answer section]
  OR
  div.prose -> [answer text with inline citations]
```

## Investigation Steps

To implement this fix, someone needs to:

1. **Send a Perplexity query manually** through the ChatPlayground browser
2. **Inspect the DOM** after the response completes to identify:
   - The response container selector
   - The answer text container (separate from citations)
   - The "searching/loading" indicator selector
3. **Take a screenshot** of the Perplexity response UI for reference
4. **Update the bridge code** with Perplexity-specific selectors
5. **Test** with both search-triggered and non-search queries

## Priority

**Medium** — Perplexity is the only model with this issue, and it's one of 8 available models. The workaround is to use web scraping for factual queries instead. However, Perplexity is the only search-backed model in ChatPlayground, so fixing it would add significant value.

## Alternative Approaches

1. **Skip ChatPlayground for Perplexity** — Use the Perplexity API directly (requires paid API key)
2. **Use browser scraping as fallback** — When Perplexity times out via MCP, fall back to browser-based extraction
3. **Add a "Perplexity mode" to the bridge** — Detect model type and switch response extraction logic accordingly

## New Models Detected

ChatPlayground now shows additional models beyond what's configured in the MCP bridge:

| Model | Status in Bridge |
|-------|-----------------|
| Perplexity Sonar | Not configured |
| Perplexity Sonar Pro | Configured (but broken) |
| Perplexity Sonar Pro Reasoning | Not configured |
| ChatGPT-5.2 | Not configured |
| ChatGPT-5.1 | Not configured |
| ChatGPT-5 | Not configured |
| ChatGPT-4o | Not configured |
| ChatGPT O4-mini | Not configured |
| ChatGPT O3-mini | Not configured |
| Kimi K2.5 | Not configured |
| DeepSeek V4 Pro | Not configured |
| DeepSeek R1 | Not configured |
| DeepSeek V3.2 | Not configured |
| Llama 4 Scout | Not configured |
| Llama 4 Maverick | Not configured |
| Grok 4.1 | Not configured |
| Mistral Large 3 | Not configured |
| Ministral 14B | Not configured |

These could be added to the bridge config for additional free model access.
