# Architecture Document: SignalFeed

**Status:** Draft v1.0  
**Target Model:** Single-User, Self-Hosted  
**Runtime:** Rootless Container (Docker & Podman)

---

## 1. Language Choice: Python vs. Rust

**Recommendation: Python** (specifically Python 3.12+ with `httpx`, `feedparser`, `trafilatura` or `readability-lxml`, `feedgen`, `PyYAML`, `pydantic`, and built-in `sqlite3`).

### Rationale
* **Library Ecosystem for "Odd" Sources:** News feeds and web pages exhibit messy HTML, inconsistent date formats, malformed XML, and encoding quirks. Python offers mature, battle-tested libraries for feed parsing, main-content extraction, and resilient HTTP fetching with minimal boilerplate.
* **Rapid Iteration on Prompts & Schemas:** Classification thresholds, source prompts, multi-day synthesis prompts, and JSON schemas evolve frequently during tuning. Python allows fast iteration and clear validation via `pydantic`.
* **Container & Resource Fit:** While Rust produces smaller static binaries and lower baseline memory (~10–30 MB vs. ~80–150 MB in Python), a background ingestion and LLM-backed service is I/O- and network-bound rather than CPU-bound. Running as a rootless Python container with slim dependencies is well within standard self-hosting budgets.

*(Note: If future scale or extreme resource constraints ever demand a rewrite, the modular port/adapter boundaries defined below keep a migration to Rust straightforward.)*

---

## 2. High-Level Architecture & Execution Model

SignalFeed is a single-user background service split into decoupled stages coordinated by an internal scheduler:

1. **Configuration Loader (`sources.yaml` + Environment Variables):**
   * `sources.yaml` is the single source of truth for global thresholds, target feeds, sources, regex pre-filters, and source-specific prompts. It is never stored in SQLite and is reloaded on each cycle so edits take effect without rebuilding or restarting the container.
   * Secrets and runtime settings (API keys, base URLs, polling intervals, digest schedule, bind port) are injected via environment variables.
2. **Ingestion & Normalization Worker (Runs as often as needed, e.g., every 15–60 min):**
   * Fetches RSS/Atom feeds and handles "odd" sources via configurable adapters (standard RSS/Atom, external RSS-Bridge/feed URLs, or best-effort HTML link/article extraction).
   * Applies regex title exclusions (`exclude_title_patterns`) and URL/GUID deduplication against SQLite before expensive steps.
   * Optionally fetches and cleans full article text (`fetch_full_text`), falling back cleanly to feed summaries/teasers on paywalls, blocks, or timeouts.
3. **Classification Stage (Decision Model):**
   * Evaluates candidate items to decide whether they pass relevance/noise criteria and assigns a score/confidence.
4. **Summarization & Multi-Day Trend Synthesis Stage (LLM):**
   * Summarizes qualifying items in German with structured output (`title_de`, `summary_markdown`, `why_it_matters`).
   * Periodically analyzes qualifying/stored items within a **rolling 7-day window** to detect cross-article patterns or multi-day trends and produces trend synthesis entries.
5. **Feed Publisher & HTTP Server (Feed updates once daily by default, configurable):**
   * Rebuilds static or cached RSS/Atom XML feed files on a configurable schedule (default: **once per day**, e.g., `07:00` local time, plus optional manual trigger/startup behavior if configured) using a **7-day lookback window** to bound volume.
   * Serves the generated XML feeds over HTTP internally behind a reverse proxy.

---

## 3. AI Model Interfaces: LLM & Decision Model

### 3.1 Generative LLM (Summarization & Multi-Day Trends)
* **Protocol:** OpenAI-compatible Chat Completions API (`base_url` + `api_key` + `model`).
* **Self-Hosting Support:** Works seamlessly with hosted OpenAI or OpenAI-compatible self-hosted runtimes (such as `llama.cpp` server, `vLLM`, or `Ollama`) simply by changing `LLM_BASE_URL`, `LLM_MODEL`, and `LLM_API_KEY`.
* **Structured Output:** Requests strict JSON output and validates it against a schema; retries or degrades gracefully if a model wraps JSON in markdown fences or omits optional fields.

