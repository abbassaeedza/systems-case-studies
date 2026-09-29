# Multi-Source Lead Sourcing & Enrichment Engine

A from-scratch Python system that scrapes SMB business listings across nine sources and resolves owner contact information through a six-provider enrichment waterfall — real software, not a low-code workflow.

> Sanitized write-up. Real target-vertical specifics beyond the generic (home-services businesses), real API usage figures, and client-identifying details are withheld. Architecture, engineering decisions, and test discipline are real.

## Role / Context

- **Role**: sole engineer, from-scratch build.
- **Type**: employer project (sanitized).
- **Status**: live, tested (dozens of test files across scrapers and enrichers).

## The problem

Sourcing small-business leads at scale means scraping sites that actively try to block scrapers, then resolving a business listing (name, address, phone) into an actual decision-maker's contact info — where no single data provider has good coverage on its own. The system needed to do both reliably, resumably, and without getting the operator's infrastructure blacklisted by the sites it scrapes.

## Architecture

```
CLI (Click)
  -> 9 source-specific scrapers, each yielding leads as a generator (not a bulk list)
  -> Postgres — phone number (E.164-normalized) is the unique dedup key
  -> 6-provider enrichment waterfall:
       Provider A (owner + firmographics)
       -> Provider B (email + name)
       -> Provider C, via a scraping proxy (owner name, public-record fallback)
       -> Provider D (reverse-phone owner-name, last resort)
       -> Provider E (email fallback)
       + Provider F as a firmographic fallback
  -> CSV export
```

## What I built

- **Nine source-specific scrapers** behind one shared interface, each a generator rather than a function that returns a full list — so a scrape of thousands of listings never needs to hold them all in memory at once.
- **A resumable job system**: a job-tracking table records progress at the city × source × niche level, and a `--resume` flag skips every combination already completed — a multi-hour scrape interrupted at hour three restarts at hour three, not hour zero.
- **Cloud-only, anti-detection fetching by design**: every request is routed through a residential-proxy pool by default, with a stealth-proxy fallback if the primary proxy is unavailable — there is deliberately no code path that makes a direct request from the operator's own IP.
- **A six-provider enrichment waterfall with a hard non-destructive invariant**: every provider in the chain only writes a field if it's currently empty — a later, cheaper provider can never clobber a better answer an earlier provider already found. This invariant has an explicit test case per provider.
- **Automatic API-key rotation on rate-limit**: the primary enrichment provider auto-rotates to a secondary key on a 429/402/401 response, so a single provider's rate limit doesn't stall the whole run.
- **LLM-assisted query expansion**: a free-form search term gets expanded into morphological variants (e.g., a trade name and its plural/gerund forms) via an LLM call, each variant scraped separately and merged by phone-number dedup — with an explicit, documented fallback (no LLM key configured → exact-term search only, never a silent failure).

## Technical decisions

- **Fail closed on proxying, not fail open.** The system will not make a direct-IP request under any configuration — if no proxy is available, it falls back to a second, paid stealth-proxy provider rather than ever risking the operator's own IP getting blacklisted by a bot-protected site. A scraper that "just works" until it gets the whole operation IP-banned is a worse outcome than one that's slower but never exposes the source IP.
- **Non-destructive writes as a tested invariant, not a convention.** Enrichment providers run in a fixed, documented order specifically because a cheaper, lower-quality provider running after a better one has already answered must never be allowed to overwrite that answer — this is enforced with an explicit "does not overwrite" test per provider, not just documented and hoped for.
- **Dedup on phone number, not on source.** Nine overlapping sources will rediscover the same business repeatedly — deduplicating on a normalized phone number means overlapping sources merge into one record automatically instead of producing near-duplicate leads.

## Hard parts

- Getting the enrichment waterfall's *order* right required weighing data quality against cost-on-miss for each provider — a provider that's slightly worse but free-on-a-miss earns a place earlier in the chain than its raw data quality alone would suggest, because the chain's total cost matters as much as any single provider's accuracy.
- The LLM query-expansion step is a genuinely agentic component sitting inside an otherwise fully deterministic pipeline — it needed an explicit, tested fallback path so a missing API key degrades the system to "less thorough" rather than "silently broken," which is a different reliability bar than the rest of the pipeline's deterministic steps.
- Testing scrapers and enrichers without ever touching the real network or database in unit tests required a consistent contract per component (every scraper returns leads in a shared shape; every enricher has a documented five-case minimum: happy path, empty result, rate-limit, missing key, non-overwrite) — the discipline of writing that contract once paid for itself every time a new source got added.

## Reliability

- Resumable by design at the city × source × niche level — no wasted re-work after an interruption.
- Fails closed on network identity (no direct-IP path exists at all) and fails soft on rate limits (auto key-rotation) rather than stalling the run.
- The enrichment chain's non-destructive invariant is enforced by test, not convention.
- Unit tests never touch the network or the database — every scraper/enricher has a documented minimum test surface, checked in CI-style discipline before a new source ships.

## Results

Ran a self-initiated ~4,000-contact phone-verification pass against the scraper's output — nobody assigned this, it was a sanity check on whether the enrichment chain's owner-resolution actually held up against a real, unscripted phone test (no email enrichment was run on this batch; this pass was phone numbers only).

- **~4,000 SMB contacts** scraped and enriched for the test.
- **Live pickup rate ran about 2%** — in line with what cold-dialing a small-business owner's direct line typically looks like; most SMB owners don't pick up unknown numbers during work hours.
- **Live answers were overwhelmingly the actual business owner**, not a receptionist or front-desk gatekeeper — the outcome the enrichment chain's provider order (Apollo owner+firmographics → Hunter → State SOS owner fallback → IPQS reverse-phone owner-name-as-last-resort) was specifically designed to produce.
- **Calls that didn't connect went to voicemail, and the voicemail greeting confirmed the same thing**: the name on the greeting matched the resolved owner's name. So even the ~98% that didn't pick up still validated that the number belonged to the right person, not just a business's generic line.

The headline number here isn't the pickup rate — it's that a real, unscripted test confirmed the system resolves to the *owner*, which is the harder and more valuable problem than just finding *a* phone number for the business.

## Tech stack

Python · Click (CLI) · SQLAlchemy / Postgres · A residential-proxy pool with a stealth-proxy fallback · A six-provider contact-enrichment waterfall · An LLM for query expansion · pytest

## What this proves

This is from-scratch software engineering, not workflow assembly — a resumable job system, a tested non-destructive data-merge invariant, and a deliberate anti-detection architecture, built and tested the way a production Python service should be, in a domain (GTM tooling) where most solutions stop at "a script that works on the demo."
