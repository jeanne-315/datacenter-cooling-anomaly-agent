# Data Center Cooling Anomaly-Alerting Agent

**A data-center cooling anomaly agent — when it alerts, engineers can trust it's real.**

Status: 🚧 Full-year analysis and the core detection pipeline are implemented; interactive demo in progress. *Portfolio prototype on public ORNL Frontier data — not yet deployed.*

---

## The problem

Most monitoring systems can tell you a number looks unusual. Very few can tell you whether that's worth waking someone up for.

That gap matters more than it used to. NVIDIA's Vera Rubin generation runs 100% liquid cooling with coolant entering at 45°C — hot enough that facilities can reject heat with outdoor dry coolers and remove the chiller entirely. Chiller-based cooling typically accounts for around 40% of a data center's electricity, so the savings are substantial.

But removing the chiller also removes the slack. When a chiller is in the loop, a degrading pump or a fouled heat exchanger gets absorbed as "slightly more electricity." Without it, the facility is directly exposed to loop efficiency and outdoor conditions.

**The industry is moving toward an architecture that is more efficient and less forgiving. That architecture needs a monitoring layer that engineers actually trust.**

This project builds that layer, using a full year of real telemetry from a facility already running warm-water cooling.

---

## Who this is for

Data center facility engineers. The design goal is not maximum detection sensitivity — it is that **when the system raises an alert, the engineer on shift can trust it's real.** On a plant floor you can't switch cooling alarms off; the failure mode isn't a muted alert, it's an engineer who has quietly stopped believing the ones that keep crying wolf. So the system is explicit about what it *cannot* judge, and it earns each interruption.

---

## Data

