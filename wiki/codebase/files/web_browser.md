---
type: file
status: ingested
---
# web_browser.py

💡 **Role**: `WebBrowser`, a Playwright-backed real-browser automation tool (not HTTP scraping) — DOM interaction, screenshots, console capture, and accessibility-tree snapshots, explicitly modelled on VS Code 1.110's native browser integration per its own module docstring. [[perception-and-actuation]] already covers this file's narrative ("a TOOL only... `brain_core.should_browse()` owns the decision of WHEN to browse") — confirmed accurate this pass, both from this file's own design-rules docstring and from directly reading [[cognitive_loop]]'s `_execute_web_browsing()`, which does exactly what the docstring claims: checks `agent.web_browser.browse_queue`, pops up to two URLs per cognitive cycle, calls `browser.browse()` directly. This page adds the full method catalog and, more importantly, **a real self-caught correction**: [[agent]]'s existing page says the browser routes pass through `agent.browser`. That's wrong. Every actual call site — this file's own integration helper, every route in `agent.py`, and `cognitive_loop.py`'s caller — uses `agent.web_browser`. Confirmed by direct search, not assumed; [[agent]] corrected below.

## Imports

External only: `asyncio`, `base64`, `json` (imported, never referenced — see Problems/Solutions), `logging`, `time`, `collections.deque`, `dataclasses` (`dataclass`, `field`), `typing`, `urllib.parse` (`urljoin`, `urlparse`). Two guarded optional imports: `playwright.async_api` (`Browser`, `BrowserContext`, `ConsoleMessage`, `Page`, `Playwright`, `async_playwright` — sets `PLAYWRIGHT_AVAILABLE`) and `bs4.BeautifulSoup` (sets `BS4_AVAILABLE`). No internal project imports at all — this file only ever touches its caller via the `agent` object passed into `__init__`, duck-typed (`agent.agent_id`, `agent.memory.remember()`), never imported.

**`BS4_AVAILABLE`/`BeautifulSoup` is genuinely dead** — confirmed by search, referenced nowhere past the import guard itself. Every extraction method in this file goes through Playwright's own `page.evaluate()` (a JS DOM tree-walker for visible text, `querySelectorAll` for links/metadata/headings) or `page.accessibility.snapshot()`, never BeautifulSoup. This is a different situation from `PLAYWRIGHT_AVAILABLE`, which **is** load-bearing — `WebBrowser.__init__` raises `RuntimeError` immediately if it's `False`.

## Classes

### `ConsoleEntry` (dataclass)

`level`, `text`, `timestamp`, `url` — one browser console message.

### `DOMElement` (dataclass)

`role`, `name`, `tag`, `selector`, `attributes`, `children` (recursive) — a simplified accessibility-tree node. `to_dict()` recurses into a plain dict. Declared but not actually the type `_extract_accessibility_tree()` returns (that method returns flat dicts directly via `_flatten_a11y()` — see below); not confirmed whether anything else in this file constructs a `DOMElement` at all.

### `PageSnapshot` (dataclass)

The rich "what the agent saw" record: `url`, `timestamp`, `title`, `visible_text`, `accessibility_tree`, `screenshot_b64`, `console_logs`, `js_errors`, `links`, `metadata`, `headings`, `status_code`, `load_time_ms`.

- `get_summary(max_length=600)` — title/URL/load-time/status/first-3-JS-errors header plus a truncated text preview.
- `to_memory_dict()` — the exact shape written into agent memory: type `"web_page"`, truncated text/console/errors/links (500 chars, 20 logs, 10 errors, 15 links respectively), first 50 accessibility-tree entries, a `has_screenshot` boolean rather than the screenshot data itself.

### `InteractionResult` (dataclass)

`success`, `action`, `selector`, `message`, `screenshot_b64`, `console_after` — the return shape for every DOM-interaction method below.

### `WebBrowser`

**Methods:**

