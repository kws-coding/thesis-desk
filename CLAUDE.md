# Thesis Desk: AI-assisted equity research tool

## What this is
A personal, single-user research tool. I give it an investment thesis, we refine it together, and it surfaces stocks that fit, with the full picture a serious investor wants before staking a position: bull and bear cases, fair value, entry prices, financial health, red flags, and ongoing monitoring with push alerts.
It is decision support only. It does not trade, does not connect to brokerages, is not investment advice, and is not a service for others. Every output should say it is a model result based on stated assumptions.
The first thesis I will use is "AI-driven efficiency becomes the norm," but theses are data, not hard-coded features. Any thesis must work.

## Core concept: the Thesis
A saved, editable record:
- claim in plain English and a time horizon
- screening criteria derived from it (structured, user-confirmed)
- thesis-fit rubric defining what a good fit means
- kill criteria: observable conditions that would prove it wrong
- status and review date

Thesis workshop: before anything runs, the LLM proposes criteria, a rubric, and kill criteria through conversation. I edit and approve. Nothing runs against the universe until I confirm.

## How a run works
1. Strategy or thesis -> structured criteria (validated Java record, shown to me for confirmation).
2. Candidate generation, two paths:
   - deterministic screener over a universe of about 300 US stocks
   - LLM brainstorm of additional names
   Every LLM-suggested ticker must be validated against the market data provider and must pass the same deterministic screen. Never trust an LLM-suggested ticker or fact without validation.
3. Deterministic scoring and valuation on all candidates.
4. Top 10 form the bucket. The LLM writes a dossier for each.
5. I promote 3 to the active watchlist. The other bucket names stay on cheap monitoring.
6. Alerts monitor the watchlist.
7. Every decision and the data I saw at the time go to the decision log.

## Non-negotiable architecture rules
1. All financial math (fair value, buy-below, scores, health metrics) is plain deterministic Java. The LLM never calculates numbers.
2. The LLM only reads text (filings, news) and writes summaries, cases, red flags, and the thesis-fit score against the rubric. Every claim must cite the supplied data. If the data does not support a claim, the answer is "not in the provided data."
3. Every analysis run stores provenance: thesis, criteria, input data snapshot, prompt, model, output, timestamp.
4. The decision log is append-only. Never update or delete its records. Store what was known at the time, not only current data, so later reviews avoid hindsight bias.
5. External providers sit behind Java interfaces: MarketDataProvider, PriceProvider, FilingsProvider, NewsProvider, MacroProvider. Cache raw responses in MongoDB with timestamps.
6. Secrets come from environment variables only. Never commit keys. Keep .env.example with blank placeholders.
7. Respect rate limits through a shared throttle. SEC EDGAR requests carry a descriptive User-Agent with a contact email and stay under about 9 requests per second.
8. LLM calls are cost-aware: log token usage per run, make the model and provider configurable, support a small-run mode (a few tickers) for development, and enforce a configurable per-run budget limit.
9. Spring AI changes quickly. Before writing any Spring AI code, check the documentation for the installed version. Do not rely on memory of older versions.

## Data sources
- Market data and fundamentals: Financial Modeling Prep (primary), Finnhub (backup for quotes and insider data). Free tiers are limited (FMP about 250 calls/day, end-of-day data), so cache aggressively and spread ingestion across days. Verify current limits in provider docs.
- Filings: SEC EDGAR directly (data.sec.gov submissions and XBRL companyfacts JSON, no key, 10 requests/second cap). Monitor by sweeping the daily index and diffing against already-ingested accession numbers. XBRL facts lag the filing, so read the filing text for new 8-Ks.
- News: Finnhub company news. Do our own materiality classification with the LLM instead of vendor sentiment.
- Macro: FRED (free key). Used as context and regime flags, not for reacting to every headline.
- Price alerts run post-close on free end-of-day data. PriceProvider is an interface so a real-time feed can be added later.
- Free tiers are personal-use. Never redistribute raw data or build anything public that displays it.

