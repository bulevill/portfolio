# Portfolio

Longer write-ups of what I've built — the reasoning, the results, and what went wrong.
Each case study is the version I'd want to be asked about in an interview, not a feature list.

---

## Case studies

| Project | What it is | Outcome | Code |
|---|---|---|---|
| **[Alpha Engine](case-studies/alpha-engine.md)** | Options research platform: signal generation, a backtester priced off real historical OPRA chains, and a risk-managed IBKR execution engine (~31k lines of Python) | Four strategies tested: two retired on the evidence, two still standing and blocked on named constraints — drawdown and execution access. **Nothing funded yet; the research is active.** | [repo](https://github.com/bulevill/alpha-engine) |

| **[Dopplerganger](case-studies/dopplerganger.md)** | Acoustic geolocation: matches aircraft Doppler curves in a recording against US ADS-B flight data and terrain to estimate where it was recorded | 19/24 located on realistic renders from real flight tracks; detector 22/31 on held-out real flyovers. No field recording located to date; 0.8% of US land gives a 30-min recording a ≥ 50% chance of two usable passes. | [repo](https://github.com/bulevill/dopplerganger) |

<!-- Add new rows here -->

---

## How to read these

Every case study follows the same shape, so you can skip to what you care about:

- **The one-paragraph version** — the whole thing, if that's all the time you have
- **What I built** — scope and the design decisions I'd defend
- **Results, stated honestly** — including the parts that didn't work
- **The part I'd actually want to talk about** — one concrete incident, in depth
- **What I'd do differently** — real hindsight, not ritual humility

I include failures and dead ends on purpose. A portfolio where every project succeeded tells
you nothing about how someone handles the case where it doesn't — which is most cases.
