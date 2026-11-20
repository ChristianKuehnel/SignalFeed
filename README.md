# SignalFeed

SignalFeed is a lightweight, self-hosted service that turns noisy news streams into focused, readable RSS feeds. Configured via a single sources.yaml, it fetches RSS/Atom feeds (and optional web sources or full texts), applies rule-based pre-filters and deduplication, evaluates relevance using LLMs (including optional triage/decision models), summarizes relevant items, and publishes them as standard RSS feeds for any reader.

Key Features:

YAML-driven configuration: Define feeds, target outputs, regex pre-filters, and source-specific LLM prompts in one place.

Noise & cost reduction: Deduplicates seen items, skips paywalled/unwanted headlines upfront, and optionally pairs a fast triage/decision step with a stronger summarization model.

Reader-native output: Serves clean, curated RSS/Atom XML feeds ready for any standard RSS reader—read on your own schedule.
