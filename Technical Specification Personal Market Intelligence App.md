# Technical Specification: Personal Market Intelligence App

Draft v0.1 | November 17, 2026 | Companion to the PRD (v0.1)

## 0. About this document

This spec says how to build what the PRD describes, using only free and open-source software and free data sources. It is written so the system can be built in small, testable pieces, including with a local coding model.

Conventions:

- PRD requirement IDs (F1-03, NP-5, TR-1 and so on) are cited so every technical choice traces back to a need.
- &#91;DECISION NEEDED\] marks a choice only you can make. \[ASSUMPTION\] marks something I assumed. \[VERIFY\] marks a fact about an outside service (limits, terms, URLs) that changes, so confirm it in the first week of the project.
- Version numbers are deliberately not pinned. Use the current stable release when the project starts and pin it in the lockfile.

## 1. Technical goals and constraints

Goals:

1. Free and open source only. Every library, runtime, database and model is open source, and every data source is free. No paid APIs, and no cloud AI by default.
2. Swap the LLM in minutes with no code change and no redeploy of the app: one config block plus an automated check.
3. Built for slow AI. The UI never waits on the model. Everything the home screen shows is precomputed.
4. Traceable and reproducible. Every number has a source and an as-of date. Every recommendation records the data, rules, prompt and model that produced it (PRD TR-1).
5. Small to operate. One household, one user, one machine to start. Fewer moving parts beat clever ones.
6. Secure, and reachable from anywhere (PRD section 14).

Constraints:

- Default model is Gemma 4 31B, which Google released under Apache 2.0 (confirm on the model card before relying on it). The Qwen model you mentioned is for coding, not for the app.
- Free data means rate limits, delays and gaps (section 5). The system must degrade gracefully and never guess.
- Hardware is unknown \[DECISION NEEDED, see 2.5\].
- The app is informational only (PRD section 15). No trade execution, and no brokerage credentials.

## 2. Architecture

### 2.1 Shape of the system

A modular monolith: one codebase, a few processes. No microservices, because one household and one operator don't need them.

```
                      +--------------------------+
  Phone / laptop ---> |  Reverse proxy (Caddy)   |  HTTPS + access control
    (browser)         +------------+-------------+
                                   |
               +-------------------+-------------------+
               |                                       |
        +------v-------+    REST + SSE         +------v-------+
        |  Web app     | -------------------->  |  API service |
        |  (React SPA) |                        |  (FastAPI)   |
        +--------------+                        +------+-------+
                                                       |
                 +-------------------------------------+
                 |                                     |
          +------v-------+   jobs via Postgres   +------v-------+
          |  PostgreSQL  | <-------------------> |   Workers    |
          |  + pgvector  |                       |  ingest      |
          +--------------+                       |  analysis    |
                                                 |  llm         |
                                                 +--+--------+--+
                                                    |        |
                                +-------------------+        +------------------+
                                |                                               |
                       +--------v---------+                         +-----------v-----------+
                       |   LLM server     |                         |  Free data sources    |
                       | Ollama / llama   |                         |  SEC EDGAR, Alpaca,   |
                       | .cpp / vLLM      |                         |  FRED, RSS feeds ...  |
                       | (OpenAI-compat)  |                         +-----------------------+
                       +------------------+
```

### 2.2 Components

- Web app: React single-page app, mobile-first. It talks only to the API.
- API service (FastAPI): serves stored results, accepts portfolio input, enqueues report jobs and streams their progress. It never calls the LLM or an external data source while handling a normal page request.
- Workers: the same codebase run as separate processes by queue. The `ingest` queue handles data and news, `analysis` handles deterministic scoring, portfolio and scorecard work, and `llm` handles every model call. Separate queues mean slow model work never blocks data ingestion.
- Scheduler: periodic tasks defined in code and run by the queue library. No separate cron.
- PostgreSQL: the only datastore. It holds market data, the job queue, the recommendation log and the scorecard. pgvector stores embeddings.
- LLM server: a separate process exposing an OpenAI-compatible HTTP API. The app knows only its URL and model name.
- Reverse proxy: TLS, security headers, rate limiting.
- Backup job: encrypted nightly dump (section 10).

### 2.3 The daily cycle

Times are US Eastern (market time) and configurable. The briefing deadline is \[DECISION NEEDED\] (PRD open question 2).

1. After the close (about 5:30 pm): ingest end-of-day prices for the universe, then run the corporate-action check.
2. Evening: ingest the day's SEC filings and refresh fundamentals for companies that filed.
3. Night: compute flags (penny, volatile) and factor scores for the whole universe. This is deterministic and takes minutes.
4. Night: for the shortlist (top candidates plus all holdings), run LLM analysis, build recommendations and apply the stickiness rules.
5. Early morning: build the briefing and alerts and write the home snapshot.
6. Continuously (every 15 minutes around market hours, hourly otherwise): fetch news, classify it, score materiality, and queue re-analysis for affected tickers.
7. Daily: mark every logged recommendation to market and resolve the ones whose horizon has ended.

### 2.4 Deployment topologies \[DECISION NEEDED\]

Both use the same Docker Compose files. The difference is where each service runs.

- Topology A, single box: everything runs on the one machine with the GPU. Simplest. If that machine is off, the app is off.
- Topology B, split: the app, API and Postgres run on a small always-on host (a mini PC at home, or a free-tier VPS). The GPU machine runs only the LLM server and the `llm` worker, joined to the app host by a private WireGuard link. When the GPU machine is off, the app still serves the last good briefing and LLM jobs wait in the queue.

My recommendation: A if the GPU machine can stay on 24/7, otherwise B. One rule makes both work: workers connect out to Postgres, and Postgres is never exposed to the internet.

### 2.5 Hardware assumptions 

- Apple Silicon M5 pro with 48 GB or more of unified memory, for a 31B dense model at 4-bit quantization. My estimate is roughly 17 to 20 GB for the weights alone, before the context cache.
- 48 GB of system RAM and a 1 TB SSD.
- Data volume is small. About 10 years of daily bars for roughly 6,000 tickers is on the order of 15 million rows, which one Postgres instance handles easily.

Tell me your real hardware. The throughput plan in 4.13 depends on it.

### 2.6 Key decisions

- ADR-1, Python backend, TypeScript frontend. Finance, data and LLM libraries are Python-first. The API contract (OpenAPI) keeps the boundary clean, so a different backend language stays possible, but you would be rebuilding data-science plumbing.
- ADR-2, PostgreSQL only: data, job queue and vectors in one database. One thing to back up and one thing to learn.
- ADR-3, modular monolith, not microservices.
- ADR-4, every model is reached through one OpenAI-compatible interface (section 4).
- ADR-5, precompute everything. The request path reads stored results only.
- ADR-6, the model never supplies numbers (PRD principle 2). Numbers come from data and code, and the model writes around them.
- ADR-7, the recommendation log is append-only. A recommendation is superseded, never edited, so the scorecard stays honest.
- ADR-8, configuration as code. Models, sources, rating weights and limits live in versioned YAML validated at startup.

## 3. Technology stack

Backend

