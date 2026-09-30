# Systems Case Studies

Sanitized architecture write-ups of production revenue/GTM and AI-systems work I've built. Client names, internal credentials, and business-sensitive specifics are withheld or generalized — the architecture, engineering decisions, and hard problems are real.

## Case studies

| System | What it does | Stack highlights |
|---|---|---|
| [Intent-Based Multi-Source Outbound Pipeline](./intent-based-outbound-pipeline/) | Scores companies on buying intent across 10 independent signal sources before spending enrichment budget on them | n8n, PostgREST, multi-source scraping, contact-resolution waterfall |
| [CRM Outbound Sync](./crm-outbound-sync/) | Bidirectional sync between CRM lead-qualification events and an outbound platform - reply events write back to CRM automatically | n8n, CRM webhook + REST API, LLM reply classification |
| [RB2B ICP Router](./rb2b-icp-router/) | Routes de-anonymized website visitors into the right outbound sequence via a two-factor intent x fit score | n8n, visitor de-anonymization, weighted scoring |
| [Guest-Post Prospecting Pipeline](./guest-post-prospecting-pipeline/) | Search-discovery fan-out, two-tier email resolution, verify-then-send with dual-mailbox rotation | n8n, SERP API, NeverBounce, LLM-personalized outreach |
| [White-Label Outreach Pipeline](./white-label-outreach-pipeline/) | Resumable scraper + enrichment + compliant cold-send pipeline with cryptographically-verified unsubscribe | Node.js, Cloudflare Workers, HMAC tokens |
| [Multi-Source Lead Sourcing & Enrichment Engine](./multi-source-lead-enrichment-engine/) | From-scratch Python system scraping 9 sources and resolving contacts through a 6-provider enrichment waterfall | Python, Postgres, anti-detection fetching, pytest |
| [RevFlow Client Systems](./revflow-client-systems/) | Independent consulting: CRM + automation infrastructure for local-service businesses, real named clients | GoHighLevel, voice agents, ad-platform integrations |
| [Natural-Language Ops Assistant](./natural-language-ops-assistant/) | Chat, email-triage, and scheduled-digest assistant over a project board — classifies intent before acting, drafts (never auto-sends) client-facing email | n8n, OpenAI (two-stage), Gmail, Slack |
| [AI Payment Reminders](./ai-payment-reminders/) | Escalating invoice-reminder sequence, idempotent on aging-bucket transitions rather than a fixed schedule (clean-room demo) | Python, stdlib only |
| [AI Operations Stack](./ai-operations-stack/) | CRM-integration layer around a third-party voice-agent product plus a human-approval-gated AI content pipeline, built twice for two businesses | n8n, CRM API, voice-AI platform API, LLM content generation |

## A pattern across all eight

The same idiom shows up independently in three different systems above: a **multi-source-waterfall** — try the best/cheapest option first, fall back through progressively different sources, and never let a later, worse answer overwrite an earlier, better one. I've built this pattern from scratch in both a low-code orchestration platform (n8n) and raw Python — it's a design instinct, not a tool-specific trick.

## What's not here

Real client names (beyond what clients have already made public themselves), production credentials, exact database contents, and internal business logic that would give away a competitive advantage to whoever's currently paying for these systems.
