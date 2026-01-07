# LLM prompt: implement nodriver-style anti-bot behaviors in Go (chromedp)

You are a large language model tasked with **adding anti-bot / anti-detection features** to a Go codebase that uses **chromedp**. Your deliverable is a concrete implementation plan and code changes in Go.

## Context: nodriver’s observable anti-bot behavior (detailed)

### What the project claims (from README)
- Positions itself as the successor to **undetected-chromedriver**, avoiding Selenium/WebDriver and using direct DevTools Protocol (CDP) control.
- States that “direct communication” improves resistance against WAFs and that defaults are optimized to “stay undetected for most anti‑bot solutions.”

### Concrete implementation details in the codebase

#### 1) Direct CDP control (no WebDriver)
**Claim:** it avoids Selenium/WebDriver binaries.

**Implementation detail:** the package’s `Browser`/`Tab` objects talk to Chrome via the DevTools protocol connection (CDP), and the `start()` helper returns a `Browser` object created from a `Config` instance rather than a WebDriver session. This removes the WebDriver automation surface entirely (no `chromedriver` binary).

**Relevant code paths:**
- `nodriver.core.util.start()` constructs `Config` and then `Browser.create(config)`.
- `nodriver.core.browser.Browser.create()` spawns the browser and uses CDP internally.

#### 2) Default launch arguments (stealth-ish defaults)
**Claim:** “best practice defaults” for quick startup and staying undetected.

**Implementation detail:** `Config` contains a **default argument list** that is always passed to Chrome unless overridden. These flags can reduce common automation artifacts, popups, and infobars that can be fingerprinted:

```python
# nodriver/core/config.py (Config._default_browser_args)
[
    "--remote-allow-origins=*",
    "--no-first-run",
    "--no-service-autorun",
    "--no-default-browser-check",
    "--homepage=about:blank",
    "--no-pings",
    "--password-store=basic",
    "--disable-infobars",
    "--disable-breakpad",
    "--disable-dev-shm-usage",
    "--disable-session-crashed-bubble",
    "--disable-search-engine-choice-screen",
]
```

When building the final CLI args, `Config.__call__()` also sets:
- `--user-data-dir=<temp profile>` (fresh profile per run, unless you pass a custom profile).
- `--disable-features=IsolateOrigins,site-per-process` (and optionally `DisableLoadExtensionCommandLineSwitch` if extensions are used).
- `--headless=new` if headless mode is enabled.
- `--no-sandbox` if sandbox is disabled.
- `--remote-debugging-host` and `--remote-debugging-port` if you attach to an existing session.

**Implication:** these are the concrete launch flags that shape the browser’s automation surface and behavior out of the box.

#### 3) Fresh profile + cleanup
**Claim:** uses a fresh profile and cleans up after exit.

**Implementation detail:** when `user_data_dir` is not provided, `Config` uses `temp_profile_dir()` and later cleanup logic in `deconstruct_browser()` removes the profile directory. This reduces cross‑run state and can mitigate certain detection vectors tied to stale profiles.

#### 4) Headless UA cleanup
**Claim:** avoid typical headless detection heuristics.

**Implementation detail:** in headless mode the Tab prepares a sanitized UA string by removing the literal `"Headless"` token.

```python
# nodriver/core/tab.py (_prepare_headless)
resp = await self._send_oneshot(cdp.runtime.evaluate(expression="navigator.userAgent"))
...
ua = response.value
await self._send_oneshot(
    cdp.network.set_user_agent_override(user_agent=ua.replace("Headless", ""))
)
```

**Implication:** removes a common headless marker at runtime without changing other UA details.

#### 5) “Expert mode” overrides (explicitly *more* detectable)
**Claim:** expert mode is for debugging and can make you more detectable.

**Implementation details:**
- `start(expert=True)` documents that it adds **`--disable-web-security`** and **`--disable-site-isolation-trials`**. This relaxes isolation and can surface different fingerprints.
- When expert mode is enabled, `_prepare_expert()` injects a script that forces `Element.attachShadow()` to always open shadow roots.

