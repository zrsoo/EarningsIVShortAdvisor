# Project Brief — Earnings IV Short: Scraper + Decision Maker (Greenfield Repo)

## 0) Executive Summary

We will build a **from-scratch, implementation-agnostic** system that ingests upcoming **earnings events** and **options/price data**, computes the **volatility signals** we’ve defined, and outputs an **actionable watchlist** with a single, clear label per ticker: **Recommended / Consider / Avoid**. The first iteration focuses on **high-fidelity screening and decisioning** (not execution), producing outputs that a trader can act on manually and that future tooling (backtester/execution engine) can consume.

---

## 1) Strategy Objective (What we’re exploiting)

Ahead of earnings, **front-month implied volatility (IV)** often rises above both:

* the stock’s recent **realized volatility (RV)**, and
* the **longer-dated IV** (creating an **inverted term structure**).

We seek to **systematically short that overpriced front-month IV** (e.g., via short straddles/strangles, double calendars, iron condors, etc.), and capture the **post-earnings IV crush**. The decision maker will **only recommend candidates** where this setup is statistically compelling **and** practically tradable.

---

## 2) Scope & Non-Goals

### In-Scope (v1)

* **Data ingestion** for: earnings timing, daily OHLCV, option chains across expirations.
* **Signal computation** for each candidate ticker (see §5).
* **Decision logic** mapping signals to **Recommended / Consider / Avoid** with transparent reasons.
* **Deterministic outputs** as machine-readable files (e.g., CSV/JSON) plus human-readable summary.
* **Robustness**: caching, throttling, failure handling, reproducible runs.

### Out-of-Scope (v1)

* **Trade execution** (no broker API orders).
* **Full historical backtests with point-in-time options** (we’ll plan them, but not ship in v1).
* **Complex UI**. Outputs can be consumed by any external viewer or spreadsheet.

---

## 3) Users & Constraints

* **Primary user**: a retail trader operating a **small account (\~\$1k–\$3k)** in the EU/RO, choosing **defined-risk** structures by preference.
* **Data constraints**: free/public feeds may be **rate-limited, stale off-hours, or incomplete**; the system must handle this gracefully, document gaps, and make conservative decisions when data are borderline.
* **Time zones**: default operational timezone is **Europe/Madrid**; earnings calendars and market sessions must be normalized to **absolute timestamps**.

---

## 4) Inputs & Outputs

### Required Inputs

* **Earnings events** for a target date or window (ticker, date, session: before-open / after-close / during).
* **Daily OHLCV** (≥ 3 months look-back) to compute **RV30** and **30-day average volume**.
* **Options chains** for near expiries up to and including the first expiry **≥ 45 DTE** (front weekly(s) + monthlys as needed).
* **Spot price** (close or last trade) at the time of screening.

### Outputs

* **Watchlist file** with one row per ticker containing:

  * Identifiers: symbol, as-of timestamp, earnings timestamp, session.
  * **Liquidity**: avg\_volume\_30d (shares), volume\_pass (bool).
  * **Vol signals**:

    * IV30 (annualized)
    * RV30 (annualized, **Yang–Zhang**)
    * IV30/RV30 ratio and pass/fail (≥ **1.25**)
    * Term structure **slope** front→45DTE (per-day) and pass/fail (≤ **–0.00406**)
  * **Expected move %** (front ATM straddle mid ÷ spot)
  * **Decision label**: Recommended / Consider / Avoid
  * **Audit fields**: expiries used, DTE list, ATM strikes picked, data freshness flags, any fallbacks triggered.

* **Human-readable summary** per run:

  * Count by decision label
  * Top 10 candidates with the strongest edges
  * Any data quality warnings (e.g., missing IVs at ATM, wide spreads)

---

## 5) Signal Definitions (Ground Truth)

> All annualization conventions and windows must be explicit and consistent; log all assumptions in the run metadata.

1. **Liquidity filter**

   * **avg\_volume\_30d (shares)** = 30-day simple average of daily share volume.
   * **Pass if ≥ 1,500,000**.