## Scoring (weights configurable per thesis)
- Valuation 40%: discount to fair value range, FCF yield, multiples versus own history and peers.
- Quality 25%: ROIC, margins, balance sheet strength, cash conversion.
- Thesis fit 20%: LLM scores against the thesis rubric and stores its reasoning. Least reliable component, so keep its weight modest.
- Red-flag penalty 15%: dilution, rising debt, customer concentration, restatements, heavy insider selling, auditor or going-concern language.

Fair value: conservative DCF, earnings-power value, and historical-multiple value, blended into a low/base/high range with stated assumptions (growth, margins, discount rate). Buy-below = low end of the range minus a configurable margin of safety (default 20%). Support entry tiers (for example "interesting at X, compelling at Y"). Show the percentage gap between current price and each tier.

## Candidate dossier (required fields)
- business summary
- thesis fit, with cited reasoning
- bull case and bear case with evidence. The bear case must address what is already priced in and how the thesis could backfire on this company. A candidate without both cases cannot enter the bucket.
- fair value range with bear/base/bull scenarios and assumptions
- buy-below and entry tiers
- financial health computed in code: leverage, interest coverage, liquidity, free cash flow and cash conversion, margin trend, share dilution, customer concentration where disclosed
- red flags and what changed in the latest filing's risk factors
- insider activity
- catalysts and dates (earnings, debt maturities, events)
- monitoring triggers and thesis kill criteria
No position sizing. That is my decision.

## Alerts (ntfy.sh)
Types: entry-tier crossing, big move up or down (daily and since added), valuation revised, new filing (8-K, 10-Q, 10-K, Form 4), material news, thesis kill criteria, weekly digest of the bucket.
- Alert rules live in MongoDB per ticker and thesis and are editable at runtime.
- Fire on threshold crossing, not on state. Per-rule cooldown and a hysteresis re-arm band.
- Priority maps to ntfy priority: kill criteria and entry tiers are high, filings default, news and digest low. Quiet hours hold non-urgent alerts.
- Message contains ticker, trigger, current price, buy-below, gap percent, one-line reason, link to the dossier. No position sizes or account data.
- After each new 10-Q or 10-K, recompute fair value and buy-below and alert if buy-below changes materially.
- Every alert is appended to the decision log with the data snapshot and, later, my action (acted, dismissed, ignored).
- ntfy topic name comes from config and must be a long random string. The public server does not password-protect topics, so never put sensitive data in messages.
- Alerts never place trades.

## Stack
Java, Spring Boot, Spring AI, MongoDB (via Docker Compose), scheduled jobs, ntfy.sh. Maven or Gradle, JUnit 5.

## Version 1 scope
In: thesis and strategy parser, data ingestion with point-in-time snapshots, valuation and scoring with unit tests, LLM dossiers with provenance, decision log, simple REST endpoints or CLI to view the bucket and promote picks, a small evaluation set (about 10 strategies with expected parsed criteria).
Next: ntfy alerts, scheduled filing and news monitoring, macro regime flags.
Later: web UI, backtesting the decision log.
Out of scope: trading, brokerage integration, multi-user, recommendations to others.

## Build order
1. Skeleton, Docker Compose for MongoDB, Spring AI hello world, ntfy test notification
2. Thesis and strategy parser (natural language to validated criteria record)
3. Data ingestion and MongoDB snapshots behind provider interfaces
4. Fair value, buy-below, scoring, and health metrics with unit tests
5. LLM dossier generation with citations and provenance storage
6. Watchlist, alert rules, scheduled monitoring, ntfy alerts
7. Evaluation set, README with architecture diagram and disclaimer

## Working agreements
- Always start in plan mode: propose a plan, wait for my approval before editing files.
- Work one build-order step at a time. Do not build ahead.
- Small steps. Write unit tests for all deterministic code before or alongside it.
- Commit after each working step with a clear message.
- Record each design decision and its reason in docs/decisions.md as a short entry.
- If a requirement is ambiguous or conflicts with these rules, ask instead of guessing.
- Explain non-obvious choices briefly so I can defend them in an interview.
