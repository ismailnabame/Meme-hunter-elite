
# MEME HUNTER ELITE — v1 Hypothesis Spec

**Version:** 0.1 (draft for review)
**Status:** Pre-registration — freeze before any data is collected
**Purpose:** Define, in falsifiable terms, the single hypothesis v1 must prove or disprove before any capital is committed.

---

## 1. The hypothesis (one sentence)

> *At T+60 seconds after pool creation on Solana, a rule-based filter combining liquidity quality, independent-buyer acceleration, and hard risk vetoes identifies tokens whose expected 24-hour return, net of realistic fees and slippage, is positive and whose probability of a rug or honeypot is below 2%.*

If the data does not support this, we do not proceed to live capital. We revise the hypothesis and re-test, or stop.

---

## 2. Operational definitions (freeze these now)

### 2.1 "Pool creation" timestamp
- **Definition:** The block timestamp of the first successful `initialize` / `create_pool` transaction on the primary DEX (Raydium AMM, Pump.fun bonding curve graduation, Orca Whirlpool, Meteora DLMM) for a given mint.
- **Edge cases:** Tokens with multiple pools → use the earliest pool with ≥ $5,000 initial liquidity. Ignore pools below that threshold.
- **Data source:** Indexed from RPC + provider (Helius/Shyft) with block-level precision.

### 2.2 "T+60s evaluation window"
- **Definition:** The 60-second window starting at pool creation timestamp.
- **Why 60s:** Long enough for initial bot noise to settle and for independent buyers to appear; short enough that the opportunity is still early. **This is a parameter we will test at 30s, 60s, 120s in validation — but the primary hypothesis is fixed at 60s.**

### 2.3 "Independent buyer"
- **Definition:** A wallet whose first interaction with the token occurs within the evaluation window, *and* which does not share any of the following with another buyer in the same token:
  - Same funding wallet (direct SOL transfer within 30 days, ≥ 0.05 SOL)
  - Same fee-payer history (co-signed transactions in last 90 days)
  - Same first-transaction source (same wallet funded both at their genesis)
- **Confidence:** We classify a pair as "likely related" at ≥ 0.7 confidence from the above heuristics; below that, they count as independent. **We report both raw and de-duplicated buyer counts** — never hide the raw number.

### 2.4 "Liquidity quality"
Composite of three sub-features at T+60s:
- **L_total:** Total pool liquidity in USD (SOL price × SOL in pool, both sides).
- **L_growth:** `(L_T60 − L_T0) / L_T0` — growth in first 60s.
- **L_lock:** Whether LP tokens are burned or locked for ≥ 30 days (boolean; for Pump.fun, whether bonding curve has graduated).

### 2.5 "Buyer acceleration"
- `B_T60` = count of independent buys in first 60s.
- `B_T30` = count in first 30s.
- **Acceleration** = `B_T60 / max(1, B_T30 × 2)` — a value > 1 means buying is *speeding up* in the second 30s, not slowing.

### 2.6 "Rug or honeypot event"
Any of the following within 24h of pool creation:
- LP removed > 80% within any single transaction
- Mint authority used to mint new supply > 1% of initial
- Freeze authority used on ≥ 5% of holders
- Sell transactions fail (honeypot) for ≥ 3 distinct wallets that successfully bought
- Token price drops > 95% from its 1h high with no recovery

**Labeling:** Every token in the historical set is manually or semi-automatically labeled as `RUG`, `HONEYPOT`, `NORMAL_DECAY`, or `SURVIVOR`. This labeled set is the ground truth.

### 2.7 "Expected 24h return (net)"
- **Entry price:** The volume-weighted average price of a hypothetical $500 buy executed at T+60s (or the smallest test size that fits the model), including priority fee.
- **Exit price:** The volume-weighted average price achievable for selling the position at its *actual exit trigger* (from §3), net of slippage modeled as a function of position size vs. observed pool depth.
- **Fees:** All Solana tx fees + priority fees + DEX fees + an assumed 1.5% aggregate slippage buffer.

---

## 3. Entry and exit rules (v1 — one strategy, one exit policy)

