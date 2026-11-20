# Product Requirements Document (PRD)
**Product Name:** SignalFeed (Working Title)  
**Version:** 1.0  
**Target Audience:** Self-Hosting End User (Single-User)  
**Deployment Model:** Self-Hosted Background Service  

---

## 1. Product Vision & Problem Statement

### Current Situation
Standard news feeds and RSS subscriptions contain a high volume of short-lived daily noise (breaking alerts, stock tickers, sensationalism, day-to-day political squabbles). Consuming this content takes considerable time while burying structural, medium- to long-term developments that actually matter. At the same time, certain regional updates (e.g., local roadworks, municipal rules, cultural events) are time-sensitive and practical, requiring targeted extraction rather than broad news consumption.

### Product Goal
A lightweight, self-hosted service that automatically fetches configured news sources, evaluates, filters, and summarizes them in German using a Large Language Model (LLM) based on user-defined criteria, and exposes the relevant results as clean, custom RSS feeds readable in any standard RSS client.

---

## 2. Core Functional Requirements

### 2.1 Configuration Management (`sources.yaml`)
* **YAML Configuration Ingestion:** The service reads a structured `sources.yaml` configuration file upon startup and reloads it on each execution cycle so changes take effect without restarting the service.
* **Supported Configuration Parameters:**
  * **Global Settings:** Minimum relevance score threshold, target output language, and a global base system prompt.
  * **Output Feed Definitions:** Definition of one or more target RSS feeds (e.g., *Macro/Long-Term* vs. *Local/Practical*).
  * **Source Definitions:** Source URL, source type, assigned target feed, full-text fetching toggle, regex exclusion patterns (`exclude_title_patterns`), and a source-specific LLM prompt.

### 2.2 Source Fetching & Pre-Filtering (Ingestion)
* **RSS/Atom Feed Polling:** Scheduled, periodic fetching of all configured standard RSS and Atom feeds.
* **Non-RSS Web Sources:** Support for sources marked as `scraper_or_rss_bridge` (either by consuming an external RSS-Bridge feed URL or by extracting current article links and content directly from the target webpage).
* **Rule-Based Pre-Filtering (Cost & Noise Reduction):**
  * Articles whose titles match any regex in `exclude_title_patterns` (e.g., paywall markers like `[heise+]` or `SUR+`, routine stock market tickers) are discarded immediately before fetching full text or invoking the LLM.
* **Deduplication & State Persistence:**
  * Previously processed articles (identified by canonical URL or feed GUID) are persisted locally and never re-fetched or re-evaluated by the LLM.
* **Optional Full-Text Extraction:**
  * When `fetch_full_text: true` is configured, the service fetches and extracts the main article body from the linked webpage to give the LLM sufficient context for evaluation.
  * If full-text extraction fails due to a paywall or access restriction, the service gracefully falls back to the available RSS teaser rather than failing the item.

### 2.3 LLM Evaluation, Filtering & Summarization
* **Prompt Composition:** For each new article, the service combines the global `base_system_prompt`, the source-specific `prompt`, and the article's title and text.
* **Structured Output:** The LLM returns a structured JSON response containing:
  * Relevance score (`1–10`)
  * Inclusion decision (`keep`: boolean)
  * Concise German headline (`title_de`)
  * Compact German summary in Markdown (`summary_markdown`)
  * One-sentence explanation of practical or long-term significance (`why_it_matters`)
* **Threshold Filtering:** Only items where `keep` is `true` and `score` is greater than or equal to `min_relevance_score` are published to the target feed.

### 2.4 RSS Feed Generation & Delivery
* **Valid RSS/Atom Output:** For every `target_feed` defined in `sources.yaml`, the service generates a separate, standards-compliant XML feed file (e.g., `feed_makro.xml`, `feed_lokal.xml`).
* **Feed Item Structure:**
  * **Title:** `title_de` (optionally enriched with the relevance score and original source name).
  * **Link:** Direct URL to the original article.
  * **Description/Content:** Cleanly formatted HTML combining `summary_markdown`, `why_it_matters`, and source attribution.
  * **Publication Date:** Original publication timestamp of the article (or processing timestamp if unavailable).
* **HTTP Feed Serving:** The service exposes a lightweight HTTP endpoint so standard RSS readers can subscribe to the generated XML feeds directly over the local network or VPN.
* **Retention Limit:** Each output feed retains a configurable maximum number of recent items (e.g., the latest 100 entries) to keep feed files fast and manageable.

---

## 3. Non-Functional Requirements

### 3.1 Operations & Configuration
* **Persistent Local Storage:** Configuration (`sources.yaml`), state tracking (processed item IDs and generated feed entries), and output XML files are stored in local directories so data persists across updates.
* **Environment-Based Secrets & Runtime Settings:** Sensitive credentials and operational parameters are configured via environment variables (or a `.env` file):
  * LLM API key, API base URL (compatible with the OpenAI API standard, including local LLM servers), and model identifier.
  * Polling schedule / interval (e.g., cron schedule or interval in minutes).
  * HTTP port for serving the generated RSS feeds.

### 3.2 Reliability & Source Etiquette
* **Fault Tolerance:** If an individual source fails (timeout, malformed XML, HTTP 403/404), the error is logged without interrupting the processing of remaining sources.
* **Rate Limiting & Identification:** The service enforces a short delay between consecutive HTTP requests to the same domain and identifies itself with a clear `User-Agent` header.
* **Token Efficiency:** Excessively long article texts are truncated cleanly before being sent to the LLM to respect context limits and control API costs.

### 3.3 Observability
* **Clear Operational Logs:** Standard output logs provide a clear summary per run, including:
  * Number of new items discovered per source
  * Number of items skipped via deduplication or regex pre-filters
  * LLM evaluation outcomes (score, kept vs. discarded)
  * Fetch or LLM processing errors

---

## 4. Out of Scope (v1.0)
To keep the system simple, maintainable, and reliable, the following features are explicitly excluded from version 1.0:
* No built-in web reader UI — reading takes place entirely in the user's preferred RSS client.
* No multi-user management or built-in authentication layer (network-level access control is handled externally via local network, VPN, or reverse proxy).
* No circumvention of hard paywalls or login-protected content.

---

## 5. Success Criteria (Definition of Done)
1. The service reads the `sources.yaml` configuration file cleanly and adapts to changes on subsequent runs without requiring code modifications.
2. New articles are fetched on schedule, pre-filtered, evaluated by the LLM, and summarized into German according to source-specific rules.
3. The generated RSS feeds (`feed_makro.xml` and `feed_lokal.xml`) pass standard RSS validation and load reliably in standard RSS readers (e.g., FreshRSS, NetNewsWire, Reeder, Feedbin).
4. Previously processed items never produce duplicate feed entries or trigger redundant LLM API calls on subsequent runs.