| Layer	| Pick | Why
| --- | --- | --- |
| Language	| Python 3.13	| edgartools (MIT) already parses financials, 8-K items, Form 4 insiders, 13F and N-PORT fund holdings. That covers most of the PRD’s data needs. |
| API	| FastAPI with Pydantic v2	| Generates an OpenAPI spec, so the frontend client is typed automatically. |
| Database	| PostgreSQL 18	| --- |
| DB access	| SQLAlchemy 2, Alembic, psycopg 3	| One driver, shared with the job queue. |
| Jobs and scheduling	| Procrastinate	| Runs on Postgres, so no Redis. Has periodic tasks, priorities and retries. It’s actively released, but the project is asking for maintainers. |
| Number crunching	| pandas and NumPy	| edgartools returns pandas, and the data is small, so a second dataframe library isn’t worth it. |
| HTTP, retries, rate limits	| httpx, tenacity, aiolimiter	| Small and standard. |
| Feeds, calendar, prompts	| feedparser, exchange_calendars, Jinja2	| The calendar handles NYSE holidays and trading days. |
| Tooling	| uv, ruff, pyright	|

Data sources

Need	Pick	Why
Filings, fundamentals, insiders, ETF holdings	SEC EDGAR via edgartools	Official, with a limit of 10 requests per second.
Prices and the S&P proxy (SPY)	Alpaca Basic, called with plain httpx	The only free option with multi-symbol batch bars at a workable rate.
Macro	FRED	Free key.
News	SEC, Fed and Treasury feeds, then publisher RSS via feedparser	I’ll choose the RSS publishers in the first spike.

On Alpaca, the free default feed is IEX, which is about 2.5% of market volume. You must request feed=sip explicitly. Its docs say historical SIP queries work without a subscription if the end time is at least 15 minutes old, but I found conflicting statements, so test this in week one.

LLM layer

Layer	Pick	Why
Server	Ollama	Fastest model swapping (ollama pull, change a name). Supports JSON-schema output and thinking control over its OpenAI-compatible API.
Client	OpenAI Python SDK	Only the base URL changes between servers.
Default model	Gemma 4 31B (gemma4:31b)	Apache 2.0, 256K context.

If you ever need more control, llama.cpp is the escape hatch, with no app changes.

Frontend

Layer	Pick
Framework	React, TypeScript, Vite (single-page app)
Routing and server state	React Router, TanStack Query (no Redux)
UI and forms	Tailwind, shadcn/ui, React Hook Form with Zod
API client	openapi-typescript with openapi-fetch
Charts	Lightweight Charts v5 only. Allocation bars are plain CSS. It’s Apache-2.0 but requires a TradingView attribution link.
Lint and format	Biome (replaces ESLint and Prettier)
Packages	pnpm

Ops, security, quality

Layer	Pick
Remote access	Tailscale, free Personal plan
Proxy and TLS	Caddy
Login	Passkeys (WebAuthn) plus recovery codes
Containers	Docker Compose
Backups	restic (encrypts by itself) on a nightly pg_dump
Logging	structlog JSON, plus the admin status page
Tests	pytest, hypothesis, respx, Vitest, Playwright, and an in-house LLM eval harness
CI and scanning	GitHub Actions, OSV-Scanner

Dependency policy:

- Lockfiles with hashes for Python and npm. Updates are reviewed, not automatic, and a brand-new release waits a couple of weeks before you adopt it.
- A CI license scan with an allowlist. Copyleft licenses are fine for a private family app, but you should choose them knowingly.
- Keep the dependency list short. Every library is something you must keep patched.
- OpenBB's data platform (Apache-2.0 from v5) can be wrapped as one optional data provider behind the adapter interface in section 5. It is not a core dependency.

## 4. The LLM layer

This is the part built for your 'change the model quickly' requirement.

### 4.1 Design principles

1. One interface. The app talks to every model through one OpenAI-compatible chat API. Ollama, llama.cpp and vLLM all expose it, so swapping a model means changing two values: the server URL and the model name.
2. Roles, not model names. Code asks for a role (extractor, analyst, writer, judge, embedder). Config maps roles to models. No model name appears in application code.
3. Tasks own their prompts and output schemas. Each LLM task has a versioned prompt template and a schema. Model-specific quirks live in small override files, not in task code.
4. Every swap is tested before it goes live, by automated evals and shadow runs.
5. Every output is attributable. Each call records the model, prompt version and inputs, so the scorecard can compare models (PRD TR-5).
6. Reversible. Rollback is one command, and the old model stays registered.

### 4.2 Layers

```
  pipelines / API handlers
          |   run_task('news_materiality', input)
          v
  +--------------------+   reads   +-------------------------+
  |    LLM Gateway     | --------> |  config/llm.yaml        |
  |  role -> model     |           |  prompts/<task>/...     |
  |  build prompt      |           +-------------------------+
  |  structured output |
  |  validate + retry  |   logs    +-------------------------+
  |  fallback policy   | --------> |  llm_call table         |
  +---------+----------+           +-------------------------+
            |  OpenAI-compatible /v1
            v
    Ollama | llama.cpp | vLLM | any other compatible server
```

The gateway is one Python package (`llm/`), about a few hundred lines. LiteLLM is optional. If you want its fallbacks and routing you can run it as a proxy and point `base_url` at it, but the app works without it.

### 4.3 Roles

- extractor: news classification, entity linking, event extraction, facts from filing text. Many small calls, thinking off, temperature 0.
- analyst: the qualitative overlay on a ticker, risk narrative, 'what could go against it'. Fewer, larger calls. Thinking on where it measurably helps.
- writer: turns structured results into briefing and report prose, with number placeholders (4.14).
- judge: checks writer and analyst output against the source facts. Deterministic checks run first, and the judge only handles what they can't.
- embedder: embeddings for news de-duplication and retrieval. A separate small model with its own profile.

All roles can point at one model, and in version 1 they will.

### 4.4 Model registry (config/llm.yaml)

```yaml
version: 1
defaults:
  timeout_s: 300
  max_retries: 2
  temperature: 0.1

models:
  gemma4-31b:
    provider: openai_compat          # any server that exposes /v1/chat/completions
    base_url: http://localhost:11434/v1   # Ollama. llama.cpp is usually :8080/v1, vLLM :8000/v1
    model: gemma4:31b                # the name your server uses (placeholder, check the real tag)
    api_key_env: LLM_LOCAL_KEY       # may be empty for a local server
    family: gemma4                   # selects prompt overrides
    context_window: 32768            # what we actually run with, not the model maximum
    max_output_tokens: 4096
    concurrency: 1                   # match your server's parallel slots
    allow_remote: false              # local only. Private data refuses remote models
    capabilities:
      json_schema: true
      system_role: true
      thinking_control: true
    request_overrides:               # merged into the request body (check your runtime's docs)
      think: false

  qwen-27b:                          # example of a second model, a 6-line change
    provider: openai_compat
    base_url: http://localhost:11434/v1
    model: qwen:27b                  # placeholder
    family: qwen
    context_window: 32768
    concurrency: 1
    allow_remote: false
    capabilities: { json_schema: true, system_role: true, thinking_control: false }

  embed-small:
    provider: openai_compat
    base_url: http://localhost:11434/v1
    model: nomic-embed-text          # placeholder embedding model
    family: embed
    allow_remote: false

roles:
  extractor: gemma4-31b
  analyst: gemma4-31b
  writer: gemma4-31b
  judge: gemma4-31b
  embedder: embed-small

fallbacks:
  analyst: []      # empty means fail and flag. Never switch models silently

task_overrides:
  news_materiality: { temperature: 0, thinking: off }
  ticker_overlay:   { temperature: 0.2, thinking: on }
```