2. **RV30 (Realized Volatility, 30 trading days)**

   * **Yang–Zhang** estimator over 30 daily bars using Open/High/Low/Close.
   * Annualize via √252.

3. **ATM IV per expiry**

   * For each considered expiry:

     * Determine **front DTE** (drop same-day expiries; front = smallest DTE > 0).
     * Pick **ATM strike** as the one minimizing |strike − spot|.
     * For both call and put at that strike, compute a **mid price** (bid/ask midpoint). If missing, try the nearest adjacent strike **once**.
     * Prefer **computed implied volatility** from mid prices (invert BS) when available; if vendor IV is used, record provenance.
     * **ATM\_IV(expiry)** = robust combination of call/put IVs (e.g., mean or median if both exist; if only one side is valid, use that with a flag).

4. **IV30 (Implied Volatility at 30 DTE)**

   * Build the **term curve** from (DTE, ATM\_IV).
   * Interpolate to **30 DTE**:

     * If the curve spans below and above 30, interpolate; if 30 < min DTE, use min-DTE value; if 30 > max DTE, use max-DTE value.
   * **IV30** = term\_curve(30).

5. **Term-structure slope (front → 45 DTE)**

   * **slope = (IV\@45 − IV\@front) / (45 − front\_DTE)**.
   * **Pass if slope ≤ –0.00406 per day** (i.e., sufficiently inverted).

6. **IV richness ratio**

   * **IV30 / RV30**.
   * **Pass if ≥ 1.25**.

7. **Expected move (%)**

   * From **front expiry** ATM straddle:

     * **straddle\_mid = call\_mid + put\_mid** at ATM strike.
     * **expected\_move\_pct = straddle\_mid / spot × 100**.
   * If mid quotes are missing at exact ATM, try nearest strike once; otherwise set to **None** with a warning.

8. **Decision logic**

   * **Recommended**: liquidity **PASS**, IV30/RV30 **PASS**, slope **PASS**.
   * **Consider**: slope **PASS** and **exactly one** of {liquidity, IV30/RV30} **PASS**.
   * **Avoid**: all other cases.

---

## 6) Screening Pipeline (End-to-End)

1. **Select date/window** for earnings.
2. **Fetch earnings list** (ticker, timestamp, session). If multiple sources disagree, reconcile and log both; drop ambiguous entries.
3. For each ticker:

   * Fetch **OHLCV** (≥ 3 months).
   * Compute **avg\_volume\_30d** and **RV30 (YZ)**.
   * Fetch **expiries**; drop same-day; select all up to and including first **≥ 45 DTE**.
   * For each selected expiry, fetch **option chain**; pick **ATM**; compute IVs and **front-straddle mid**.
   * Build **term curve**; compute **IV30**, **slope**.
   * Compute **IV30/RV30**, **expected move %**.
   * Apply decision logic; assemble row with **audit fields**.
4. **Write outputs**; emit summary; return non-zero status if too many data errors occurred.

---

## 7) Data Quality & Fallbacks

* **Missing/NaN IVs** at ATM:

  * Try the **nearest neighbor** strike once.
  * If still missing, compute IV from **call-only** or **put-only** mid if available; flag the asymmetry.
* **No expiry ≥ 45 DTE** → mark **insufficient curve**; skip ticker.
* **Stale or off-hours quotes**:

  * Tag each quote with provider timestamp; emit **staleness warning** if older than a configured threshold.
* **Wide spreads**:

  * Estimate **spread/spot** for ATM; flag **illiquid** if above threshold; proceed but **downrank** confidence in summary.
* **Calendar ambiguity**:

  * If earnings timestamp is unknown or conflicts (e.g., one source says pre-open, another after-close), mark **uncertain**; include but annotate.

---

## 8) Eligibility Filters & Exclusions (Pre-Trade Practicality)

* Minimum **stock price** (e.g., > \$5) to avoid penny behaviors.
* Exclude **OTC** tickers, tickers without **weeklies**, or with **extremely low option OI** at ATM.
* Optional sector filters (e.g., exclude **binary** event names like small biotechs).
* Explicitly mark **corporate actions** (splits, M\&A) near the event.