- `__init__(agent)` — raises `RuntimeError` if Playwright isn't installed. Sets up permission sets (`allowed_domains`, `allowed_urls`), state (`visited_urls`, `page_cache`, `browse_queue` — a `deque(maxlen=200)`), a 20-entry `_recent_domains` deque (see note below), per-domain rate limiting (`min_request_interval = 2.0`s), and a `stats` dict. Playwright objects (`_playwright`/`_browser`/`_context`) are `None` until first use.
- `_ensure_browser()` _(async)_ — lazily launches headless Chromium (`--no-sandbox --disable-dev-shm-usage`) and a context (1280×800, custom user agent `DivineWorldAI/{agent_id}`) on first call; no-ops if already running.
- `close()` _(async)_ — tears down context → browser → playwright in order.
- `update_allowed_websites(websites)` — rebuilds `allowed_domains`/`allowed_urls` from a frontend-supplied list; `_extract_domain()` strips `www.` and lowercases.
- `_is_url_allowed(url)` — exact URL match, URL-prefix match, or domain/subdomain match against the allow-lists.
- `_rate_limit(url)` _(async)_ — sleeps out the remainder of `min_request_interval` per domain before proceeding.
- `browse(url, take_screenshot=True, wait_for="networkidle")` _(async)_ — the core method, called directly by [[cognitive_loop]]. Permission check → cache check (cache hits skip navigation entirely — **no TTL or invalidation anywhere in this file**, so a cached page can be arbitrarily stale for the life of the process) → rate limit → ensure browser → open a page, hook `console`/`pageerror` listeners → `page.goto()` (30s timeout) → 0.5s settle wait → extract text/accessibility-tree/links/metadata+headings → optional screenshot → build and cache a `PageSnapshot` → `agent.memory.remember()` tagged `['web', 'browsing', 'content']` → `_enqueue_links()` on discovered links → returns the snapshot (or `None` on disallowed URL, timeout, or any exception).
- `click(url, selector, take_screenshot=True)` / `type_into(url, selector, text, submit=False, take_screenshot=True)` / `scroll(url, direction="down", amount=600, take_screenshot=True)` / `evaluate(url, js_code)` _(all async)_ — each opens its own fresh page, navigates, performs the one interaction, optionally screenshots, closes the page, returns an `InteractionResult` (or a raw value for `evaluate()`). None of these four re-use `browse()`'s cache or the calling agent's own already-open tab — every interaction is a full fresh navigation.
- `screenshot(url, full_page=False)` _(async)_ — same fresh-page pattern, returns a base64 PNG string directly (not wrapped in `InteractionResult`).
- `_extract_visible_text(page)` _(async)_ — a JS `TreeWalker` over `document.body` skipping `script`/`style`/`noscript` and anything hidden via computed style, falling back to `page.inner_text("body")` if the walker throws.
- `_extract_accessibility_tree(page)` _(async)_ — `page.accessibility.snapshot(interesting_only=True)`, flattened by `_flatten_a11y()`.
- `_flatten_a11y(node, depth=0, max_depth=6)` — recursive flatten into `{role, name, value, checked, depth}` dicts, empty values dropped, depth-capped at 6.
- `_extract_links(page, base_url)` _(async)_ — every `a[href]`, resolved against `base_url`, fragment-stripped, deduplicated, `http(s)`-only.
- `_extract_metadata_and_headings(page)` _(async)_ — one `page.evaluate()` call pulling every `meta[name]`/`meta[property]` plus every `h1`/`h2`/`h3` in one pass.
- `_enqueue_links(links)` — auto-queues discovered links, skipping a fixed set of path substrings (`login`, `signin`, `register`, `cart`, `checkout`, `logout`), already-visited URLs, and disallowed URLs.
- `add_url_to_queue(url)` — the explicit-request entry point. Per its own docstring, called only by `agent.process_chat()` when the user's message contains an action trigger word alongside a URL — confirmed directly: `process_chat()` regex-matches any `https?://` URL in the message, always remembers it as a `url_mentioned` memory event, and additionally calls this method only if the message also contains one of `('check', 'look', 'browse', 'visit', 'read', 'open', 'go to', 'see', 'find')` anywhere (a plain substring match on the whole lowercased message, not tied to proximity with the URL).
- `autonomous_browse(max_pages=3, take_screenshots=True)` _(async)_ — drains the queue up to `max_pages`, calling `browse()` for each with a 1s pause between. Its own docstring is explicit that this is "kept for external callers that want to drive the browser directly" — [[cognitive_loop]] doesn't call this; it drives `browse()` itself one URL at a time so it can interleave memory storage and speech between pages. Confirmed genuinely intentional, not an orphaned duplicate: a FIX comment in `cognitive_loop.py`'s own loop cites this method's correct `popleft()` usage as the reference it was brought in line with, after a bug where the cognitive-loop path was peeking (`browse_queue[0]`) instead of dequeuing and re-browsing the same URL forever.
- `search_cached_pages(query, limit=5)` — full-text search over `page_cache` only (not persistent memory), title matches weighted 10× over body-text matches, plain substring counting rather than any semantic ranking.
- `get_stats()` — merges the running `stats` dict with live counts (`allowed_domains`, `cached_pages`, `queue_size`, `visited_urls`) and `recent_domains` (a FIX-comment-documented addition — this list previously wasn't exposed at all).

**A note on `_recent_domains`**, per its own in-line FIX comment (referencing "Chat & Web GRPO plan §3.6b" — a design doc not available to this ingest, but named consistently across this file and a matching comment in [[cognitive_loop]], so evidently real): this 20-entry deque is for introspection only (e.g. a debug panel), not the reward path. `RewardSystem`'s own diversity scoring maintains a **separate** `_recent_domains` of its own, since it has no live reference to this `WebBrowser` instance and instead reads the domain straight off each web event's payload. Two same-named trackers, confirmed genuinely independent rather than a duplicate-by-accident — the kind of thing worth checking rather than assuming either way.

## Module-level function

- `add_web_browsing_to_agent(agent)` — the actual integration point. Returns `None` with a logged warning if Playwright isn't installed (a **softer** failure than constructing `WebBrowser` directly, which raises). On success, constructs a `WebBrowser` and sets **`agent.web_browser`** — not `agent.browser`.

## Problems (faced by traditional AI systems / LLMs)

Giving an agent open-ended live web access safely is a compound problem, not one thing: unrestricted access risks the agent wandering into anything (or being used to exfiltrate/attack), un-rate-limited requests look like abuse to the sites being visited, and — specific to a system meant to _learn_ from what it reads — a raw HTML dump or a screenshot alone isn't something a language-learning pipeline with no pretrained assumptions can use directly. A second, more architectural problem: distinguishing an agent's own autonomous curiosity from a human explicitly asking it to go look at something, so the tool doesn't either ignore direct requests or treat every idle mention of a URL as an instruction to act.

## Solutions

Domain/URL allow-listing plus a 2-second per-domain rate limit addresses the first problem directly. The tool/decider split — this file never decides to browse, `brain_core.should_browse()` does, confirmed by direct read of both this file's design rules and `cognitive_loop.py`'s actual call site — keeps browsing an explicit, gated action rather than an ambient capability. The learnability problem gets a real answer: `_extract_visible_text()`'s DOM-walker plus the flattened accessibility tree turn a rendered page into token-efficient structured text before it ever reaches memory, and `to_memory_dict()`'s tagging (`web`, `browsing`, `content`) is what lets the language-learning stack pick it up downstream. The autonomy-vs-request problem is solved with a simple, honestly-not-sophisticated trigger-word list in `agent.process_chat()` rather than any deeper intent classification — adequate for "did the user ask me to look at this," not attempting more than that.

## Files Required

None imported — `agent.memory`/`agent.agent_id` are duck-typed against whatever `agent` object is passed to `__init__`, the same pattern [[chat_system]] uses for `agent.brain`.

## Files Used In

- [[agent]] — `add_web_browsing_to_agent()` is lazily imported and called once during `NPCAgent` construction, setting `agent.web_browser`. **Two things on that page need correcting, both confirmed here**: (1) its browser-routes description says `agent.browser` — every real call site, in that same file, uses `agent.web_browser`; (2) its Files Required list doesn't mention `web_browser.py` at all despite the lazy import. See the correction noted on [[agent]] directly.
- [[cognitive_loop]] — `_execute_web_browsing()` drives `browser.browse()` one URL at a time, confirmed correctly using `agent.web_browser` throughout.
- `agent.py`'s own `/browser/*` routes (`navigate`, `click`, `type`, `scroll`, `screenshot`, `stats`, `history`, `allowed_sites`) — per that file's own comment, deliberately live on the per-agent app rather than `main.py` "because the browser instance lives inside `NPCAgent`." Two of the eight are broken, confirmed by direct read, neither being a `web_browser.py` bug: `GET /browser/screenshot` calls `web_browser.screenshot_jpeg()`, a method that doesn't exist anywhere on this class (the real method is `screenshot()`, returning PNG not JPEG) — every call raises and 500s. `POST /browser/scroll` checks `hasattr(web_browser, "_page")` before doing anything — `WebBrowser` never sets `self._page` (every interaction method uses a local `page` variable, closed before returning), so that check is always `False` and the route silently does nothing while still returning `{"status": "ok"}`. The real, working `scroll()` method on this class sits unused by that route entirely.
- Not yet ingested at file level: the Electron frontend (`dw_agent`) is the actual caller of all eight `/browser/*` routes, per [[electron-frontend]]'s existing topic-level coverage and the "frontend-only browsing" comment in `agent.py` itself.