```javascript
// nodriver/core/tab.py (_prepare_expert)
Element.prototype._attachShadow = Element.prototype.attachShadow;
Element.prototype.attachShadow = function () {
    return this._attachShadow({ mode: "open" });
};
```

**Implication:** these changes are helpful for debugging and DOM access, but they are not stealth techniques; they can increase detectability.

#### 6) Visual checkbox helper (Cloudflare)
**Claim:** the README advertises a `tab.cf_verify()` helper to click Cloudflare’s “verify” checkbox.

**Implementation detail:** the README describes image‑based detection using OpenCV templates to locate the checkbox and click it. This is a human‑verification helper rather than a core stealth mechanism and only works outside expert mode.

### Summary of “bypass” mechanics (as implemented)
- **No WebDriver / chromedriver**: avoids the standard WebDriver fingerprinting surface by speaking CDP directly.
- **Launch defaults**: flags like `--disable-infobars`, `--no-first-run`, and `--disable-features=IsolateOrigins,site-per-process` reduce automation artifacts and make behavior more “normal.”
- **Headless UA cleanup**: strips the `Headless` token from the UA string when running headless.
- **Fresh profile per run**: reduces persistent state that can build a fingerprint over time.
- **Expert-mode features are *not stealth***: they intentionally trade stealth for debugging convenience.

## Goals
Implement the **same observable behaviors** in Go using chromedp. Focus on parity with the behaviors above, not on inventing new stealth techniques.

## Required features to implement
1) **Direct CDP control (no WebDriver)**
   - Ensure the implementation uses chromedp / CDP only (no WebDriver).

2) **Default launch arguments**
   - Add a default list of Chromium flags equivalent to:
     - `--remote-allow-origins=*`
     - `--no-first-run`
     - `--no-service-autorun`
     - `--no-default-browser-check`
     - `--homepage=about:blank`
     - `--no-pings`
     - `--password-store=basic`
     - `--disable-infobars`
     - `--disable-breakpad`
     - `--disable-dev-shm-usage`
     - `--disable-session-crashed-bubble`
     - `--disable-search-engine-choice-screen`
   - Add computed flags that mirror `Config.__call__()` behavior:
     - `--user-data-dir=<temp profile>` when no profile is provided.
     - `--disable-features=IsolateOrigins,site-per-process` (and append `DisableLoadExtensionCommandLineSwitch` if extensions are used).
     - `--headless=new` when headless.
     - `--no-sandbox` when sandbox is disabled.
     - `--remote-debugging-host` and `--remote-debugging-port` when attaching to an existing instance.

3) **Fresh profile + cleanup**
   - If no profile directory is supplied, create a temporary profile directory and delete it on shutdown.

4) **Headless UA cleanup**
   - When running headless, remove the literal `"Headless"` token from the UA string by querying the current UA via CDP and overriding it via `Network.setUserAgentOverride`.

5) **Expert mode (more detectable)**
   - Add an `expert` mode that:
     - Adds `--disable-web-security` and `--disable-site-isolation-trials`.
     - Injects a script on new document creation to force open shadow roots:
       ```js
       Element.prototype._attachShadow = Element.prototype.attachShadow;
       Element.prototype.attachShadow = function () {
           return this._attachShadow({ mode: "open" });
       };
       ```
   - Document clearly that expert mode can increase detectability.

6) **README-style notes (optional)**
   - If the Go repo has docs, add a short “anti-bot defaults” section referencing the above behaviors (do not oversell; mention limitations).

## Output requirements
- Provide actual Go code changes (or a patch) using `chromedp` and `chromedp/cdproto`.
- Show how to build the **execution allocator** with the full set of flags.
- Show how to create and clean up a temporary profile directory.
- Show the CDP calls for user agent override and script injection.
- Include comments that map back to the behaviors listed above.

## Constraints
- Do not add unrelated stealth features not present above.
- Keep changes minimal and focused.
- Prefer standard library for temp dirs/cleanup.
- If the codebase already has config structs, integrate there instead of inventing a new configuration system.

## Deliverable format
1) Short summary of the changes.
2) Code or patch.
3) Notes about testing / validation steps.