The file is validated against a Pydantic schema at startup. A bad config refuses to load and the previous good one stays active.

### 4.5 Tasks and prompts

Each LLM job is a task. A task declares its role, prompt version, input model, output model and validators.

```python
@dataclass(frozen=True)
class TaskSpec(Generic[In, Out]):
    name: str                 # 'news_materiality'
    role: Role                # extractor | analyst | writer | judge
    prompt_version: str       # '3'
    input_model: type[In]
    output_model: type[Out]   # Pydantic model, also used as the JSON schema
    data_class: str           # 'public' | 'private' (private never goes to remote models)
    max_input_tokens: int
    validators: list[Callable[[In, Out], None]] = ()   # e.g. number_guard

result = await gateway.run(NEWS_MATERIALITY, NewsItemIn(...), ctx=run_ctx)
```

Prompt files are versioned and separate from code:

```
prompts/
  news_materiality/
    v3.system.j2
    v3.user.j2
    overrides/
      gemma4.system.j2     # optional, used when the model family is gemma4
      qwen.system.j2
```

Model differences (instruction style, how to switch thinking off, system-role handling) go in overrides. The task code never changes when you change models.

### 4.6 Structured output

The gateway tries these in order, based on the model's capability flags:

1. Native schema-constrained output (`response_format` with a JSON schema, or the runtime's grammar feature). Most reliable.
2. JSON mode with the schema described in the prompt.
3. Plain text parsed by a fallback extractor.

Every output is validated with Pydantic. On failure the gateway retries once with the validation error shown to the model. If that fails too, the task fails, the failure is logged, and the pipeline degrades visibly (a 'data unavailable' flag) rather than guessing.

### 4.7 Capability flags and quirks

A model block declares what the model and server can do: `json_schema`, `system_role`, `thinking_control`, `tools` (unused in v1). `invest llm test` probes these automatically and warns if config and reality disagree. If a model has no system role, the gateway folds the system prompt into the first user message. This is the kind of quirk that otherwise breaks swaps.

### 4.8 Serving runtimes

All expose an OpenAI-compatible API, so switching between them is a `base_url` change.

- Ollama: easiest. `ollama pull` a model, name it in config. Loads and unloads models on demand. Good default for fast experimentation. Tune parallel slots and keep-alive through its environment settings.
- llama.cpp (`llama-server`): most control, strong grammar-constrained output and prefix caching, runs on the widest range of hardware. Pick it when you want tight control or a CPU/GPU split.
- vLLM: highest throughput through continuous batching. Pick it when you have enough VRAM and want to run many tasks in parallel. Usually needs a quantized checkpoint for a 31B model on one consumer GPU.

Start with Ollama. Move only if the benchmark in 4.13 says you need to.

### 4.9 How to swap a model

The five-minute path:

1. Serve the new model (for example `ollama pull <model>`, or start llama.cpp or vLLM with it).
2. Copy an existing block under `models:` in `config/llm.yaml` and change the name, URL and flags (6 to 10 lines).
3. `invest llm test <profile>`: confirms the server is reachable, the model answers, JSON schema and system-role work, and measures speed.
4. `invest llm eval <profile>`: runs the eval suite (4.10) and compares it with the current model and the promotion gates.
5. Optional: `invest llm shadow <profile> --role analyst --days 7` runs it alongside the live model with no user-facing effect (4.11).
6. `invest llm use analyst <profile>`: validates, writes the config and hot-reloads it. No app redeploy.

Rollback is `invest llm rollback analyst`. An admin page in the app shows the same state: which model serves each role, last test, last eval. Changing a role can be done from there too.

Three extra rules:

- You can point a role at a hosted OpenAI-compatible endpoint, but it is off by default. Tasks marked `data_class: private` (anything involving your portfolio) are blocked from models with `allow_remote: false`.
- Changing the embedding model requires re-embedding stored vectors. Each vector stores its model name, and a reindex job handles the switch.
- After a swap, existing recommendations stay valid until their normal re-evaluation. This avoids a flood of flip-flops caused only by the model change (PRD principle 4).

### 4.10 Evals and promotion gates

The eval harness (`evals/`, run with `invest llm eval`) is what makes swapping safe.

- Suites: news materiality classification, entity linking, fact extraction from filing text, narrative faithfulness (numbers and citations), schema validity across all tasks, and speed at target concurrency.
- Labels come from public ground truth where possible. For example, 8-K filings carry item codes (earnings results, officer departures, material agreements, restatement notices), which give free labels for 'was this material' without you hand-labeling thousands of items. \[VERIFY\]
- Gates live in `config/eval_gates.yaml`: absolute floors (for example schema validity at least 99%, zero numeric-faithfulness violations) and relative limits (no worse than the current model by more than a set margin). A model that misses a gate can't be promoted unless you override it, and the override is logged.
- Judging with an LLM is a last resort, because a model grading its own output is biased. Deterministic checks come first.

### 4.11 Shadow mode and rollback

Shadow mode runs a candidate model on live inputs in parallel and stores its outputs with `is_shadow = true`. Nothing shows in the app. After a few days you compare agreement on ratings, materiality calls and risk flags, plus speed and failures. Disagreements are listed for review. Promotion and rollback are both a one-line config change recorded in the audit log.

### 4.12 Versioning and attribution

Every LLM call writes an `llm_call` row: task, role, model profile, model name as served, `model_epoch`, prompt version, input hash, token counts, latency, schema-valid flag, retries and error. Every recommendation links to the calls that produced it plus an input snapshot. `model_epoch` increments whenever any role's model changes. The scorecard (PRD TR-5) reports results by epoch, so you can see whether a new model actually performed better.

### 4.13 Performance on slow local inference

- Queues and priorities: the `llm` queue's worker concurrency equals the server's parallel slots. Interactive report requests run ahead of alert re-analysis, which runs ahead of the nightly batch.
- Keep prompts small. Gemma 4 supports a 256K context, but run at 16K to 32K for speed and VRAM. Retrieve relevant snippets rather than pasting everything.
- Put static instructions first and variable data last, so servers with prefix caching (llama.cpp, vLLM) reuse the shared start.
- Thinking mode only where it pays. Classification and extraction run with thinking off.
- Result cache keyed by hash of task, prompt version, model and inputs. Unchanged inputs never re-run.
- Only the shortlist reaches the model. The universe screen is deterministic code, so the LLM sees tens of tickers a night, not thousands.
- Budget formula: nightly model time is roughly (number of calls) x (prefill time + output tokens divided by tokens per second). For example, 40 candidates at about 800 output tokens each is a few tens of minutes at typical local speeds, which fits overnight. \[ASSUMPTION\] The real numbers come from `invest llm bench` on your hardware in the first spike (section 15).
- Timeouts, backoff and a circuit breaker per server. If the model is down, jobs wait and the app serves the last good data with a staleness banner.

### 4.14 Safety: injection and numbers

News and filings are untrusted text.

- The model gets no tools and no network access, and it never writes to the database. Code validates its output and writes it.
- Untrusted text goes inside clearly delimited blocks, with instructions saying it is data. Inputs are stripped of HTML and length-capped. Outputs are schema-constrained, so injected instructions have no channel to act through.
- Number guard (ADR-6): the writer receives a dictionary of facts with IDs and must reference them as placeholders such as `{{fact:AAPL.pe_ttm}}`. The renderer substitutes values from the database. A validator rejects any numeric literal outside the placeholders, apart from an allowlist (years, quarter labels, form names). The model physically cannot invent a price or a ratio.
- Claims need evidence: every reason in a recommendation cites news item, filing or fact IDs. Claims with no evidence are dropped before display.

### 4.15 Default configuration and first steps

- All roles point at Gemma 4 31B, 4-bit quantized, served by Ollama, with the embedder on a small separate model.
- Run `invest llm bench` first. Record tokens per second, prefill time and the safe concurrency level, and put the results in `docs/benchmarks.md`.
- Build the eval suites before you build the features that depend on them. Weeks later, when a better open model appears, the swap will take an afternoon instead of a rewrite.

## 5. Data layer

### 5.1 Adapters and the source registry

Every kind of data sits behind a small interface, the same pattern as the LLM registry. Adding or replacing a source is one adapter file plus one config entry.

```python
class PriceProvider(Protocol):
    id: str
    async def daily_bars(self, symbols: list[str], start: date, end: date) -> list[Bar]: ...

class FilingsProvider(Protocol):
    id: str
    async def filings_since(self, since: datetime) -> list[Filing]: ...

class NewsSource(Protocol):
    id: str
    tier: int          # 1 = primary (SEC, company, Fed), 2 = major outlet, 3 = other
    async def fetch_new(self) -> list[RawNewsItem]: ...
```

Adapters return validated Pydantic models, never raw JSON. `config/sources.yaml` lists each source: id, type, enabled, priority within its type, rate limit, tier, credential variable, terms URL and notes. For each data type, the first enabled provider that returns fresh, valid data wins, and the result carries `source_id` and `as_of`. Raw responses are archived for 60 days so any number can be replayed and debugged.

### 5.2 What comes from where

All free. \[VERIFY\] Free-tier limits and terms change often, so confirm each one before you build on it.

| Need | Primary | Fallbacks | Notes |
| --- | --- | --- | --- |
| Company identity, CIK, SIC code | SEC company\_tickers.json and submissions API | Provider asset lists | Sector comes from SIC codes mapped to a simple sector list you maintain. GICS is proprietary. |
| Daily prices (OHLCV, adjusted) | Alpaca Basic (free account, IEX feed, 200 historical calls per minute, latest 15 minutes excluded) | Tiingo end-of-day free tier, Twelve Data (8 per minute, 800 per day), yfinance (unofficial, prototyping only) | The daily cycle only needs end-of-day bars. Alpha Vantage's 25 calls per day is too small. |
| Benchmark | SPY adjusted prices as a proxy for the S&P 500 | Index series from another free source | Document the proxy. SPY includes dividends once adjusted. |
| Fundamentals | SEC XBRL APIs (companyfacts, companyconcept, frames) and bulk ZIPs for backfill | None needed, the SEC is authoritative | 10 requests per second, declared User-Agent with contact email, required. |
| Filings and events | SEC submissions API and daily index (8-K, 10-K, 10-Q, Form 4 insider trades, 13F holdings) | None | 13F is quarterly and lagged. Form 4 is near real time. |
| Macro | FRED API (free key), Fed and Treasury releases | BLS, BEA | FRED allows roughly 120 calls per minute. |
| News, primary | SEC 8-K current-events feed, company investor-relations RSS, Fed and Treasury RSS |  | Official sources, no scraping. |
| News, major outlets | Free publisher RSS feeds (headline and summary), GDELT as a discovery layer | Finnhub free company news (personal use, limited) | Store headline, snippet and link only (NP-7). |
| Analyst consensus and targets | No dependable free source known | Finnhub free recommendation trends | Likely a gap. See 5.3. |
| ETF holdings and fees | Issuer holdings files, one small adapter per issuer | SEC N-PORT (lagged) | Expect staleness. Flag it. |
| Earnings dates | Company announcements and 8-K item 2.02 after the fact | A free calendar endpoint if one proves reliable | Partly a gap. |

Two gotchas to check in the first week:

- IEX-only volume is a small fraction of total market volume. Confirm whether free daily bars carry consolidated volume or IEX volume. Volume-based rules, such as the penny-stock volume floor, must be calibrated to whichever it is.
- The free price tier may cap how many symbols fit in one request. Plan the backfill (about 10 years for the universe) in batches. \[ASSUMPTION\] Batches of 200 to 500 symbols per call.

### 5.3 Known gaps and how the app handles them

PRD items that free, credible sources may not fully cover: analyst consensus and price targets, forward earnings estimates, intraday data, ETF fees, a complete earnings calendar, and social sentiment.

- Every report section carries an availability flag: complete, partial or unavailable. The UI shows it.
- The rating engine re-weights over the factors it has. It never fills a missing input with a guess.
- Confidence drops when key inputs are missing (6.5).
- 'Market sentiment' in reports is built from news tone and price action, and is labelled as such.

This is a trade-off of the free-only rule, and it is better to see it now than discover it in week six.

### 5.4 Ingestion rules

- A token-bucket rate limiter per source, with limits from config.
- Retries with exponential backoff and jitter. A circuit breaker per source, so one failing source doesn't stall others.
- Idempotent upserts on natural keys. Re-running any job is safe.
- Conditional requests (ETag, If-Modified-Since) for feeds.
- Every row stores `fetched_at`, and external data also stores `as_of` and `source_id`.
- SEC rules: declare a User-Agent with a contact email, stay at or under 10 requests per second (aim lower), cache immutable filings, and use bulk ZIP files for large backfills.

### 5.5 Corporate actions and adjusted prices

Store raw prices and the provider's adjusted close. Each night, compare the last few stored adjusted closes with freshly fetched ones. A mismatch beyond a small tolerance means a split or dividend changed history, so the symbol's full history is re-fetched and the event is logged. Scorecard returns use adjusted closes.

### 5.6 Fundamentals normalization

Raw XBRL is messy, so this is real work.

- A `concept_map` table maps several us-gaap tags to each standard metric (revenue, gross profit, operating income, net income, diluted EPS, operating cash flow, capex, debt, cash, equity, shares outstanding), in priority order.
- For each period, use the latest filed value (this absorbs restatements) but keep history.
- Compute trailing twelve months from quarters, handling the fact that Q4 is reported as a full year, not a quarter.
- Ratios (P/E, P/S, P/B, margins, ROE, debt to equity, interest coverage, FCF yield, growth rates) are computed in code and tested against known filings.
- Banks, insurers and REITs don't fit the standard statements. In v1 they get a reduced factor set and a visible note.
- An 8-K item 4.02 (non-reliance on previously issued financials) sets a restatement flag used by the quality gate.

### 5.7 Data quality and staleness

- Price sanity: flag large single-day jumps with no corporate action, gaps in trading days and duplicates.
- Fundamentals sanity: balance-sheet identity, sign checks, unit checks.
- Freshness targets: prices from the latest trading day, fundamentals within two days of a filing, news within one polling interval. Breaches show on the status page and as banners.
- A recommendation that depends on a stale critical input is paused and marked, not silently re-issued.

### 5.8 Terms and licensing

The source registry records terms and any personal-use limits. The app is private, so there is no redistribution. Respect robots.txt and each feed's terms, don't scrape paywalled sites, and keep `docs/data-sources.md` current.

## 6. Domain engines

These are plain code, with tests, and do most of the work. The LLM is used only where the text says so.

### 6.1 Universe and flags

The universe is US-listed common stocks and ETFs. Exclude OTC securities, warrants, rights, units and preferred shares. Each night, for every instrument, compute and store the flags with the numbers behind them. Thresholds live in `config/universe.yaml`.

```yaml
penny:
  max_price_usd: 5
  min_avg_dollar_volume_usd: 5000000   # [ASSUMPTION] starting point, PRD open question 9
  volume_window_days: 30
volatile:
  beta_gt: 1.5            # daily returns vs SPY
  annual_vol_gt: 0.40     # stdev of daily log returns x sqrt(252)
  window_trading_days: 252
```

Penny stocks are excluded from picks. They can be held and get reports with a warning. Volatile stocks are allowed and carry the tag.

### 6.2 Quality gate for individual stocks

The proposed replacement for 'highly trustworthy' (PRD F1-04), as config rules rather than a vibe:

- Positive operating cash flow in at least 4 of the last 5 fiscal years.
- Interest coverage and net debt to earnings within thresholds (looser for sectors where debt is normal).
- No restatement notice (8-K item 4.02) in the last 24 months.
- No going-concern warning in the latest annual report, found by the extractor with a quoted excerpt as evidence.
- Not a penny stock, with enough filings and news to analyze.

Thresholds start as \[ASSUMPTION\] defaults in `config/quality.yaml` and are yours to tune.

### 6.3 Factor and score engine

Deterministic. Factors are percentile ranks within the sector (within the ETF peer set for ETFs):

- Value: earnings yield, FCF yield, price to sales against its own history.
- Quality: margins and their stability, return on equity, leverage.
- Growth: revenue and EPS growth, and whether it is accelerating.
- Momentum: 6-month and 12-month returns (skipping the latest month), distance from the 52-week high, trend against the 50- and 200-day averages.
- Risk: volatility, beta, maximum drawdown, liquidity.
- News: a recency-weighted net score of material events (6.11).

Each horizon weights the factors differently, and the composite maps to a rating band. The 1-month horizon leans on momentum and news, the 12+ month horizon on quality, value and growth.

```yaml
ruleset_version: 1        # bump on any change. Stored on every recommendation
horizons:                 # [ASSUMPTION] heuristic starting weights, not truth
  1m:  { value: 0.05, quality: 0.10, growth: 0.10, momentum: 0.35, risk: 0.10, news: 0.30 }
  6m:  { value: 0.20, quality: 0.20, growth: 0.20, momentum: 0.20, risk: 0.10, news: 0.10 }
  12m: { value: 0.25, quality: 0.30, growth: 0.25, momentum: 0.05, risk: 0.10, news: 0.05 }
bands:                    # composite score 0 to 100
  strong_buy: 80
  buy: 62
  no_buy: 35              # below buy and at or above this. Lower is strong_no_buy
hysteresis_points: 5      # how far past a band edge a new score must go to change the rating
```

The weights are guesses. The scorecard decides whether they deserve to stay, and every change is a new `ruleset_version`.

### 6.4 The LLM overlay

After the deterministic score, the `ticker_overlay` task (role: analyst) receives the structured facts, the recent evidence items and the current score. It returns:

- A score adjustment between -10 and +10 points, which code clamps and applies. A non-zero adjustment must cite evidence IDs.
- Risk flags from a fixed list.
- 'What could go against it' items, each with evidence IDs.

The model alone can never move a rating more than one band, and it never touches a price or ratio.

### 6.5 Confidence

A numeric confidence score from five inputs: data completeness (how many required inputs are present and fresh), agreement among factors, evidence quality (share of primary sources, recency), model reliability on this call (valid output, no retries, no fallback) and a volatility penalty. It maps to Low, Medium or High. The raw score is stored, so once enough calls resolve the scorecard can calibrate the mapping (PRD success metric: calibration).

### 6.6 Projection engine (F1-03)

Numbers from code, not from the model:

- Base case, stocks: forward EPS is trailing EPS grown at a shrunk, clipped blend of its historical growth. The exit multiple is a blend of the current P/E and its multi-year median. Base price is EPS times multiple.
- Bear and bull: shift growth and multiple by amounts scaled to that company's own history of earnings and multiple variability.
- Range floor: the bear-to-bull range is never narrower than a volatility cone from realized volatility (for example the 20th to 80th percentile of the implied distribution). A stock's own price swings set a minimum honesty level.
- ETFs and loss-making companies: volatility cone around a conservative drift from config \[ASSUMPTION\], and the projection is labelled low reliability.
- Output: base, bear and bull at 6 and 12 months, with the assumptions saved as JSON.
- Limitation: free data has no guidance or estimates, so news about guidance moves the rating and confidence, not the projection. The UI says so until the next filing refreshes it.

Every projection is logged and scored for interval coverage: do outcomes land inside the range about as often as the range implies?

### 6.7 Stickiness engine (PRD principle 4)

One state record per instrument and horizon. A re-evaluation happens only on a trigger:

1. A material news event for the ticker.
2. A new filing (10-K, 10-Q, or a material 8-K item).
3. A scheduled review: weekly for 1 month, monthly for 6 months, quarterly for 12+ months (configurable).
4. A price shock beyond a set multiple of the stock's daily volatility.
5. A change to the investor profile.
6. A new ruleset or model epoch. This runs in shadow and applies nothing immediately (4.9).

A new rating is applied only if the band changes and the score crosses the hysteresis margin, or if a material event shows the thesis broke (an evidence-backed flag). A change creates a new recommendation that supersedes the old one, with a stored reason, and an alert if it concerns a holding or a current pick. An unchanged re-evaluation writes a row to `recommendation_review`, so the app can say 'held steady, and why' without creating a duplicate call to score.

### 6.8 Pick selection (Feature 1)

Nightly:

1. Filter the universe: not penny, quality gate for stocks, minimum liquidity for ETFs.
2. Score everything (deterministic).
3. Shortlist the top K (default 40, configurable) by score adjusted for profile fit.
4. Run the overlay on the shortlist only.
5. Rank by final score, confidence and profile fit. Apply diversification (a cap on picks per sector) and keep fewer than 10.
6. Apply stickiness: existing picks stay unless a rule removes them. Record why anything entered or left.
7. Attach position-size suggestions (6.9).

Risk band influences ranking and sizing. Volatile-tagged picks are allowed but are penalized and sized smaller for lower risk bands (PRD ON-4).

### 6.9 Portfolio and tax-lot engine (Feature 2)

Pure, deterministic code with the heaviest tests in the project.

- Lot ledger: each purchase is a lot (date, shares, cost per share). Sales are recorded as disposals against lots. Remaining shares are shares minus disposals. Lot-selection default is specific identification, with FIFO and highest-cost-first selectable \[DECISION NEEDED, PRD open question 5\].
- Holding period: long-term when the sale date is more than one year after the purchase date. `days_to_long_term` is the purchase date plus one year plus one day, minus today, floored at zero. Test the boundaries: leap-day purchases, year ends, and the exact anniversary.
- Gains: unrealized gain or loss per lot, classed short-term or long-term. Dollar tax estimates appear only if you set rates \[DECISION NEEDED, PRD open question 7\].
- Wash-sale warning: a loss sale with a purchase of the same ticker in the previous 30 days, or a planned rebuy within 30 days. This applies across accounts, retirement included. The app warns and does not compute basis adjustments.
- Account type per lot (taxable or retirement) is optional \[DECISION NEEDED, PRD open question 6\]. Tax logic is skipped for retirement lots.
- Concentration: weights by market value against limits in `config/limits.yaml` (defaults 10% per stock, 30% per sector) \[DECISION NEEDED, PRD open question 3\]. ETF look-through uses the holdings the app has and is labelled approximate.
- Position sizing: target weight = base weight for the risk band, times a confidence multiplier, times a volatility scaler, then capped by limits and remaining headroom.

### 6.10 Funding planner (F2-07)

Input: a target purchase (ticker and dollar amount), or a pick flagged 'worth funding'. It is a pure function, with no LLM.

Candidate lots are ranked, in this lexicographic order (as the PRD specifies):

1. Weakest outlook first (current action for that holding, worst first).
2. Lowest tax cost: loss lots first (with wash-sale warnings), then long-term gains, then short-term gains, deferring lots close to long-term status.
3. Reducing overweight positions.

It fills greedily until the amount is reached, respecting concentration limits after the trade. The output is the lots to sell with amounts, each lot's gain class, warnings, the portfolio after the trade, and a second-best alternative.

Proactive mode: a pick is 'worth funding' when its final score clears a threshold, beats your weakest holding by a set margin, and fits the limits. The nightly job then pairs it with a plan and creates an alert. The LLM only helps write the explanation, using placeholders (4.14).

### 6.11 News intelligence (NP-1 to NP-7)

1. Fetch from sources by tier (5.2).
2. Normalize and de-duplicate: canonical URL, title fingerprint, then embedding similarity for near-duplicates.
3. Link entities: rule-based first (ticker and company-name dictionary), and the extractor only for ambiguous cases.
4. Confirm: tier 1 sources count on their own. Other claims need a tier 1 source or two independent major outlets. Single-source, unconfirmed claims never trigger changes (NP-4).
5. Classify (extractor): event type from a fixed list (earnings, guidance change, M&A, legal or regulatory, executive change, macro or sector, other), direction, novelty, and an evidence excerpt that code verifies is an exact substring of the source text. SEC 8-K item codes give many event types with no model at all.
6. Score materiality deterministically: event-type weight times novelty times source tier times relevance to a holding or pick. The model supplies inputs. Code makes the call.
7. Trigger: above the threshold, enqueue a high-priority re-analysis of the ticker (6.7).
8. Alert when the result is a recommendation change, or severity is high.

Macro events (Fed decisions, rates, inflation prints) map to sector sensitivities in config, so one rate decision can touch many holdings.

### 6.12 Report generation (Feature 3)

A background job with these steps, each emitting a progress event:

1. Resolve the ticker and check inputs are fresh.
2. Gather all facts (deterministic).
3. Compute scores and a rating for each of the three horizons.
4. Run the overlay (analyst).
5. Compose each section (writer) using fact placeholders.
6. Run the number guard and evidence checks. Drop unsupported claims.
7. Save the report as structured JSON with the `inputs_hash`.

Cached by ticker and `inputs_hash`, so unchanged inputs reopen instantly. 'What changed since last report' is a diff of structured fields, not of prose.

The 52-week view is computed from stored bars: the 52-week high and low, position within the range, percent off the high, the 50- and 200-day averages, volume trend, and event markers from filings and material news. The browser draws it with Lightweight Charts. ETF reports add the top 10 holdings, fees (if available), sector breakdown and overlap with the portfolio.

Report JSON is a list of sections. Each has an id, title, availability flag and typed blocks (text, fact table, chart spec, claim list with evidence IDs).

### 6.13 Briefing and alerts

The briefing builder reads stored results only: a market summary (index moves from stored prices plus the day's top macro and market events, written by the writer), the picks, portfolio changes since yesterday, material news on holdings, this week's events, and the scorecard summary. If the model is unavailable, it builds a data-only briefing with a banner. A stale briefing is better than none, and a blank one is worse than both.

Alert types: recommendation change, material news, funding opportunity, limit breach, stale data and system failure. Each alert has a dedupe key and severity and moves through new, seen, acknowledged and dismissed. A daily cap protects alert quality (PRD HS-4).

### 6.14 Scorecard engine (TR-1 to TR-6)

- Log at creation, mark to market nightly, resolve at the horizon end. Horizons in calendar days: 30, 182 and 365 \[ASSUMPTION\].
- Outcome per call: return, S&P 500 (SPY) return over the same window, and excess return.
- Primary metric: risk-adjusted performance of the followed calls against SPY. A simulated portfolio holds the buy-rated calls at their suggested weights, and the metric is alpha and information ratio against SPY.
- Supporting metrics: directional accuracy (a buy is right if the stock beats SPY, a no-buy is right if it lags), downside capture (performance in windows where SPY fell), calibration by confidence bin, and projection interval coverage.
- Every metric shows its sample size and a bootstrap confidence interval, and says 'not enough data yet' below a minimum count. Overlapping windows make small samples weaker than they look, and the UI must not hide that.
- Results are split by `ruleset_version` and `model_epoch`.

## 7. Database design

### 7.1 Principles

One PostgreSQL database with plain tables. UTC `timestamptz` everywhere. Prices and money are `NUMERIC`, never floats. External data rows carry `source_id`, `as_of` and `fetched_at`. Recommendations and evaluations are append-only. Migrations use Alembic, forward-only in production, with a backup taken before each run.

### 7.2 Tables by group

- Market data: `instrument`, `instrument_alias` (ticker changes), `price_bar_daily` (primary key instrument and date), `corporate_action_event`, `fundamental_fact` (raw XBRL facts), `concept_map`, `fundamental_metric` (standardized and derived), `filing`, `insider_txn` (Form 4), `institutional_holding` (13F), `etf_holding`, `macro_point`, `universe_flag_daily` (penny and volatile flags with the numbers behind them).
- News: `source`, `news_item`, `news_cluster`, `news_entity`, `news_event` (type, direction, novelty, materiality score, confirming sources, evidence excerpt).
- Household: `app_user`, `profile_version`, `profile_answer`, `holding_lot`, `lot_disposal`, `cash_balance` (optional), `limits_config_version`.
- Recommendations: `recommendation`, `recommendation_reason` (text with fact and evidence references), `recommendation_review` (reaffirmations), `evaluation` (outcomes), `report`, `briefing`, `alert`.
- Traceability and ops: `llm_call`, `input_snapshot`, `pipeline_run`, `config_snapshot`, `eval_run`, `eval_result`, `audit_log`. The job queue tables belong to Procrastinate and are left alone.

### 7.3 Key tables

```sql
CREATE TABLE holding_lot (
  id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  instrument_id  BIGINT NOT NULL REFERENCES instrument(id),
  purchase_date  DATE NOT NULL,
  shares         NUMERIC(18,6) NOT NULL CHECK (shares > 0),
  cost_per_share NUMERIC(18,6) NOT NULL CHECK (cost_per_share >= 0),
  account_type   TEXT CHECK (account_type IN ('taxable','retirement')),  -- optional
  note           TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at     TIMESTAMPTZ
);

CREATE TABLE lot_disposal (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  lot_id          BIGINT NOT NULL REFERENCES holding_lot(id),
  sale_date       DATE NOT NULL,
  shares          NUMERIC(18,6) NOT NULL CHECK (shares > 0),
  price_per_share NUMERIC(18,6) NOT NULL
);

CREATE TABLE recommendation (
  id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind                TEXT NOT NULL CHECK (kind IN ('pick','portfolio_action','report_rating')),
  instrument_id       BIGINT NOT NULL REFERENCES instrument(id),
  horizon             TEXT NOT NULL CHECK (horizon IN ('1m','6m','12m')),
  rating              TEXT NOT NULL,       -- strong_buy|buy|no_buy|strong_no_buy or add|hold|trim|sell
  score               NUMERIC(6,2) NOT NULL,
  confidence          TEXT NOT NULL,       -- low|medium|high
  confidence_score    NUMERIC(5,2) NOT NULL,
  price_at_issue      NUMERIC(18,6) NOT NULL,
  projection          JSONB,               -- base, bear, bull at 6m and 12m, plus assumptions
  suggested_weight    NUMERIC(5,2),
  risk_band           TEXT NOT NULL,
  profile_version_id  BIGINT NOT NULL,
  ruleset_version     TEXT NOT NULL,
  model_epoch         INT NOT NULL,
  input_snapshot_id   BIGINT NOT NULL,
  issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  change_reason       TEXT,
  superseded_by       BIGINT REFERENCES recommendation(id)
);

CREATE TABLE llm_call (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  task            TEXT NOT NULL,
  role            TEXT NOT NULL,
  model_profile   TEXT NOT NULL,           -- name in config/llm.yaml
  model_name      TEXT NOT NULL,           -- as reported by the server
  model_epoch     INT NOT NULL,
  prompt_version  TEXT NOT NULL,
  input_hash      TEXT NOT NULL,
  tokens_in       INT, tokens_out INT, latency_ms INT,
  schema_valid    BOOLEAN NOT NULL,
  retries         INT NOT NULL DEFAULT 0,
  is_shadow       BOOLEAN NOT NULL DEFAULT FALSE,
  error           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Append-only enforcement: a trigger on `recommendation` rejects deletes and rejects updates, except setting `superseded_by` once from null.

```sql
CREATE FUNCTION recommendation_guard() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'UPDATE' AND OLD.superseded_by IS NULL AND NEW.superseded_by IS NOT NULL
     AND (to_jsonb(NEW) - 'superseded_by') = (to_jsonb(OLD) - 'superseded_by') THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'recommendations are append-only';
END $$ LANGUAGE plpgsql;
```

### 7.4 Indexes

- `price_bar_daily`: primary key on (instrument\_id, bar\_date). That covers almost every query.
- `recommendation`: partial index on (instrument\_id, horizon, issued\_at desc) where `superseded_by IS NULL`, which answers 'what is the current call' fast.
- `news_item`: index on published\_at, and `news_entity` on (instrument\_id, published\_at).
- Embeddings: an HNSW index on the pgvector column for near-duplicate search. Each vector stores the embedding model name (4.9).

### 7.5 Retention

- Raw API archives: 60 days. News metadata and snippets: 18 months.
- `llm_call` detail: 90 days, except calls linked to a recommendation, which are kept through their `input_snapshot`.
- Recommendations, evaluations, lots, disposals, profile history and the audit log: kept for good. The scorecard depends on them.

### 7.6 Seed and config sync

`concept_map`, the source registry and the question set for the risk questionnaire are loaded from versioned files at startup. A `config_snapshot` row records the hash of every config file whenever it changes, so any past recommendation can be tied to the exact rules and models in force.

## 8. API design

### 8.1 Conventions

- REST and JSON under `/api/v1`. FastAPI generates the OpenAPI spec, and the web app's typed client is generated from it at build time (openapi-typescript or Orval, both MIT), so frontend and backend can't drift.
- Decimal values travel as strings. Timestamps are ISO 8601 in UTC. Cursor pagination on lists.
- Every data object carries `as_of` and where relevant `source`, and every generated object carries `model` metadata (profile, epoch, prompt version).
- Errors use problem+json (RFC 9457). Creating endpoints accept an idempotency key.
- Additive changes only within `v1`. Contract tests run in CI.
- Only `POST /reports` and the admin test endpoints touch the LLM, and only workers call outside data sources. Everything else reads stored results.

### 8.2 Endpoints

| Endpoint | Purpose | PRD |
| --- | --- | --- |
| GET /home | Briefing summary, top alerts, scorecard summary, data freshness | HS-1 to HS-3 |
| GET /briefings/{date} | A full briefing | HS-2 |
| GET /alerts, POST /alerts/{id}/acknowledge | List and acknowledge alerts | HS-3 |
| GET /picks, GET /picks/{id} | Current picks and detail | F1 |
| GET /portfolio | Positions, lots, tax context, concentration | F2-02, F2-05, F2-06 |
| POST /portfolio/lots, PATCH and DELETE /portfolio/lots/{id} | Manage lots | F2-01 |
| POST /portfolio/lots/{id}/disposals | Record a sale | F2-01 |
| GET /portfolio/actions | Suggested action per holding and horizon | F2-03 |
| POST /portfolio/funding-plans | Deterministic plan to fund a buy | F2-07 |
| GET and PUT /portfolio/limits | Concentration limits | F2-06 |
| POST /reports, GET /reports/{id}, GET /reports?ticker= | Request, read and list reports | F3 |
| GET /reports/{id}/events | Progress stream (SSE) | F3-05 |
| GET /instruments/search?q= | Ticker lookup | F3-01 |
| GET /instruments/{ticker}/chart?range= | Bars and event markers | F3-03 |
| GET /scorecard, GET /scorecard/calls | Metrics and individual calls | TR-1 to TR-6 |
| GET /profile, POST /profile/answers | Questionnaire and risk band | ON-1 to ON-3 |
| GET /admin/status | Pipelines, data freshness, models in use | NFR reliability |
| GET /admin/llm, POST /admin/llm/test, /use, /rollback | Model registry, test, swap, roll back | section 4 |
| POST /admin/pipelines/{name}/run | Run a job now | operations |
| GET /export, DELETE /data | Export and delete all data | NFR security |

### 8.3 Example objects

Shown as YAML for readability. The wire format is JSON.

```yaml
# GET /picks/4812
id: 4812
instrument: { ticker: EXMP, name: Example Corp, kind: stock, sector: Industrials }
horizon: 12m
rating: buy
confidence: medium
flags: [volatile]
price_at_issue: '48.20'
projection:
  6m:  { bear: '42.10', base: '51.30', bull: '58.90' }
  12m: { bear: '40.00', base: '55.80', bull: '68.20' }
  note: Range is never narrower than the stock's own volatility implies
suggested_weight_pct: '4.0'
reasons:
  support:
    - text: 'Revenue grew {{fact:EXMP.rev_cagr_3y}} a year over three years'
      evidence: [fact:EXMP.rev_cagr_3y, filing:0000000000-26-000123]
  against:
    - text: 'A large customer accounts for {{fact:EXMP.top_customer_pct}} of sales'
      evidence: [filing:0000000000-26-000123]
as_of: '2026-11-17T22:05:00Z'
model: { profile: gemma4-31b, epoch: 3, prompt: ticker_overlay@2 }
ruleset_version: '1'
```

```yaml
# POST /portfolio/funding-plans
request:  { buy: { ticker: EXMP, amount_usd: '5000' } }
response:
  sells:
    - { lot_id: 77, ticker: OLDX, shares: '20', proceeds_usd: '2400', gain_class: long_term_loss }
    - { lot_id: 81, ticker: SLOW, shares: '15', proceeds_usd: '2600', gain_class: long_term_gain }
  warnings: ['OLDX: purchase within 30 days, wash-sale risk']
  after_trade: { max_position_pct: '8.1', max_sector_pct: '24.0', limits_ok: true }
  alternative_plan_id: 9
```

The text strings are shown with `{{fact:...}}` placeholders. The API resolves them against stored facts before returning them, so the browser always receives finished text with the values filled in.

### 8.4 Long jobs

`POST /reports` returns 202 with a report id and creates a job. `GET /reports/{id}/events` streams Server-Sent Events: step started, step finished, section ready, done or failed, with heartbeats. The client reconnects with `Last-Event-ID`, and falls back to polling `GET /reports/{id}`. A request for a ticker whose cached report is still valid returns it immediately.

### 8.5 Targets

&#91;ASSUMPTION\] Stored-data endpoints: p95 under 300 ms on the home network and under 1 second remotely. The home snapshot is one query to a precomputed row. Set final numbers after the first benchmark.

## 9. Frontend

### 9.1 Structure

- React and TypeScript on Vite. TanStack Query for server state, React Router for routes, Tailwind and shadcn/ui for components, React Hook Form with Zod for forms (lot entry, questionnaire).
- The typed API client is generated from OpenAPI. Components never build URLs by hand.
- Charts: Lightweight Charts for price and 52-week views (keep the attribution TradingView's license asks for), Recharts or ECharts for allocation and scorecard charts.

### 9.2 Screens

- Home: Daily Briefing and Alerts first, then a compact scorecard summary, with a data-freshness line (HS-1).
- Picks: fewer than 10 cards with rating, horizon, confidence, flags, 6 and 12 month range, suggested size, and a 'why' drawer showing reasons, risks, evidence and sources.
- Portfolio: positions with expandable lots, gain class and days to long-term, concentration bars against limits, suggested actions by horizon, and the funding planner.
- Report: ticker search, progress view while generating, the report (ratings per horizon, support and no-buy facts, fundamentals, market view, 52-week chart, catalysts, risks, what changed), and history.
- Scorecard: primary metric, supporting metrics with sample sizes and intervals, calls table.
- Profile: onboarding questionnaire, risk band, history.
- Admin: status, data freshness, pipeline runs, model registry with test, eval, swap and rollback.

### 9.3 Mobile first

- Phone layout is the default. A bottom tab bar replaces the sidebar. Tables collapse into cards, and charts get touch gestures and their own scroll containers. The page body never scrolls sideways.
- Large tap targets, light and dark themes.
- The home screen fetches once (`GET /home`) and renders from that single response.

### 9.4 Performance and resilience

- Route-level code splitting. Skeleton screens. Stale-while-revalidate caching.
- Optional PWA: a service worker keeps the last home snapshot, so the app opens instantly even on a slow connection, with an 'as of' banner. It never shows cached data without saying how old it is.

### 9.5 Trust patterns

- Every number has a tooltip with its source and as-of date.
- Confidence shown as a labeled chip, never color alone.
- Availability flags (complete, partial, unavailable) on report sections.
- Ranges drawn as bands, not single lines. Projections are always labelled as estimates.
- A stale-data banner whenever a critical input is old, and a footer with the informational-only notice (PRD section 15).

### 9.6 Accessibility

Contrast to WCAG AA, full keyboard navigation, text alternatives that summarize each chart, and visible focus states.

## 10. Security and privacy

### 10.1 What we are protecting against

Internet scanners, a lost or stolen phone or laptop, malicious text in news or filings (prompt injection), a compromised dependency, portfolio data leaking to a remote model, and plain data loss. Multi-tenant isolation is not a concern, since there is one household.

### 10.2 Access and network

- public HTTPS: Caddy with automatic certificates, strong authentication, and rate limiting, optionally behind a free tunnel service. Easier for family members, a larger attack surface.

### 10.3 Authentication

- Passkeys (WebAuthn) as the primary login, one per device. TOTP as a fallback, plus printed recovery codes. Argon2id for any password.
- Session cookies are HttpOnly, Secure and SameSite=Lax, with a sliding idle timeout. CSRF protection on state-changing requests.
- Login rate limiting with backoff. Re-authentication for sensitive actions: export, delete, and swapping a model.

### 10.4 Data protection

- TLS on every hop. Disk encryption on every host that stores data (LUKS, FileVault or BitLocker). Encrypted backups.
- Data minimization: manual entry means no brokerage credentials, and no government IDs or account numbers are ever stored.
- Requests to data providers reveal the tickers you ask for. Fetching prices for the whole universe, not just your holdings, keeps your holdings out of provider logs.

### 10.5 Secrets and database access

- No secrets in git. SOPS with age encrypts config secrets, and production uses container secrets. Rotate API keys periodically.
- Separate database roles: the API (read and limited writes), workers (writes to pipeline tables), and a read-only role for analysis. Grants plus the append-only trigger protect the recommendation log.

### 10.6 Supply chain

- Lockfiles with hashes. Container images pinned by digest, minimal base images, non-root containers.
- CI runs pip-audit, npm audit and a container scan (Trivy). Updates are reviewed monthly, and brand-new releases wait a couple of weeks.

### 10.7 LLM-specific controls

- Prompt-injection controls and the number guard from 4.14. Models have no tools and no database access.
- `allow_remote` and `data_class` block private data from remote models (4.9).
- Logs never contain secrets or full portfolio contents. Prompts are logged as hashes unless a debug flag is on.

### 10.8 Web hardening

Content-Security-Policy with no inline scripts, HSTS, `X-Content-Type-Options`, `frame-ancestors none`, a strict Referrer-Policy, CORS limited to the app's own origin, Pydantic validation on every input, and parameterized queries only.

### 10.9 Backups and recovery

Nightly encrypted dump with restic and age to two places (a local disk, and a second machine or free offsite storage). A restore test every month. \[ASSUMPTION\] Recovery point of 24 hours, recovery time of a few hours.

### 10.10 Audit log

Logins, lot changes, config changes (including every model swap), exports and deletions. Kept for a year.
