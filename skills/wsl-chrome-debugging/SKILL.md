---
name: wsl-chrome-debugging
description: Drive a real Chrome browser from WSL to verify, screenshot, and debug a web app served from WSL — using the Windows Chrome install, a PowerShell TCP relay for CDP, and puppeteer-core. Use when running under Windows/WSL2 and you need browser verification, screenshots, console logs, or DevTools Protocol access, or when chrome-devtools-mcp fails with "Target closed".
---

# Driving Chrome from WSL (Windows host)

Under WSL2 there is **no working Linux Chrome** and you should not try to create one. Chrome is
installed on Windows. These are verified working steps.

## Do NOT do these

- Do not `apt-get install` Chromium or its libs (`libnss3`, `libnspr4`, `libasound2t64`).
- Do not use Playwright's bundled Chromium in `~/.cache/ms-playwright/` — it is missing those
  libs. `ldd` showing "not found" there is expected, not a fixable problem.
- `chrome-devtools-mcp` failing with `Target closed` is this environment issue, not a server bug.

## The two networking rules (WSL2 NAT)

Direction matters and the two are **not** symmetric:

| direction | how |
|---|---|
| Windows → WSL | `localhost` works. Chrome on Windows reaches a WSL dev server at `http://localhost:5173`. |
| WSL → Windows | `localhost` does NOT work. Use the gateway IP: `ip route show default \| awk '{print $3}'` |

Start dev servers with `--host 0.0.0.0` to be safe.

## Level 1 — screenshots and DOM, no CDP needed

Enough for "show me the UI" and many checks. No relay required.

```bash
CHR="/mnt/c/Program Files/Google/Chrome/Application/chrome.exe"

"$CHR" --headless=new --disable-gpu --no-first-run --user-data-dir=C:\\shot-profile \
  --window-size=1700,1000 --screenshot=C:\\Users\\<you>\\shot.png "http://localhost:5173/"

"$CHR" --headless=new --disable-gpu --user-data-dir=C:\\dump-profile \
  --dump-dom "http://localhost:5173/" > dom.html
```

Read the PNG back through `/mnt/c/...`.

**A fresh `--user-data-dir` is mandatory.** Without it the command attaches to the already-running
Chrome and prints "Opening in existing browser session" instead of starting a controllable one.

### The trap: `--virtual-time-budget` lies about live data

`--virtual-time-budget=25000` fast-forwards the virtual clock, so 25s of "virtual" time can pass
in well under a second of real time. WebSocket and network data are **real** time, so a live app
screenshots as empty or half-loaded and looks broken when it is fine. If the app streams data,
use Level 2 and a real `setTimeout` wait instead.

## Level 2 — full CDP (console, network, evaluate, MutationObserver)

Chrome **ignores `--remote-debugging-address=0.0.0.0`** and always binds `127.0.0.1:9222`, which
WSL2 NAT cannot reach. Verify with `netstat.exe -ano | grep 9222` — it shows `127.0.0.1:9222`.
`netsh interface portproxy` needs Administrator. So relay it with PowerShell (no admin needed).

### Step 1 — write the relay (once), to a Windows path

`C:\Users\<you>\cdp-relay.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
$listener = New-Object System.Net.Sockets.TcpListener([System.Net.IPAddress]::Any, 9223)
$listener.Start()
Write-Host "CDP relay listening on 0.0.0.0:9223 -> 127.0.0.1:9222"
while ($true) {
    $client = $listener.AcceptTcpClient()
    $ps = [PowerShell]::Create()
    [void]$ps.AddScript({
        param($client)
        $server = $null
        try {
            $server = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 9222)
            $cs = $client.GetStream(); $ss = $server.GetStream()
            $t1 = $cs.CopyToAsync($ss); $t2 = $ss.CopyToAsync($cs)
            [void][System.Threading.Tasks.Task]::WaitAny(@($t1, $t2))
        } catch { } finally {
            if ($client) { try { $client.Close() } catch {} }
            if ($server) { try { $server.Close() } catch {} }
        }
    }).AddArgument($client)
    [void]$ps.BeginInvoke()
}
```

A raw TCP relay is correct here — it passes the HTTP requests *and* the WebSocket upgrade.

### Step 2 — start Chrome and the relay (both backgrounded)

```bash
"/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" \
  --headless=new --disable-gpu --no-first-run --no-default-browser-check \
  --remote-debugging-port=9222 --user-data-dir=C:\\cdp-live \
  --remote-allow-origins=* about:blank

/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe \
  -NoProfile -ExecutionPolicy Bypass -File 'C:\Users\<you>\cdp-relay.ps1'
```

### Step 3 — verify from WSL

```bash
GW=$(ip route show default | awk '{print $3}')
curl -s "http://$GW:9223/json/version"     # expect {"Browser":"Chrome/…"}
```

### Step 4 — connect puppeteer-core

`/json/version` returns a `webSocketDebuggerUrl` pointing at Windows loopback. **Rewrite its host
to the relay** or the connection hangs:

```ts
import puppeteer from 'puppeteer-core';
const GW = '<gateway ip>';
const ver = await (await fetch(`http://${GW}:9223/json/version`)).json();
const browser = await puppeteer.connect({
  browserWSEndpoint: ver.webSocketDebuggerUrl.replace(/ws:\/\/[^/]+/, `ws://${GW}:9223`),
  defaultViewport: { width: 1700, height: 1000 },
});
const page = await browser.newPage();
page.on('console', m => console.log(`[${m.type()}] ${m.text()}`));
page.on('pageerror', e => console.log('PAGEERROR', e.message));
page.on('response', r => { if (r.status() === 404) console.log('404', r.url()); });
await page.goto('http://localhost:5173/', { waitUntil: 'domcontentloaded' });
await new Promise(r => setTimeout(r, 15000));   // REAL time — live data needs it
```

`bun add puppeteer-core` in a scratch dir; no browser download is needed since Chrome is on Windows.

## Verifying transient visual effects (flashes, animations)

Polling misses them. A cell flash lasts ~500ms; polling every 2.5s catches roughly 20% of them, so
"I saw zero" is **inconclusive, not negative**. Install a `MutationObserver` in the page and let it
count:

```ts
await page.evaluate(() => {
  (window as any).__f = { hits: 0 };
  new MutationObserver(ms => { for (const m of ms) {
    const cl = (m.target as HTMLElement).className;
    if (typeof cl === 'string' && cl.includes('ag-cell-data-changed')) (window as any).__f.hits++;
  }}).observe(document.body, { subtree: true, attributes: true, attributeFilter: ['class'] });
});
await new Promise(r => setTimeout(r, 45000));
console.log(await page.evaluate(() => (window as any).__f));
```

## Web Workers

A dedicated worker is its own CDP target. `page.on('workercreated', w => …)` reports it, and the
page-level `Network` domain does **not** see WebSockets opened inside the worker — their absence
from `Network.webSocketCreated` is expected and not evidence of a broken connection.

## Cleanup

```bash
/mnt/c/Windows/System32/tasklist.exe | grep -i chrome
/mnt/c/Windows/System32/taskkill.exe /F /PID <pid>
```
