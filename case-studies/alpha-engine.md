# Alpha Engine — an options research platform that talked me out of trading

**Role:** sole designer, engineer, and researcher
**Duration:** ongoing since 2026
**Stack:** Python · pandas · NumPy · SciPy · ib_insync · FastAPI · Parquet · OPRA options data
**Code:** [github.com/bulevill/alpha-engine](https://github.com/bulevill/alpha-engine)

---

## The one-paragraph version

I built an end-to-end options research platform — signal generation, a backtester priced off
real historical option chains, a risk-managed execution engine wired to Interactive Brokers,
and a reporting layer — roughly 31k lines of Python. Then I used it to test four strategies.
Three came back negative or null. The fourth had a real edge that arrived with a −75%
drawdown, which fails the standard I set at the outset: *beat passive index investing by
enough to justify the added risk.* So none of them are funded. **The system's most valuable
output has been the hypotheses it killed**, and the discipline that made those kills
trustworthy is the thing I'd point an employer at.

---

## Why I built it

I'd been trading a discretionary dip-buying strategy by hand and wanted to know whether it
was skill or a bull market. That question — *is this edge real, or am I fooling myself?* —
turned out to be the whole project. Building something that executes trades is a weekend
problem. Building something whose answers you can trust is not.

---

## What I built

| Layer | What it does |
|---|---|
| **Signal research** | Kernel-regression bottom detection, regime filters (HMM), IV rank/percentile series built from raw chains, LLM-scored news sentiment as an offline signal |
| **Backtester** | Event-driven, prices options off **real historical OPRA bid/ask** at the actual strike and expiry — not Black-Scholes approximations. Split-aware, memory-bounded over 2020–2026 |
| **Validation** | Placebo controls, oracle controls, weekly block bootstrap CIs clustered by entry date, pre-registration ledger, single-use holdout |
| **Execution** | Autonomous IBKR loop: conviction sizing, per-sleeve −30% circuit breaker, liquidity gates, kill switch, paper-port enforcement |
| **Ops/reporting** | FastAPI dashboard, P&L snapshot log, trade reconciliation, automated daily digest |

Priced against real chains matters more than it sounds. A signal can look excellent on
underlying returns and still lose money once it pays the actual spread on the actual contract.
Two of my strategies died exactly there.

---

## Results, stated honestly

| Strategy | Verdict |
|---|---|
| Opportunistic (technical + catalyst scoring) | **Negative edge** — −99% on a fair full-sample universe |
| News → stock (102k LLM-scored articles) | **Clean null.** An oracle control run through the same harness printed +0.37 avgR, so the harness was working — the signal wasn't |
| Rebound LEAPS (kernel-bottom dip buying) | **Edge, fails the bar.** Beat SPY in the holdout (+43% vs +27%) at ~5× the drawdown |
| Earnings vol (short vol into high-implied prints) | **Real and placebo-clean** (+13.8%/trade) but dies on a bid/ask spread I don't have access to — breakeven needs a ~2.3% half-spread, and it's only ~42 events/year |

The success bar was written down before the results: *beat passive ETF investing by enough to
justify the higher risk.* Positive edge alone was explicitly not the bar. Nothing has cleared
it. Reporting that is the honest outcome, and it's why the live money isn't in.

---

## The part I'd actually want to talk about

### An AI-generated function was quietly reading tomorrow's price

Most of this codebase was written with AI coding assistants. That was a real productivity
multiplier and a real hazard, and here is the hazard in one line.

The rebound strategy backtested at **+30.9%/yr**. That number was good enough to make me
suspicious rather than happy, so I commissioned an independent review of the codebase — a
fresh model session, given two written briefs (verify the code; stress-test the statistics)
and an explicit mandate to find self-deception.

It found this:

```python
d1 = np.gradient(m)     # central difference: d1[t] = (m[t+1] - m[t-1]) / 2
```

The smoothing kernel `m` was causal — it only looked backward. Its **derivative** was not.
`np.gradient` uses central differences at interior points, so at every historical bar the
entry condition read `m[t+1]`, a function of **tomorrow's close**. A dip-buyer that secretly
knows tomorrow went up will look like a genius.

Two things made this genuinely nasty:

1. **The parity check passed vacuously.** I had a harness asserting the live signal matched
   the backtest signal. It compared the backtest function to the live function over the same
   array — the same function — so it always passed. "Bar-for-bar parity" was true and
   meaningless.
2. **Live traded a different strategy than anything I'd ever tested.** `np.gradient` falls
   back to a one-sided difference at the *last* bar, which is the only bar the live engine
   evaluates. So the deployed signal was a weaker estimator that had never appeared in any
   backtest.

### What I did about it

- **Fixed it** — three lines, a backward difference (`np.diff(m, prepend=m[0])`), applied to
  both copies of the function.
- **Retired every affected number rather than restating it.** +30.9%/yr, +46.9%/yr OOS,
  "2.2× SPY", Sharpe 2.38 — struck from the record, and the docs still say so. Every
  downstream sweep (exit rules, expiry selection, universe selection, IV gating) inherited
  the bias, because they all built their signal map from the same function, so all of them
  were invalidated too.
- **Re-ran once.** Frozen config, no re-tuning, one run — because a re-tune after seeing the
  result is how you launder a dead strategy back to life. The edge survived at roughly half
  size (avgR +0.253, 95% CI +0.148…+0.365) and the drawdown blew out from −44.5% to −74.6%.
  **Drawdown, not edge, became the binding constraint.**
- **Went looking for the same class of bug**, and found two more: the live engine was scoring
  an *incomplete, still-forming* daily bar during market hours, so completed-bar signals were
  never evaluated at all (the strategy was structurally unable to take its own validated
  entries); and exits were filling against stale prior-day quotes. Three bugs, one family —
  *which bar does this code think "now" is?* That question is now a standing check.

A later fix to the stale-fill bug pushed the measured edge down to nothing significant
(avgR +0.045, CI spanning zero) — which is what redirected the entire research program away
from entry signals and toward exits.

### What I took from it

- **Verify hardest when the output flatters you.** The bug survived months precisely because
  the result was the one I wanted.
- **AI-generated code fails in a specific way**: it's syntactically clean, idiomatic, and
  semantically wrong in a way that reads fine. `np.gradient` is the *correct* function for a
  derivative — it's the wrong function for a *causal* derivative, and nothing about the line
  looks alarming.
- **A test that can't fail isn't a test.** The parity harness gave me false confidence, which
  is worse than no confidence. I now ask what a check would have to see to fail.
- **Adversarial review works.** Bringing in a reviewer whose job was to find my errors — with
  a written brief, not a vague "look this over" — found in one pass what I'd missed all along.

---

## Engineering problems worth mentioning

- **The engine once ran blind for a full day.** A dead broker socket meant `holdings()`
  returned an empty book instead of raising, so the loop reported "no positions," silently
  wiped its own max-hold clocks, and evaluated exits it couldn't see — with two real positions
  open. Fixed by making the failure loud: connection retry with the loop *holding* while
  disconnected, and `holdings()` raising rather than reporting a false flat.
- **Delayed market data is adversarial, not just late.** A stop fired but never filled: its
  limit was priced off a stale quote, and the re-post logic compared against that same stale
  base, so it never chased. A position went from −21% to −73%. The executor now crosses the
  bid, flags placeholder quotes explicitly, and re-posts stranded closes.
- **~5% of my option-price noise was a data-layout artifact.** Raw chain files carry up to 18
  rows per contract-day — one per OPRA exchange. Taking an arbitrary row injects noise with no
  directional bias, which is the kind of error that quietly widens every confidence interval
  in the project. Consolidated on the median across venues.
- **A full-history chain rebuild that OOM'd** is now chunked, incremental, and resumable.

---

## What I'd do differently

- **Pre-register from day one.** I added the pre-registration ledger *after* a review pointed
  out that my "out-of-sample" number was really the selection window of dozens of sweeps with
  no multiple-testing correction. Every result before that is worth less than it looked.
- **Write the falsification test before the strategy.** I'd build the placebo and oracle
  controls first, and refuse to look at a headline number until they pass.
- **Treat data provenance as a first-class component.** Venue duplication, splits, and
  delayed-feed gaps each cost me more than any modelling decision did.
- **Budget for the spread earlier.** Two strategies died on transaction costs. Costs should
  have been in the first-pass model, not a later adjustment.

---

## Honest scope note

This is a personal research project, not a funded book. The numbers here are backtests and
paper trading. I'm including it because the interesting content isn't a return figure — it's
the audit trail of how a promising result got taken apart, and what was left standing.
