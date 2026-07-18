---
name: reauth-google-voice
description: Re-authenticate the Google Voice tools of a self-hosted Google Workspace MCP when they stop sending or reading SMS. Google Voice has no public API or service-account — the MCP authenticates with scraped browser cookies that expire, and Google bot-walls headless/automated logins, so an in-MCP "reconnect" that drives a headless browser never completes. This skill uses the path that reliably works: launch a debug Chrome on its own profile, sign in once by hand, extract the full cookie jar (including the HttpOnly SID/HSID/SSID/APISID/SAPISID auth cookies and the COMPASS voice-api token) over the Chrome DevTools Protocol, and inject it via the MCP's `set_voice_cookies` tool. Trigger phrases "google voice not working", "reauth voice", "reauth google voice", "voice mcp 401", "voice connector down", "send_voice_sms fails", "voice session expired", "voice cookies expired", "text won't send", "get_voice_session_status unhealthy", "Voice API returned 401 for POST /account/get".
---

## Requirements
- A Google Workspace MCP that exposes the Voice tools `get_voice_session_status`, `set_voice_cookies`, and `send_voice_sms`.
- Local Chrome + Python with Playwright (`py -3.11 -m pip install --user playwright`, or `pip install --user playwright`).

## Why the obvious path fails (don't retry it)
Google bot-walls headless/automation logins, so any "connect" flow that drives a headless browser stalls and its handoff link expires before sign-in completes. The required auth cookies — `SID`, `HSID`, `SSID`, `APISID`, `SAPISID` — are **HttpOnly**, so page JavaScript and extensions can't read them. Only a full browser *session* can surface them, via the DevTools Protocol's `storage_state()`. Driving a real, user-logged-in Chrome over CDP is the reliable route.

## Recipe

### 1. Launch a debug Chrome on its own profile, pointed at the Voice login
Write `launch-voice-debug.ps1` (Windows shown; adapt the Chrome path for macOS/Linux) and run
`powershell.exe -NoProfile -ExecutionPolicy Bypass -File launch-voice-debug.ps1`:
```powershell
$dst = "$env:TEMP\chrome-debug-voice"
$chrome = 'C:\Program Files\Google\Chrome\Application\chrome.exe'
if (-not (Test-Path $chrome)) { $chrome = 'C:\Program Files (x86)\Google\Chrome\Application\chrome.exe' }
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Where-Object { $_.CommandLine -like "*chrome-debug-voice*" } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
Start-Sleep -Milliseconds 500
$proc = Start-Process $chrome -PassThru -ArgumentList @(
  "--user-data-dir=$dst","--remote-debugging-port=9224","--remote-allow-origins=*",
  "--no-first-run","--no-default-browser-check","--restore-last-session=false",
  "--new-window","https://accounts.google.com/ServiceLogin?service=voice&continue=https://voice.google.com/u/0/messages")
"launched pid=$($proc.Id)"
for ($i=0; $i -lt 25; $i++){ Start-Sleep -Milliseconds 800
  try { $v=Invoke-RestMethod "http://127.0.0.1:9224/json/version" -TimeoutSec 3; "CDP_UP="+$v.Browser; break }
  catch { if($i -eq 24){"CDP_DOWN: $($_.Exception.Message)"} } }
```
- **Chrome ≥136 ignores `--remote-debugging-port` on the normal profile dir** for security → a *separate* `--user-data-dir` is the essential trick.
- It kills only its own prior `chrome-debug-voice` instance, never your main Chrome.
- **The debug profile stays signed in across relaunches**, so most future re-auths need no new login.

### 2. Check auth state and extract cookies over CDP
Write `voice_extract.py` and run it (use forward slashes in POSIX shells so the path isn't mangled):
```python
import json
from playwright.sync_api import sync_playwright
REQ = ["SID", "HSID", "SSID", "APISID", "SAPISID"]
with sync_playwright() as p:
    b = p.chromium.connect_over_cdp("http://127.0.0.1:9224")
    ctx = b.contexts[0]
    vpage = next((pg for pg in ctx.pages if "voice.google.com" in pg.url), None)
    if vpage is None:
        vpage = ctx.pages[0] if ctx.pages else ctx.new_page()
        try: vpage.goto("https://voice.google.com/u/0/messages", wait_until="domcontentloaded", timeout=20000)
        except Exception as e: print("nav_err", type(e).__name__)
    try: vpage.bring_to_front()
    except Exception: pass
    url = vpage.url
    authed = ("voice.google.com" in url) and ("accounts.google.com" not in url) and ("ServiceLogin" not in url)
    st = ctx.storage_state()
    cookies = {c["name"]: c["value"] for c in st.get("cookies", [])
               if c.get("domain", "").endswith("google.com") and "youtube" not in c.get("domain", "")}
    have = [k for k in REQ if k in cookies]
    json.dump({"cookies": cookies}, open("vc.json", "w"))
    print("URL:", url); print("authed_guess:", authed)
    print("required_present:", sorted(have), "of", len(REQ)); print("total_google_cookies:", len(cookies))
    b.close()
```
- `authed_guess: False` / a `ServiceLogin` URL → **sign in by hand** in the debug Chrome window with the account that owns the Voice number, get to your text threads, then re-run. Never automate the password/2FA.

### 3. Inject the cookies into the MCP
Read `vc.json` and call `set_voice_cookies` with the **inner `{name: value}` dict** (not the `{"cookies": …}` wrapper). Keep the `.google.com` set — it must include `OSID`, `__Secure-1PSIDTS`, `SIDCC`, and **`COMPASS`** (which carries the `voice-api=` token).

### 4. Verify
`get_voice_session_status` should show all 5 required cookies present. A `400 "JSPB Fava message don't accept top-level braces"` is **benign** — a quirk in the smoke-test's own request payload, not an auth failure (a dead session returns **401**, not 400). Confirm for real by calling `send_voice_sms` (to your own number, or the real recipient if that's the task).

### 5. Cleanup
- Close only the debug instance (match the `chrome-debug-voice` user-data-dir substring), never your main Chrome.
- **Delete `vc.json`** — it holds live session cookies.

## Gotchas
- **Session lifetime is short** — `SIDCC` / `__Secure-1PSIDTS` rotate; if it's been a while since extraction, relaunch the still-signed-in debug profile and re-extract before a batch of sends.
- Never enter the Google password or 2FA, and never kill the user's main Chrome — target only the `chrome-debug-voice` profile.
- A keep-alive that merely re-navigates an existing session can only **extend a still-valid** session; it cannot resurrect one Google has invalidated (it may even report success while storing dead cookies). This hand re-seed is the real recovery.
