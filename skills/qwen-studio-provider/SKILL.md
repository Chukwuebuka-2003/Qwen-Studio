---
name: qwen-studio-provider
description: Use when setting up, debugging, or restoring Qwen Studio (chat.qwen.ai) as a model provider in Hermes via the qwen_browser_proxy.py web-scraping bridge. Covers the CDP attach bug, the empty-content payload bug (system/assistant roles), the Aliyun WAF captcha, and the noVNC tunnel for manual login.
---

# Qwen Studio as a Hermes Model Provider (qwen_browser_proxy)

Qwen Studio (chat.qwen.ai) has no public API key for the web product. This setup
bridges it into Hermes as an OpenAI-compatible provider by driving a **real logged-in
Chromium** over CDP and forwarding chat-completion requests to Qwen's internal
`/api/v2/chat/completions` endpoint.

## Architecture

```
Hermes gateway
   -> http://127.0.0.1:8000/v1  (OpenAI-compatible, custom:qwen-studio provider)
      -> qwen_browser_proxy.py (FastAPI + Playwright)
         -> attaches to logged-in Chrome over CDP (QWEN_CDP_URL=http://127.0.0.1:9222)
            -> POST /api/v2/chats/new  (create chat)
            -> POST /api/v2/chat/completions?chat_id=...  (stream completion)
```

Key paths:
- Proxy: `/opt/data/qwen3-reverse/qwen_browser_proxy.py`
- Daemon (starts Xtigervnc + Chrome + proxy): `/opt/data/qwen3-reverse/ensure-qwen-daemon.sh`
- Logged-in Chrome profile: `/opt/data/qwen3-reverse/vnc/browser-profile` (has the valid `token` cookie)
- Env: `/opt/data/qwen3-reverse/.env` (VALID_TOKENS, QWEN_CDP_URL)
- Hermes provider config: `/opt/data/config.yaml` (provider `custom:qwen-studio`, api `http://127.0.0.1:8000/v1`)

## Setup (first time)

1. Launch a headed Chrome on display `:1` (Xtigervnc must be running) with
   `--remote-debugging-port=9222` using the `browser-profile` user-data-dir, opening
   `https://chat.qwen.ai/`.
2. Expose it via noVNC + Cloudflare quick tunnel so the user can log in manually
   (see "Manual login / WAF captcha" below). Qwen's login is interactive — cannot be
   scripted.
3. Set `QWEN_CDP_URL=http://127.0.0.1:9222` in `/opt/data/qwen3-reverse/.env` so the
   proxy attaches to the existing Chrome instead of spawning its own.
4. Run the proxy: `cd /opt/data/qwen3-reverse && .venv/bin/uvicorn qwen_browser_proxy:app --host 127.0.0.1 --port 8000`
5. Register the Hermes provider in `config.yaml`:
   ```yaml
   providers:
     qwen-studio:
       name: qwen-studio
       api: http://127.0.0.1:8000/v1
       key_env: QWEN_STUDIO_API_KEY
       transport: chat_completions
       default_model: qwen3.7-plus
   ```
   The `QWEN_STUDIO_API_KEY` value must be one of the `VALID_TOKENS` in the proxy `.env`.

## CRITICAL GOTCHA 1 — proxy must attach to CDP, not spawn Chrome

If `QWEN_CDP_URL` is **unset**, the proxy falls into
`launch_persistent_context(headless=False)` and fails with:
`BrowserType.launch_persistent_context: ... Missing X server or $DISPLAY`.
Health then reports `ok:false`. **Always set `QWEN_CDP_URL` in `.env`.**

## CRITICAL GOTCHA 2 — empty content from system/assistant roles (the silent killer)

Qwen's web chat API (`/api/v2/chat/completions`, `t2t` mode) **returns an empty SSE
stream** (just `data: [DONE]`, no content) when the request message array contains:
- a leading `role: "system"` message, OR
- a pre-filled `role: "assistant"` turn (i.e. multi-turn history).

The Hermes gateway sends a `system` prompt + full conversation history on **every**
turn. So the proxy returned empty content constantly and the gateway logged
`Empty response (no content or reasoning) after 3 retries` — even though the browser
was logged in and a plain single-user curl worked.