[Energy dataset of Frontier supercomputer for waste heat recovery](https://www.nature.com/articles/s41597-024-03913-w) — Oak Ridge National Laboratory, published in *Scientific Data* (2024), licensed under CC BY 4.0.

49,869 records · full year 2023 · 10-minute resolution · 18 fields covering IT power, cooling power, three coolant sub-loops (flow, return temperature, waste heat), overall supply temperature, and PUE.

### Why this dataset — and why I rejected another

I evaluated a peer-reviewed smart building energy dataset first. It had cooling data, six years of it, and a real chiller plant. I rejected it.

Building energy is driven by envelope, solar gain, and occupancy. Data center energy is driven by IT load, with weather entering only through the cooling side. The physical profiles are different enough that patterns learned on one would be actively misleading on the other.

**Domain knowledge is what let me discard a dataset that looked fine on paper.**

### Five-gate validation

| Gate | Question | Result |
|---|---|---|
| Source | Who published it? Peer reviewed? | ✅ ORNL, *Scientific Data* |
| License | Can I use this publicly? | ✅ CC BY 4.0 |
| Fields | Are power, cooling, and timestamps present at usable resolution? | ✅ 18 fields, 10-min |
| Completeness | What's missing, and is it random? | ✅ 448.5 hours missing — **not random** |
| Detectability | Are there events visible in the data at all? | ✅ Confirmed |

The last gate is the one most projects skip. A dataset can be clean, well-documented, and completely unable to support the thing you want to build.

---

## What the data actually said

![Overview](figures/fig1_overview.png)

### 1. PUE and Total Power are derived fields, not measurements

`Total Power − (Compute + Facility accessory)` has a residual of **exactly zero**. So does `Total ÷ Compute − PUE`.

These aren't well-calibrated measurements. They're arithmetic. Which means **they cannot be used to detect sensor faults** — the relationship holds by construction, always.

The independent measurements are: three sub-loop temperatures and flows, overall supply temperature and flow, compute power, and facility accessory power. Everything else is downstream of those.

### 2. The missing data is not random — and it coincides with an operational change

![Data gaps](figures/fig3_gaps.png)

448.5 hours are missing across 303 gaps. But **86% of it falls in May and June**, with a single continuous 63.8-hour gap from May 26–29. Other months are near-complete.

### 3. A cooling regime change in late April

![Regime change](figures/fig2_regime_change.png)

| | Jan–Apr | May–Dec |
|---|---|---|
| Loop1 flow | ~780 gpm | ~1,450 gpm |
| Loop3 flow | ~1,500 gpm | ~2,900 gpm |
| Loop1/2 return temp | ~20°C | ~30°C |
| **Loop3 return temp** | ~36°C | **~31°C** |
| Loop1/2 waste heat | 0.7 / 0.9 MW | 2.0 / 2.1 MW |
| **Loop3 waste heat** | **7.1 MW** | **4.65 MW** |

Before April, Loop3 carried 7.1 MW of an ~8.8 MW total load while the other two idled. After, flows roughly doubled and heat redistributed to 2.0 / 2.1 / 4.65.

**This is the finding that shaped the entire product.**

A baseline fit across the full year would average two genuinely different operating modes into one that matches neither — and then flag roughly half the year as anomalous. This is the most common way anomaly detection projects die in production: not from a weak model, but from a baseline that quietly stopped being true.

So the system segments baselines by regime, and treats "the baseline may need to be rebuilt" as its own detectable condition.

> Note on causality: the data gap and the regime change overlap in time. That is suggestive of a single maintenance or retrofit event, but the dataset does not establish which caused which — or whether both share a third cause.

### 4. IT load is queue-driven and largely unpredictable

I expected seasonality. There isn't any meaningful amount.

| Factor | Variance explained |
|---|---|
| Month | 5.20% |
| Hour of day | 0.84% |
| Day of week | 0.73% |
| All three combined | 21.90% |

Weekends run *higher* than weekdays (11.42 vs 10.82 MW median). Load peaks at 2 AM and dips at 10 AM. Autocorrelation decays from 0.88 at 10 minutes to 0.16 at one day. 12.33% of samples show a step change greater than 2 MW.

Flat, then sudden jumps — the signature of jobs entering and leaving a scheduler queue, not of any continuous driver.

**Design consequence:** you cannot build a *time* baseline for this system. You can only build a *conditional* one — "given this IT load, how should the cooling side behave." Which is why the primary indicator is a ratio: **waste heat ÷ IT power**, median 0.808. Load is divided out; what remains is efficiency.

### 5. Higher supply temperature correlates with *higher* cooling power

Controlling for IT load, a 1°C rise in supply temperature is associated with cooling power going **up** by 0.31–1.07%. The industry rule of thumb says raising the setpoint 1°C *saves* around 4%.

Both are correct, because they describe different architectures. In a chiller plant, supply temperature is a **setpoint** — raise it and the compressor works less. In warm-water cooling with no chiller, supply temperature is a **weather outcome**. Hot day → dry coolers struggle → fans and pumps draw more → supply temperature and cooling power rise together.

The correlation is real. The causation runs through a third variable.

> Limitation: outdoor temperature and humidity are not in this dataset, so weather cannot be properly separated. Observed range is 7.3–33.6°C — **extrapolating to NVIDIA's 45°C is not supported** by this analysis.

---

## The one I was proudest of — and then removed: November 19

![Nov 19 event](figures/fig4_nov19_event.png)

For most of this project, a dramatic efficiency drop on **November 19th** was the centerpiece — a clean, diagnosable anomaly, an entire section of its own. It made the perfect demo.

Then I unified the load-stability standard across both detectors and tightened it to compare **consecutive** readings rather than just the endpoints of an event. November 19th failed the stricter test. Its Major segment (00:20–01:50) lined up exactly with a **single-step +38% surge in compute load at 01:00**, with waste heat falling in step — the signature of a load change, not a cooling fault. This behaviour reclassifies as **LOAD_CONFOUNDED**, and the verdict holds across every load-stability threshold from 5% to 25%.

So I removed it. My flagship example was a false positive, and a system whose whole promise is *"when it alerts, it's real"* cannot be introduced with an alert that isn't. I would rather lose my best demo than ship a system that cries wolf — and I hold myself to the same standard I'm selling.

What the episode still teaches stays in the design: **overall efficiency tells you *that* something looks wrong; per-loop comparison and a load-stability check together tell you *whether it's real*.** And **"return temperature below supply temperature" remains a deterministic check** — physically impossible, so its occurrence is information by itself, no model or threshold required. Probabilistic methods drive the false-negative rate down; deterministic checks drive specific failure rates to zero. A serious system uses both, and knows which is which.

---

## Product decisions

### What counts as an anomaly

> **A significant deviation of liquid-cooling heat capture efficiency from the normal range of the current operating regime, within a valid load range.**

I considered three definitions:

- **Statistical outlier** — rejected. 2,731 samples have a ratio below 0.5, but most are the machine sitting idle, where a small denominator makes the ratio meaningless. This definition flags shutdowns as emergencies.
- **Baseline deviation** — collapses into the next one. Because load is unpredictable (Finding 4), any usable baseline must be conditional on load — and a load-conditioned baseline *is* an efficiency measure.
- **Efficiency deviation** — adopted. It matches what a facility engineer actually wants to know: is the cooling system doing its job, given what's being asked of it.

### Three evaluation layers

**Layer 0 — Sensor health (deterministic; these are anomalies in themselves)**

| Pattern | Count |
|---|---|
| Ratio > 1.0 (waste heat exceeds IT power — violates energy balance) | 1,843 (3.7%) |
| Loop1 return < supply | 273 |
| Loop2 return < supply | 274 |
| Loop3 return < supply | 37 |
| Return temperature NaN | 122 |

These are classified separately from equipment faults, because the response differs completely: one sends someone to the equipment, the other sends someone to the instrument.

**Layer 1 — Interpretability gate (not an anomaly; an admission)**

Below **6 MW IT load**, the efficiency ratio is not interpretable:

![6 MW threshold](figures/fig5_threshold.png)

| IT load | Ratio median | Coefficient of variation |
|---|---|---|
| 0–2 MW | 0.152 | **181%** |
| 4–6 MW | ~0.27 | **142–146%** |
| **6–8 MW** | **0.799** | **17.1%** |
| 8–16+ MW | 0.80–0.82 | 15–21% |

The threshold comes from the data, not from a round number. Above 6 MW the ratio is stable; below it, variance explodes. (The 4–6 MW band contains only 16 samples all year — the machine is either running or essentially off.)

When this gate trips, the system reports **"cannot assess: load below interpretable range"** rather than staying silent or guessing. Same for data gaps and regime transitions.

**Layer 2 — Efficiency deviation (probabilistic; confidence-scored and tiered)**

Two complementary detectors, because the two failure modes look nothing alike:

- **Detector A — sudden jumps.** A sharp step in efficiency, filtered by a consecutive-sample load-stability check so that a jump in *load* is never mistaken for a jump in *cooling*.
- **Detector B — persistent drift.** A sustained efficiency deviation that never trips a single dramatic threshold, gated by the same load-stability standard.

Unifying that load-stability standard across both detectors is what caught the November 19 false positive above.

---

## Alerting: earning the right to interrupt

Tiers follow **ISA-18.2**, the industrial alarm management standard, rather than an invented scheme.

| Tier | Meaning | Response | Here |
|---|---|---|---|
| Critical | Imminent outage risk | Immediate | Single-sample trip, absolute physical criteria |
| Major | Significant degradation | 15 min | Consecutive-sample confirmation |
| Minor | Developing condition | 1 hour | Daily digest |
| Informational | Awareness only | — | Logged, retrievable |

### The dividing line

> **Major is a change of state. Minor is a change of degree.**

A chiller failure doesn't present as a reading that's slightly high. It presents as a reading that stops, or falls off a cliff. Continuous drift — fouling, gradual efficiency loss — is Minor by nature.

### The sampling interval sets the ceiling

Data arrives every 10 minutes, so "immediate" cannot mean faster than that. This forces a clean split: **Critical must trip on a single sample** — so the condition has to be one that cannot plausibly be a false positive — while **Major confirms across consecutive samples** (the "it looked wrong, and 10 minutes later it still looks wrong" test).

**Critical thresholds (absolute physical events — robust under parameter sweeps):**

| Condition | Triggers per year |
|---|---|
| Any loop return > 50°C | 1 |
| Flow = 0 under valid load | 0 |
| Waste heat = 0 under valid load | 1 |

50°C rather than 45°C: Loop3 return has a 99.9th percentile of 45.5°C, so a 45°C threshold fires 92 times a year on normal operation. At 50°C it fires once — a 65.9°C spike that warrants human eyes regardless of whether it's real heat or a failed sensor.

### The headline number — stated honestly

After unifying the load-stability standard, the year's persistent efficiency anomalies converge to **12 clean, load-stable, diagnosable events** — down from 66 before the standard was unified, a **78% cut** that included my own November 19 flagship.

Two honest caveats travel with that number, because the whole point of this system is that you can trust what it tells you:

- **The count is a function of the threshold, not an absolute truth.** At a 10% load-stability standard the year yields 12 events; at 5% / 15% it yields 9 / 19. There is no flat plateau. **I publish the method and the threshold, not a magic number.**
- **Some of the ~40 removed load-drop events may be real faults.** A cooling fault can itself cause a node to de-load, so cause runs both ways and this dataset cannot separate them. Removing them is an informed trade-off, not a pure win.

### The real data was quiet, not noisy

Across the full year the real-time channel would fire roughly **18 times (~0.05 per day)**, and the alert budget stays dormant at any reasonable cap. The "alert fatigue" problem this kind of product usually pitches against barely exists on this dataset — if anything the challenge is the opposite: don't miss the rare real event, and be honest when you can't assess. I'm reporting that plainly rather than keeping a tidier pitch, because a precision-first system *should* stay quiet unless it's sure.

### Alert budget

The real cost of a real-time channel isn't the interruption. It's that an engineer woken three times for nothing will mute the fourth alert permanently.

So real-time alerts have a **daily cap**. Anything beyond it is demoted to the digest, and the digest states plainly: *N alerts were suppressed today by the alert budget.* A system that quietly drops things is worse than one that never alerts at all.

### Telling "no reading" apart from "nothing running"

A dead sensor and a stopped pump both produce silence, and they need opposite responses.

The April shutdown made the discriminator obvious. During those 122 records: all three loop temperatures still read normally, flow and waste heat sat at exactly zero, and IT power had dropped to 0.85 MW. Instruments were alive; the equipment was off.

| Situation | Signature | Tier |
|---|---|---|
| Planned shutdown | Sensors reading, values at physical zero, **IT drops in step** | Informational |
| Instrument failure | One channel NaN, **other channels on the same loop normal** | Major |
| Communication loss | **All channels from a loop vanish at once** | Major |
| Equipment failure | Sensors reading, values collapse, **IT load still high** | Major |

**The key discriminator is whether IT load drops with the cooling.** In a planned shutdown they fall together. In an equipment failure, the load is still running and the cooling is gone — which is the scenario that matters.

---

## What I got wrong, and what remains unverified

**My own flagship anomaly.** November 19 (above) was going to headline the whole project until a stricter, unified load-stability test showed it was load-confounded. Catching it meant deleting my best example — which is exactly the discipline the product is built on.

**A predicted seasonality that wasn't there.** I expected higher load in winter. Summer runs higher, and load turned out to have almost no time structure at all. Finding 4 exists because that prediction failed.

**An approach that didn't survive contact with the data.** I tried using time-to-threshold — extrapolating temperature rise rate to estimate when a limit would be breached, standard practice in industrial alarming. On this data it produces roughly 5 alerts per day at a one-hour horizon. Return temperature fluctuates fast enough that linear extrapolation over an hour is mostly noise, and the measurement itself contains spikes (Loop2 maxes at 65.9°C against a 99.9th percentile of 43.3°C). Extrapolating from a contaminated signal compounds the error. Abandoned.

**Relative statistical criteria are fragile; absolute physical ones are robust.** Under parameter sweeps, the absolute checks (a temperature crossing a hard limit; a large efficiency drop) barely move, while percentile- and z-score-based rules swing widely. The trusted-alert tier rests on the robust criteria; the fragile ones are demoted to triage. A count like "12 events" is only meaningful with its threshold attached — which is why I attach it.

**No ground truth for single-loop failure.** The dataset contains no instance of one loop failing while the facility runs under load — the only zero-flow records belong to the April shutdown. The discrimination rules above are grounded in physics, not validated against labelled examples.

**No labels at all, in fact.** This is unlabelled anomaly detection, so conventional precision and recall are not computable, and I make **no claim of a specific accuracy or false-positive percentage** — that number would be unverifiable, and claiming it would break the whole premise. Verification currently rests on physical consistency, event-level inspection, and comparison against published work on the same dataset. Making that evaluation approach rigorous is open work, and stating the limitation plainly is part of the design.

---

## Roadmap

**v1 (in progress)** — segmented baselines, three-layer evaluation, confidence tiering, evidence packages, and a conversational layer for asking why a call was made.

**v2** — NOAA outdoor conditions to separate weather from loop efficiency; dry-cooler-specific analysis as chillerless architectures spread; extension to manufacturing facilities, where I have prior domain experience.

---

## Attribution

Data source: *Energy dataset of Frontier supercomputer for waste heat recovery*, **Scientific Data** (2024), Oak Ridge National Laboratory. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). https://www.nature.com/articles/s41597-024-03913-w

Related work on the same dataset: [Machine Learning Guided Cooling Optimization for Data Centers](https://arxiv.org/html/2601.02275v1) — which pursues optimization; this project pursues detection and human triage.

Industrial alarm management practice follows ISA-18.2.
