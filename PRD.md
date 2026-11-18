# Personal Market Intelligence App

**Product Requirements Document**

> **Status:** Draft v0.1 | **Last updated:** November 17, 2026 | **Title:** working title, to be renamed

## Table of contents

1. [Overview](#1-overview)
2. [Problem](#2-problem)
3. [Goals, non-goals and success metrics](#3-goals-non-goals-and-success-metrics)
4. [User and context](#4-user-and-context)
5. [Design principles](#5-design-principles)
6. [Definitions](#6-definitions)
7. [Investor profile and onboarding](#7-investor-profile-and-onboarding)
8. [Home screen](#8-home-screen)
9. [Feature 1: Market picks](#9-feature-1-market-picks)
10. [Feature 2: Portfolio and actions](#10-feature-2-portfolio-and-actions)
11. [Feature 3: Ticker report](#11-feature-3-ticker-report)
12. [News intelligence](#12-news-intelligence-used-by-all-features)
13. [Scorecard](#13-scorecard)
14. [Non-functional requirements](#14-non-functional-requirements)
15. [Disclaimers and compliance](#15-disclaimers-and-compliance)
16. [Risks and mitigations](#16-risks-and-mitigations)
17. [Open questions](#17-open-questions)
18. [Proposed delivery phases](#18-proposed-delivery-phases)

Items marked **(Proposed)** are suggestions that still need your confirmation.

## 1. Overview

A private web app that watches the US public markets for one household. It suggests what to buy, hold or sell and why, tracks the household's portfolio down to the tax lot, produces a due-diligence report on any ticker, and keeps a running scorecard of how its own calls turned out. It works in desktop and mobile browsers and is reachable from anywhere.

The household plans to act on its suggestions with real money, so accuracy, and honesty about uncertainty, come before everything else.

## 2. Problem

- There is far more market news than the household can follow.
- The household has limited market and investment knowledge, so it can't easily judge a headline or a tip on its own.
- What's missing is one place that turns news, fundamentals, the actual holdings and tax impact into a clear action with reasons.

## 3. Goals, non-goals and success metrics

### Goals

1. Give clear, explained suggestions for 1-month, 6-month and 12+ month horizons.
2. Surface only the news that materially affects holdings or current picks.
3. Fit suggestions to the household's risk profile and to the portfolio's tax lots.
4. Keep a permanent, honest scorecard of every recommendation.
5. Stay usable on slow local AI: heavy analysis runs in the background and screens load from stored results.

### Non-goals (v1)

- Placing trades or linking brokerage accounts (portfolio entry is manual).
- Native mobile apps.
- Multiple users or profiles.
- Goals-based planning (retirement, education and so on).
- Options, futures, crypto and non-US-listed securities.
- Intraday or real-time trading signals.

### Success metrics

**Primary:** risk-adjusted performance of followed recommendations against the S&P 500 over the same periods, reported per horizon. When the metrics below conflict, this one wins.

**Supporting:**

- Directional accuracy at 1, 6 and 12 months.
- Downside capture: how the picks fall relative to the S&P 500 in down markets.
- Calibration: high-confidence calls should be right more often than low-confidence ones.
- Alert quality: few alerts, nearly all of them genuinely material.
- Operations: briefing ready on time, screens load fast.

Numeric targets will be set once there is baseline data. A 12-month call takes 12 months to resolve, so early numbers are interim and always show their sample size.

## 4. User and context

- One household, one login, one investor profile. Suggestions and the portfolio are household-wide.
- Used on phone and desktop browsers, from anywhere.
- The user is not a market expert, so suggestions must be decisive and short, with a brief reason and the basis for it.

## 5. Design principles

1. **Accuracy over everything.** When the evidence is weak, say so and suggest holding or doing nothing. Hold is a valid output and is scored like any other.
2. **The AI model never supplies numbers.** Prices, ratios and financials come from data sources, each with a source and as-of date. The model extracts, classifies, summarizes and explains. *(Proposed)*
3. **Every suggestion shows** its horizon, confidence, risks, what could go against it, and sources.
4. **Stable by default.** A recommendation changes only on material news or events, and the app says why it changed.
5. **Short explanations:** 2 to 3 lines of reasoning and the basis for it, not tutorials.
6. **Honest about uncertainty.** Projections are ranges with stated assumptions.

## 6. Definitions

| Term | Definition |
|---|---|
| Universe | US-listed stocks and ETFs, excluding penny stocks. |
| Penny stock | Price under $5, or average daily trading volume below a minimum threshold (value to be set). Excluded from picks. Can be held in the portfolio and get reports, always with a warning. |
| Volatile tag | Beta above 1.5 versus the S&P 500, or annualized volatility above about 40% over the trailing 12 months. Volatile stocks are allowed and always shown with the tag and the numbers behind it. |
| Horizons | Short = 1 month, Medium = 6 months, Long = 12+ months. |
| Report rating scale | Strong Buy, Buy, No Buy, Strong No Buy, one per horizon. |
| Portfolio action scale | Add, Hold, Trim, Sell, one per horizon. *(Proposed)* |
| Confidence | Low, Medium or High, checked against the scorecard over time. |
| Tax terms | A lot is long-term once held for more than one year, short-term before that. |

## 7. Investor profile and onboarding

- **ON-1:** First launch runs a scenario-based risk questionnaire. Examples: how would you react if the portfolio fell 20% in a month, or if a holding rose 40% in a few weeks. The result is a risk band (conservative, moderate or aggressive) saved as the household profile.
- **ON-2:** Goals are out of scope for v1 onboarding.
- **ON-3:** The profile can be retaken and edited. Each change is dated, and the scorecard records which profile was active for each recommendation.
- **ON-4:** The risk band affects which picks surface, suggested position sizes and confidence thresholds. Volatile-tagged picks are allowed but sized more conservatively for lower risk bands. *(Proposed)*

## 8. Home screen

- **HS-1:** The first screen shows the Daily Briefing and the Alerts section, plus a compact scorecard summary.
- **HS-2 Daily Briefing:** ready before the US market opens (exact time to be set). It holds what happened in the market and why it matters, fewer than 10 picks (Feature 1), portfolio changes since yesterday, news affecting holdings, and this week's upcoming events for holdings such as earnings dates. It is precomputed, so it is there the moment the app opens.
- **HS-3 Alerts:** material news on holdings or current picks, and important actions, for example "sell X" or "this pick is worth funding, sell Y". Each alert shows severity, what happened, the source, the suggested action and the time. Alerts stay until dismissed. In v1 alerts live inside the app only, with no email or push.
- **HS-4:** Only material events and changed actions create alerts.

## 9. Feature 1: Market picks

**Purpose:** stay on top of the market and know what is worth buying.

- **F1-01:** Fewer than 10 picks at a time, across stocks and ETFs, ranked by profile fit and confidence.
- **F1-02:** Each pick shows ticker, name, type and sector; horizon(s) and rating; reasons that support a buy (2 to 3 lines plus the basis); risk assessment (main risks, volatility tag, risk level); projected price at 6 and 12 months; confidence; suggested position size; sources and as-of dates.
- **F1-03:** Projections are a base case with a bear-to-bull range and the assumptions behind each, labelled as estimates. Every projection is logged and scored.
- **F1-04:** Quality gate for individual stocks *(Proposed, replaces the original "highly trustworthy" wording)*: consistent profitability, manageable debt, no accounting red flags, adequate liquidity and news coverage. To be confirmed.
- **F1-05:** Picks are sticky. A pick stays until its thesis breaks or its horizon passes. The briefing explains why anything was added or removed.
- **F1-06:** Position sizes respect the concentration limits in Feature 2.
- **F1-07:** When a pick is worth funding, the app says so proactively and proposes what to sell (see F2-07).
- **F1-08:** Selection steps: a rules-based fundamentals screen, then AI analysis of news and qualitative factors, then ranking by profile fit.

## 10. Feature 2: Portfolio and actions

- **F2-01 Manual entry:** each ticker can have several lots. A lot has a purchase date, number of shares and cost per share. Lots can be edited and deleted, and sales can be recorded (lot-selection method to be decided).
- **F2-02 Tracking:** current price, value, gain or loss per lot and per position, share of portfolio, sector allocation and volatile tags.
- **F2-03 Suggested action** per ticker for short, medium and long horizons, each with the action, the reason, what could go against it, confidence and sources.
- **F2-04:** Actions are sticky. They change on material events only, and the change is explained.
- **F2-05 Tax context per lot:** short-term or long-term, days until it turns long-term, gain or loss, and warnings such as wash-sale risk when selling at a loss and rebuying within 30 days. This is classification and estimates only, with a disclaimer. Dollar tax estimates appear only if the user enters a tax rate (open question).
- **F2-06 Concentration limits** *(Proposed defaults, adjustable)*: 10% in any single stock, 30% in any sector. The app warns when a limit is exceeded and flags overlap between ETFs and individual holdings.
- **F2-07 Funding suggestions:** when a pick is worth funding, or the user says they want to buy Y for $Z, the app proposes what to sell. It weighs, in this order: (1) weakest outlook, (2) tax cost, (3) overweight positions. It shows the portfolio after the trade, the tax effect per lot and any warnings. It does this proactively, not only on request.
- **F2-08:** Penny stocks can be held and show a warning.

## 11. Feature 3: Ticker report

- **F3-01 Input:** one ticker. Stocks, ETFs and penny stocks (with a warning) are allowed. Unknown or non-US tickers get a clear error.
- **F3-02 Rating:** one per horizon (1 month, 6 months, 12+ months): Strong Buy, Buy, No Buy or Strong No Buy, each with confidence.
- **F3-03 Sections:**
  1. Recommendation and reasoning
  2. Facts that support a buy
  3. Facts that support a no-buy
  4. Fundamentals: valuation versus peers and its own history; balance sheet and debt; cash flow and earnings quality; growth and demand; competitors and moat
  5. Market view: recent news, analyst consensus and price targets, insider and institutional activity, sentiment
  6. 52-week view: price chart with the 52-week high and low, where the price sits in that range, volume, major events marked, and trend context
  7. Upcoming catalysts
  8. Risks and what could go wrong
  9. What changed since the last report
  10. Sources and as-of dates, plus warnings (penny stock, volatile, low liquidity)
- **F3-04 ETFs:** the same sections where they apply, plus top 10 holdings with weights, fees, sector breakdown and overlap with the household's portfolio.
- **F3-05:** Reports generate in the background with a progress indicator, and the user can leave and come back. A fresh cached report opens instantly, with a regenerate option. Reports are saved per ticker with history.
- **F3-06:** Report ratings are logged to the scorecard.

## 12. News intelligence (used by all features)

- **NP-1:** Monitored: holdings, current picks, and broad market and macro news.
- **NP-2 Sources, in this order:** primary sources first (SEC filings, company press releases, earnings calls), then major financial outlets. Social media and rumors are ignored unless confirmed by a primary or major source. Free, credible sources only.
- **NP-3 Material events:** earnings, guidance changes, M&A, lawsuits or regulatory action, executive changes, and macro events (the Fed, rates) that hit a holding's sector.
- **NP-4:** Unconfirmed single-source claims do not trigger changes. Duplicate stories are merged.
- **NP-5:** A material event triggers re-analysis of the affected ticker. If the recommendation changes, an alert explains what changed and why.
- **NP-6 Efficiency:** process only new items, cache results, and run heavy work in the background.
- **NP-7:** Follow each source's terms of use. Store extracted facts, short summaries and links, not full copyrighted articles.

## 13. Scorecard

- **TR-1:** Log every recommendation: picks, portfolio actions and report ratings, including holds and removed picks. Record date, ticker, price, horizon, rating or action, confidence, projected range, the risk band at the time, and the model, prompt and rule versions with a snapshot of the inputs.
- **TR-2:** At the end of each horizon, compare the outcome with the S&P 500 over the same window.
- **TR-3 Metrics:** risk-adjusted performance versus the S&P 500 (primary), directional accuracy, downside capture, and calibration by confidence level.
- **TR-4:** Always show sample size, and show "not enough data yet" until enough calls have resolved.
- **TR-5:** When the model, prompts or rules change, report results separately by version.
- **TR-6:** A summary sits on the home screen and the full view in the main navigation.

> [!WARNING]
> Do not backtest the AI model on past dates to claim accuracy, because it may already know what happened. Accuracy counts only for calls logged going forward.

## 14. Non-functional requirements

### Platform and performance

- Responsive web for phone, tablet and desktop browsers.
- The AI runs locally on Gemma 4 31B and will be slow. The UI never waits on the model: precompute daily results, queue background jobs, cache, and show progress. The model sits behind an interface so it can be swapped later.
- Screens load from stored results. Targets are set after benchmarking on the real hardware. Starting goals: stored screens load in about 2 seconds, new reports finish in under 10 minutes.

### Data

- Free, authentic, credible sources only, for example SEC EDGAR, company investor pages, Federal Reserve and Treasury data, free price data and major financial outlets. Check each source's rate limits and license terms before committing.
- Every number shows its source and as-of time. Missing or stale data is flagged, never guessed.
- A verification step rejects any figure in the model's output that is not in the source data.

### Security and access

- Reachable from anywhere over HTTPS. Login required, strong authentication (passkey or 2FA recommended), session timeouts.
- Portfolio, lot and profile data encrypted in transit and at rest, with encrypted backups. No brokerage credentials are stored.
- Export and delete of all data. Audit log of data changes.

### Reliability

- The daily briefing is ready by its set time on nearly all days. If a run fails, the app shows the last good briefing and a visible status, never a blank screen.
- The app keeps working when the local machine is off (see [open questions](#17-open-questions)).

## 15. Disclaimers and compliance

> [!NOTE]
> Informational only, not financial, tax or legal advice. Estimates and projections can be wrong.

- Tax output is classification and estimates. Confirm with a tax professional.
- Family use only. If it is ever shared beyond the family, get legal advice first, because personalized investment advice to others can trigger registration rules.

## 16. Risks and mitigations

| # | Risk | Mitigation |
|---|---|---|
| 1 | **No real edge.** Beating a plain index fund is hard, even for professionals. | The S&P 500 is the benchmark for everything, hold is a valid output, and the scorecard sits on the home screen so a weak record is visible. |
| 2 | **Local model limits.** A mid-size local model may be slower and less reliable at financial reasoning than large hosted models. | Use it for extraction, classification and explanation only. Take numbers and scores from data and rules. Build an evaluation set from past material events. Keep the model swappable. |
| 3 | **Made-up facts.** | Structured data only, source and date on every figure, and a verification step. |
| 4 | **Gaps in free data.** | Source tiers, stale-data flags, and accepting lower coverage over guessing. |
| 5 | **Churn from daily re-analysis.** | Materiality threshold and sticky recommendations. |
| 6 | **Overconfidence with no paper-trading phase.** | Concentration limits, confidence labels, and the scorecard on the home screen. |
| 7 | **Local compute versus access from anywhere.** | Decide hosting early and fall back to the last good briefing. |
| 8 | **Tax logic errors.** | Classify and estimate only, disclaimers, and tests on holding-period and wash-sale edge cases. |
| 9 | **Misleading backtests.** | Judge accuracy only on calls logged going forward. |

## 17. Open questions

- [ ] **Hosting:** the AI runs locally but the app must be reachable from anywhere. Where do the app and database live, and what happens when the local machine is off or asleep?
- [ ] What time should the daily briefing be ready?
- [ ] Are the proposed concentration limits (10% per stock, 30% per sector) right?
- [ ] Should the portfolio track cash? It affects funding suggestions.
- [ ] When a position is sold, which lots are used: oldest first, or chosen by the user?
- [ ] Are some holdings in retirement accounts? Tax logic applies only to taxable accounts, so lots may need an account-type field.
- [ ] Should the app estimate tax in dollars (needs a tax rate) or only classify gains and warn?
- [ ] With the universe open to all non-penny stocks, confirm the quality gate in F1-04.
- [ ] What is the minimum average daily volume for the penny-stock rule?
- [ ] Is the scope US-listed securities only?
- [ ] What mix of ETFs and stocks should the daily picks have?

## 18. Proposed delivery phases

1. **Foundations:** free data ingestion and a source registry, onboarding and risk profile, portfolio with tax lots, scorecard logging structure.
2. **Research:** Feature 3 reports, news pipeline and alerts.
3. **Guidance:** daily briefing and Feature 1 picks, Feature 2 actions, funding suggestions and tax context.
4. **Hardening:** security review, performance tuning on the real hardware, full scorecard views.