### 3.1 Entry rule (all must be true)

| # | Condition | Threshold |
|---|---|---|
| 1 | Passes all **hard risk vetoes** (§4) | All pass |
| 2 | `L_T60` | ≥ $25,000 |
| 3 | `L_growth` | ≥ 0.20 (≥ 20% growth in first 60s) |
| 4 | Independent buyers `B_T60` | ≥ 15 |
| 5 | Buyer acceleration | > 1.0 |
| 6 | Top-10 holder concentration (excl. LP, burn) | < 40% |
| 7 | Simulated sell of 5% of hypothetical position | Succeeds, slippage < 8% |

If all seven pass, **the token enters the paper portfolio** with a fixed position size (see §5).

### 3.2 Exit rule (single policy for v1)

Execute in this order — first trigger wins:

1. **Hard stop:** Position value ≤ −40% of entry → exit immediately (subject to sellability; if sell fails, mark as `STUCK` and trigger portfolio circuit breaker).
2. **Staged take-profit:** Sell 50% at +100%, sell 30% at +300%, leave 20% running.
3. **Volatility-scaled trailing stop** on remaining position: trail at `max(25%, 2 × ATR_5min)` from the highest price since entry.
4. **Liquidity deterioration:** If pool liquidity drops > 40% from entry-time liquidity in any 5-minute window → exit remaining position.
5. **Time exit:** At T+24h, exit any remaining position at market.

**No discretionary overrides in v1.** If the rules say exit, we exit. This is the entire point of the test.

---

## 4. Hard risk vetoes (short-circuit — score is irrelevant if any fail)

These are **binary**. A token failing any one is rejected, regardless of how good everything else is. This is the single most important design decision in the spec.

| Veto | Check |
|---|---|
| **V1. Mint authority** | Mint authority is null OR is a known-good multisig with on-chain timelock. If active and unverified → reject. |
| **V2. Freeze authority** | Freeze authority is null. If active → reject. |
| **V3. Honeypot simulation** | Simulated sell of the intended position size succeeds on-chain *at simulation time*. Failure → reject. |
| **V4. LP lock** | LP burned or locked ≥ 30 days. Unlocked → reject (unless Pump.fun pre-graduation, which has its own bonding-curve rules). |
| **V5. Deployer history** | Deployer wallet has not launched ≥ 2 prior tokens that rugged. 1 prior rug → flag but allow. 2+ → reject. |
| **V6. Concentration** | Top-10 holders (excluding LP, burn address, known CEX) < 40% of supply. |
| **V7. Seller diversity** | At least 3 distinct wallets have successfully *sold* the token (proves sellability beyond simulation). |

**Rule:** Any change to this list is a **spec revision**, logged with a date and reason. We do not silently add or remove vetoes to make results look better.

---

## 5. Position sizing and portfolio rules (for paper trading)

- **Base position:** Fixed $500 notional per qualifying token (paper).
- **Max concurrent positions:** 10.
- **Max daily entries:** 20.
- **Portfolio circuit breaker:** If cumulative paper drawdown exceeds 25% of starting paper capital → halt all new entries for 24h and require manual review.
- **No pyramiding, no re-entry** on the same token within 24h.

These are deliberately simple. Position-sizing optimization is a v2 concern; v1 tests whether the signal itself has edge.

---

## 6. Success criteria (pre-registered — write these down and do not change them after seeing data)

The hypothesis is **supported** if, over the validation window (≥ 4 weeks of live data, ≥ 100 qualifying signals):

| Metric | Minimum threshold |
|---|---|
| Net expected return per trade (after fees/slippage) | > +5% |
| Win rate (trades with net positive return) | ≥ 35% |
| Profit factor (gross wins / gross losses) | ≥ 1.5 |
| Max drawdown of paper portfolio | ≤ 25% |
| Rug/honeypot rate among *accepted* tokens | ≤ 2% |
| Sample size | ≥ 100 accepted trades |

The hypothesis is **rejected** if *any* of the following:

- Net expected return per trade ≤ 0%
- Rug rate among accepted tokens > 5%
- Fewer than 30 accepted trades in 4 weeks (signal too rare to be useful)
- Max drawdown > 40%