---

## 9) Configuration, Thresholds, and Reproducibility

* All **thresholds** and **time conventions** are **externally configurable** (no hard-coding):

  * min\_avg\_volume\_shares (default 1,500,000)
  * min\_ivrv\_ratio (default 1.25)
  * max\_slope\_per\_day (default –0.00406)
  * lookback\_days\_ohlc (default 90)
  * expected\_move\_method flags
* Every run writes a **manifest**: version, parameters, start/end times, counts by label, warnings, data-source info, and a hash of input lists—so results are auditable.

---

## 10) Risk & Suitability Notes (For the Human Loop)

* **Tail risk exists**: even perfect signals can’t prevent outsized losses on rare earnings shocks. The tool **recommends** candidates but does **not** size or execute.
* For small accounts, the summary should **suggest defined-risk constructions** (e.g., double calendar or iron condor) and note when **naked** structures are **not advisable**.
* Include an **educational footer** in summaries: what “Recommended” means, typical **exit timing** (close next open), and **common pitfalls** (illiquidity, slippage).

---

## 11) Testing & Validation (Core)

* **Unit tests**:

  * Yang–Zhang over known fixtures.
  * ATM selection (ties, odd strikes).
  * Term-curve interpolation edges (extrapolation, flat segments).
  * Slope math (front DTE near 1–7).
  * Decision mapping truth table.
* **Golden tests**:

  * A handful of **well-known tickers** around recent earnings with manually captured chains; verify stable outputs.
* **Property tests** (optional):

  * IV30/RV30 monotonicity under synthetic perturbations (sanity checks).

---

## 12) Operational Considerations

* **Rate-limit hygiene**: built-in throttling; request-level caching; batch ordering that minimizes duplicate calls; backoff on 429/5xx.
* **Idempotent runs**: same inputs + same time window → same outputs (modulo provider updates).
* **Observability**: structured logs per ticker, per request; summary of cache hits/misses; top failure reasons.

---

## 13) Roadmap (Post-v1)

* **Backtest & Monte Carlo**: event-driven backtests with **point-in-time options** (requires proper vendor); MC using historical earnings gaps.
* **Multi-source enrichment**: secondary earnings/quote sources for redundancy; broker data to replace public IV.
* **Confidence scoring**: beyond PASS/FAIL, compute a confidence metric (data quality + signal strength).
* **Execution handoff**: export compatible tickets (strikes/width/size) for brokers; later, optional auto-execution.
* **Portfolio limits**: per-event and aggregate exposure guidance; watchlist capacity analysis per earnings season.

---

## 14) Success Criteria (What “good” looks like)

* On a typical earnings week, the tool produces a **clean watchlist** (low error rate) with **clearly inverted term structures** and **rich IV30/RV30** cases at the top.
* Human checks in a broker platform **agree qualitatively** with the outputs (front IV >> back IV, reasonable expected move, liquid ATM).
* Over multiple weeks, **post-mortems** show that **Recommended** names, when traded with defined-risk structures and exited next open, generally realize the expected IV crush (acknowledging normal tail outliers).

---

## 15) Compliance & Disclaimers

* Outputs are **for research/education**; **not investment advice**.
* No claims to profitability; results depend on slippage, fees, and shocks.
* Respect data providers’ **terms of service**; implement polite request behavior.

---

### One-page TL;DR (to paste ahead of work)

* Build a **new repo** that **ingests earnings + OHLCV + options**, computes **IV30, RV30 (YZ), front→45DTE slope, expected move**, and labels each ticker **Recommended / Consider / Avoid** per fixed thresholds: **avg vol ≥ 1.5M**, **IV30/RV30 ≥ 1.25**, **slope ≤ –0.00406/day**.
* Focus on **robust screening**, **transparent audit fields**, and **machine-readable outputs**.
* Be conservative with **data quality**; cache, throttle, and annotate all fallbacks.
* No tech stack or UI commitments here—keep it implementation-neutral so we can choose in the next thread.