### 3.2 Decision Model (Classification)
Because there is no single universal wire standard for specialized "System One" / decision models (such as TypeSafe AI's Jev or open self-hosted decision/triage models), SignalFeed defines an **internal Decision Port** with pluggable adapters:
* **Self-Hosted / Custom Decision Endpoint Adapter:** Sends a compact classification payload (source context, classification rule/question, title, excerpt/text) over HTTP/JSON to a self-hosted decision service or local sidecar, mapping its response into a normalized internal decision object:
  * `keep: bool`
  * `score: float` (normalized `0.0–10.0` or `1–10`)
  * `confidence: float | null`
* **OpenAI-Compatible Fallback / Alternative Adapter:** Allows using a fast/small OpenAI-compatible model (hosted or self-hosted) behind the same internal `DecisionProvider` interface when no dedicated decision endpoint is configured.

---

## 4. 7-Day Window & Daily Feed Publication

* **7-Day Rolling Window:** Both feed generation and multi-day trend analysis operate over a configurable **7-day window** (`7d` default) relative to the current time (`now - 7 days`). This keeps LLM context sizes predictable, prevents stale items from dominating feeds, and caps output volume.
* **Daily RSS Feed Publication:**
  * Ingestion and classification can run frequently so fresh items and local alerts (such as roadworks or upcoming events) are captured promptly.
  * **RSS output generation runs once per day by default** (configurable via cron/interval/time-of-day setting), assembling qualifying items from the active 7-day window (and any newly synthesized trend entries) into valid RSS 2.0 / Atom XML files per `target_feed`.
* **Multi-Day Trend Synthesis:**
  * Uses the generative LLM over items collected within the 7-day window (grouped by feed/topic/source cluster) to identify recurring themes, multi-day developments, or shifting narratives rather than isolated single-day news.
  * Trend items can be emitted into a dedicated trend section/feed or as clearly tagged synthesis items within the target feed, governed by the same daily publication cycle.

---

## 5. Persistence, "Odd" Sources & Resilience

### 5.1 SQLite Database (Articles & State Only)
* Stores **only** operational state and processed items: seen item keys (canonical URL / GUID / content hash), raw/extracted metadata, classification scores, generated German summaries, multi-day trend records, and fetch/error timestamps.
* **Configuration stays exclusively in `sources.yaml`** and environment variables.
* **SQLite Settings:** Enable `PRAGMA journal_mode = WAL;`, `PRAGMA synchronous = NORMAL;`, and a generous `busy_timeout` so concurrent reads (e.g., feed serving) and writes (ingestion/LLM workers) do not collide.
* **Pruning:** Periodically purge full text or old non-kept bodies beyond the retention/window period while retaining minimal deduplication keys (`url_hash` / `guid`) so already-seen items are not re-processed endlessly.

### 5.2 Handling "Odd" Sources Flexibly
Real-world sources rarely conform to clean RSS:
* **Fallback Chain for Ingestion:**
  1. Standard RSS/Atom parser (`feedparser` with tolerant charset/XML handling).
  2. Configurable feed-bridge/wrapper URL (e.g., an external RSS-Bridge instance).
  3. Best-effort HTML scraper/extractor for pages lacking feeds (extracting candidate links and main article text via readability/DOM heuristics + per-source CSS/XPath selectors when needed).
* **Politeness & Fault Tolerance:** Per-domain request pacing/rate limiting, explicit `User-Agent`, request timeouts, and per-source error isolation so one broken or blocked site never aborts the rest of the run.

---

## 6. Security, HTTPS & Container Deployment (Docker & Podman)

### 6.1 HTTPS Everywhere & Reverse Proxy
* **Outbound HTTPS:** All external feed fetches, web scrapes, and LLM/decision API calls enforce `https://` by default (allowing `http://` only explicitly for internal/loopback self-hosted endpoints if configured).
* **Inbound Behind Reverse Proxy:** SignalFeed listens on an internal HTTP port (e.g., `8080`) bound to a container/internal network. TLS termination (`https://`) is handled by an upstream reverse proxy (such as Caddy, Traefik, or Nginx).
* **Proxy Headers:** Configure the HTTP server to trust `X-Forwarded-Proto` and `X-Forwarded-Host` only from known proxy addresses so generated feed self-links and URLs reflect `https://`.

### 6.2 Rootless Docker & Podman
* **Non-Root User:** The container image defines a dedicated unprivileged user/group (e.g., `1000:1000`) and runs all processes as non-root.
* **Podman & Docker Compatibility:** Avoid Docker-socket assumptions or privileged flags; design file permissions and volume mounts so both `docker` and rootless `podman` work out of the box (including `--userns=keep-id` or matching UID/GID mappings on host directories).
* **Filesystem Layout (Persistent Volumes):**
  * `/config/sources.yaml` (can be mounted `:ro`)
  * `/data/signalfeed.db` (SQLite database)
  * `/output/*.xml` (generated RSS feeds served over HTTP)
* **Single-User Scope:** No user accounts, login flows, or multi-tenant isolation in v1; access control to the RSS endpoints is delegated to the reverse proxy, VPN, or private network.

---

## 7. Selected Libraries Summary

| Concern | Recommended Library / Tooling | Purpose |
| :--- | :--- | :--- |
| HTTP client | `httpx` | Async/sync HTTP with timeouts, retries, and proxy/TLS support |
| RSS/Atom parsing | `feedparser` | Tolerant parsing of varied RSS 1.0/2.0 and Atom feeds |
| Main text extraction | `trafilatura` (with fallback to `readability-lxml` / `beautifulsoup4`) | Extracting clean article text from HTML and handling odd pages |
| RSS/Atom generation | `feedgen` | Producing standards-compliant XML feeds |
| Config & validation | `PyYAML` + `pydantic` | Loading `sources.yaml` and validating LLM/decision outputs |
| Database | Python `sqlite3` (WAL mode) | Zero-dependency, reliable local persistence for items & state |
| HTTP server | `uvicorn` + `fastapi` (or lightweight `aiohttp`/`starlette`) | Serving generated XML feeds behind a reverse proxy |