**Inconclusive** → extend by 4 weeks once, then decide.

**Critically:** These thresholds are set *now*, before data. Changing them after seeing results is how backtests become lies.

---

## 7. What we measure but do *not* optimize in v1

To prevent overfitting, we log but do not tune:

- Optimal entry time (30s vs 60s vs 120s)
- Optimal take-profit levels
- Optimal trailing-stop multiplier
- Wallet-quality signal contribution
- Regime-conditional performance

These become tuning targets in v2, tested on a *separate* held-out window.

---

## 8. Data requirements (Stage 0 deliverable)

Before a single signal is evaluated, we need:

- **≥ 90 days** of historical Solana pool creation + swap events for the target DEXes (Raydium, Orca, Meteora, Pump.fun).
- **≥ 90 days** of holder snapshots at T+0, T+60s, T+24h for every qualifying token.
- **≥ 90 days** of deployer wallet history (all prior launches, rug outcomes).
- **Labeled rug/honeypot dataset** — we will construct this by combining provider labels (where available), heuristic detection, and manual spot-check of a random 10% sample.
- **Replay capability** — the ability to feed historical data through the exact live-strategy code, not a parallel Python reimplementation.

If we cannot obtain the labeled rug dataset at sufficient quality, the hypothesis is untestable and we must solve data first. **Do not skip this.**

---

## 9. Deliverables at end of v1

1. **This spec, frozen** (git tag, dated).
2. **Labeled historical dataset** — versioned, shareable, reusable for v2.
3. **Replay harness** — one command, historical data in, trade log out.
4. **Live paper-trading log** — ≥ 4 weeks, every signal, every trade, every veto reason logged.
5. **Results report** — plain language: *did the hypothesis hold? By how much? What surprised us?*
6. **Go/no-go recommendation** with explicit reasoning against §6 criteria.

**No live capital is deployed until #6 says go, and the recommendation is reviewed by a second person.**

---

## 10. Known threats to validity (write these down now so we can't pretend we didn't see them)

1. **Survivorship in labeling** — if our rug dataset only contains rugs we *found*, we'll underestimate base rate. Mitigation: label a random sample of *all* tokens, not just suspicious ones.
2. **Simulation ≠ execution** — sell simulation at T+60s can succeed while the same sell at T+70s fails. Mitigation: model slippage conservatively and apply 1.5% buffer; treat paper P&L as an upper bound.
3. **Regime dependence** — a 4-week window may be an unusually good or bad meme market. Mitigation: report results split by week; require consistency, not just aggregate.
4. **Look-ahead leakage** — using T+90s data to make a T+60s decision. Mitigation: replay harness enforces a strict clock; every feature has a `knowledge_time` field.
5. **Selection of validation window** — if we start the 4-week clock *after* seeing initial results, we'll pick a favorable window. Mitigation: start the clock on a pre-committed date, before the first signal.
6. **Spec creep** — adding vetoes or rules mid-validation to fix a losing streak. Mitigation: any spec change resets the validation clock and requires a written justification.

---

## 11. Open decisions needing your sign-off

1. **Target DEXes for v1:** Raydium + Pump.fun only.
2. **Historical data provider:** Helius primary, Shyft backup.
3. **Validation window start date:** committed to calendar before any signal.
4. **Second reviewer:** named before go/no-go.
5. **Rug labeling budget:** in-house manual labeling with spot-check.

---

## 12. What this spec deliberately does *not* contain

- **No wallet-copying strategy** in v1. Wallet intelligence is v2.
- **No AI/ML model.** Rule-based only.
- **No multi-strategy.** One entry rule, one exit policy.
- **No commercial features.** No subscriptions, no user accounts.
- **No live execution.** Paper only.

These exclusions are the spec. If we can't make v1 work with this much scope removed, adding features won't save us.

---

## Sign-off

| Role | Name | Date | Signature |
|---|---|---|---|
| Hypothesis owner | | | |
| Second reviewer | | | |
| (Optional) Capital allocator | | | |

**This spec is frozen once signed. Any change requires a new version and a reset of the validation clock.**