**Fix (in `messages_to_qwen()`):**
- Drop all but the **last user turn** (each proxy completion opens a fresh Qwen chat,
  so history isn't needed by the API).
- Fold any `system` content in front of that last user turn.
- Treat `tool` role as `user` (tool results as context).
This is implemented in `qwen_browser_proxy.py::messages_to_qwen`.

Symptom/diagnosis recipe:
```bash
TOK=$(python3 -c "import re,json;m=re.search(r'VALID_TOKENS=(.*)',open('/opt/data/qwen3-reverse/.env').read(),re.M);print(json.loads(m.group(1))[0])")
# works (single user, no system):
curl -s http://127.0.0.1:8000/v1/chat/completions -H "Authorization: Bearer $TOK" \
  -d '{"model":"qwen3.7-plus","messages":[{"role":"user","content":"hi"}]}'
# breaks (system or assistant turn) -> empty content:
curl -s http://127.0.0.1:8000/v1/chat/completions -H "Authorization: Bearer $TOK" \
  -d '{"model":"qwen3.7-plus","messages":[{"role":"system","content":"x"},{"role":"user","content":"hi"}]}'
```

## CRITICAL GOTCHA 3 — Aliyun WAF / "Access Verification" captcha

Alibaba's anti-bot (baxia + Aliyun NC slide captcha) interferes in two ways:
- **Interactive captcha**: page shows "Access Verification / Drag to complete the
  puzzle" (cross-origin iframe, unsolvable by automation). Proxy `refresh_status()`
  detects it (`challenge:true`) and health reports `manual WAF verification required`.
  Fix: solve it manually in the noVNC browser.
- **Silent throttle**: after a burst of automated fetches, the chat-API XHR just
  **hangs** (no response, no error). Clearing the `isg`/`tfstk` risk cookie via CDP
  often clears the visible captcha but may NOT restore the throttled chat API — a
  fresh browser profile with a clean risk score is the reliable recovery.

Cookie clear recipe (clears the risk cookie that forces the captcha):
```python
import asyncio
from playwright.async_api import async_playwright
async def main():
    pw = await async_playwright().start()
    b = await pw.chromium.connect_over_cdp("http://127.0.0.1:9222")
    ctx = b.contexts[0]
    for c in [x for x in await ctx.cookies() if x["name"].lower() in ("isg","tfstk","sca","x-ap","cbc")]:
        try: await ctx.clear_cookies([c])
        except: pass
    await b.contexts[0].pages[0].reload(wait_until="domcontentloaded")
    await b.close(); await pw.stop()
asyncio.run(main())
```

## Manual login / WAF captcha via noVNC tunnel

The browser is on X display `:1` (VNC port 5901). Expose it:
1. `websockify --web=<novnc>/vnc/vendor/novnc 6082 127.0.0.1:5901` (no auth, or with
   `--web-auth --auth-plugin=BasicHTTPAuth --auth-source="user:pass"`).
2. `cloudflared tunnel --url http://127.0.0.1:6082` -> gives a `*.trycloudflare.com` URL.
3. User opens the URL, logs into chat.qwen.ai (no captcha on a fresh profile), solves
   any "Access Verification" if shown.

The **browser-profile** dir holds the working logged-in session (valid `token` cookie,
expiry ~2034). Do NOT delete it. If it gets throttled, create `browser-profile2`, log
in there via noVNC, and point the daemon at it.

## Health / verification

```bash
curl -s http://127.0.0.1:8000/health        # ok:true, challenge:false
curl -s http://127.0.0.1:8000/v1/models -H "Authorization: Bearer $TOK"   # 17 models
```
`/v1/models` falls back to a hardcoded 17-model catalog if the live `/api/models` call
is throttled, so the model list never truncates.

## Restart procedure (after a crash)

```bash
bash /opt/data/qwen3-reverse/ensure-qwen-daemon.sh   # starts Xvnc+Chrome+proxy if down
# if proxy mis-attached, kill and restart explicitly:
pkill -f "qwen_browser_proxy:app"; sleep 2
cd /opt/data/qwen3-reverse && .venv/bin/uvicorn qwen_browser_proxy:app --host 127.0.0.1 --port 8000 > proxy.log 2>&1 &
```

## Model list (17, verified live)

qwen3.7-plus, qwen3.8-max, qwen3.7-max, qwen3.6-plus, qwen3.6-max-preview,
qwen3.6-27b, qwen3.5-plus, qwen3.5-omni-plus, qwen3.6-35b-a3b, qwen3.5-flash,
qwen3.5-397b-a17b, qwen3.5-omni-flash, qwen3-max-2026-01-23, qwen-plus-2025-07-28,
qwen3-coder-plus, qwen3-vl-plus, qwen3-omni-flash-2025-12-01
