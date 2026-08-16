# Event-Driven Forecasting for Enrollment Analytics

An event-driven forecasting system for enrollment analytics that reacts
to new data as it arrives instead of running on a fixed batch schedule,
improving both prediction accuracy and operating cost.

## Requirements

**Functional**
- The system produces an enrollment forecast reflecting the most recent
  available data, not data up to a full batch cycle stale.
- A forecast updates automatically when meaningfully new data arrives,
  without a manual trigger.
- Analysts/stakeholders can view the current forecast and how it's
  changed over time.

*Below the line:* multiple forecast models compared simultaneously;
long-range forecasting beyond the immediate enrollment cycle.

**Non-functional**
- **Cost should scale with actual data-arrival volume**, not run a
  fixed-cost job on a schedule regardless of whether there's anything
  new to process — the main cost-reduction lever.
- Forecast recomputation shouldn't be so frequent that it creates
  noisy, flapping predictions — stability matters as much as freshness.
- No need for dedicated always-on infrastructure for what's a
  relatively low, bursty volume of enrollment-related events.

## Input / Output & Data Flow

**Input:** new enrollment-related data points as they become available
(new applications, new registrations, changes to existing records).

**Output:** an updated forecast, available whenever meaningful new data
has arrived, rather than only once a day.

1. A new enrollment-related event occurs.
2. The event triggers a lightweight check: is this significant enough
   to warrant a forecast recompute, or should it be batched with other
   recent events first?
3. If a recompute is warranted, the forecasting model runs against the
   latest available data and produces an updated prediction.
4. The updated forecast is stored and published for consumption.

## High-Level Design

**Event-driven trigger** — *What:* a change to underlying enrollment
data triggers the forecasting pipeline directly, instead of a fixed
schedule running the same job regardless of whether anything relevant
changed. *Why event-driven over cron:* a fixed schedule pays the same
compute cost whether one record changed or a thousand did, and can't
reflect a big data change until the next scheduled run — event-driven
cost is proportional to actual activity, and freshness isn't bounded by
a schedule interval. *When a fixed schedule would still make sense:* if
underlying data changes so continuously that "an event" isn't a
meaningful unit — at that point a short-interval schedule is simpler
than debouncing a constant stream of triggers.

**Debounce/batching layer** — *What:* nearby events within a short
window are coalesced into one recompute rather than recomputing on
every single event. *Why:* enrollment data often arrives in small
bursts — recomputing per-event wastes compute repeating nearly the same
calculation, and produces a forecast that visibly jitters between
near-identical values.

**Serverless/on-demand compute** — *What:* the forecasting job runs
on-demand, triggered by the debounced event, rather than on dedicated
always-on infrastructure. *Why:* enrollment-related activity is bursty
and often low-volume outside specific periods — paying for always-on
infrastructure to handle an inconsistent, often-idle workload is the
opposite of matching cost to actual use.

**Forecast store with history** — *What:* each computed forecast is
stored with a timestamp, not overwritten in place. *Why keep history
rather than only the latest value:* a forecast that changes should be
inspectable — did it change because of a real shift in the data, or
does it look like noise — which requires seeing the trend, not just the
current number.

```mermaid
flowchart LR
    E[Enrollment Data Change] -->|event| DB[Debounce / Batching Layer]
    DB -->|triggers| F[Forecasting Job<br/>on-demand]
    F --> UF[Updated Forecast]
    UF -->|write| FS[(Forecast Store, with history)]
    FS --> A[Analysts / Dashboards]
```

## Deep Dives

Why these three: the interesting problems in event-driven forecasting
are avoiding wasted recomputation on noise, keeping forecasts stable
rather than flapping, and making sure cost savings don't come at the
expense of missing a genuinely significant data change.

### 1. How do you avoid recomputing on every trivial event?

Given the cost savings depend on not running unnecessarily.

**Simple approach:** recompute on literally every incoming event.
Sufficient? Technically freshest, but defeats the cost-efficiency
goal — many individual events don't meaningfully change what the
forecast should say, so paying full recompute cost for each is
wasteful.

**Better approach:** debounce events within a short window, plus a
lightweight significance check — does the incoming change plausibly
move the forecast — before triggering a full recompute. New problem: an
overly aggressive significance check risks skipping a genuinely
important change that doesn't match the heuristic's expectations,
silently making the forecast stale despite being "event-driven."

**What I'd pick:** debouncing as the primary, always-on mechanism
(cheap, low-risk), with the significance check as an optional secondary
filter only if debouncing alone doesn't hit the cost target —
debouncing alone captures most of the savings with much less risk of
silently missing something.

### 2. How do you keep forecasts from visibly jittering between near-identical values?

**Simple approach:** publish whatever the model outputs on every
recompute, unfiltered. Sufficient? No — small noise in the underlying
data produces small, meaningless swings that make the forecast look
unstable over time, undermining trust even if each number is
technically defensible.

**Better approach:** apply a smoothing step to published forecasts
(weighting a new prediction against recent prior predictions) rather
than publishing raw model output directly. New problem: smoothing
introduces lag — a genuinely large, real shift takes longer to fully
show up in smoothed output than in the raw one.

**What I'd pick:** light smoothing with a threshold that lets a
sufficiently large raw change bypass the smoothing — pure smoothing
trades real signal for stability; a threshold-based bypass keeps
stability for noise while still surfacing genuinely large moves
quickly.

### 3. How do you verify the cost savings aren't hiding a missed, significant enrollment shift?

**Simple approach:** trust that debouncing/significance filtering is
working correctly and move on. Sufficient? Not verifiable on its
own — the entire risk of an event-driven, cost-optimized system is a
filtered-out event that should have triggered a recompute, and that
failure mode is invisible unless specifically checked for.

**Better approach:** keep a low-cost, infrequent (e.g. daily) baseline
recompute running as a backstop alongside the event-driven path, and
compare its output against the event-driven forecast's latest value — a
meaningful divergence is a signal something the event-driven path
should have caught, didn't.

**What I'd pick:** an infrequent baseline recompute purely as a
correctness check, not the primary forecasting path — cheap relative to
a full always-on schedule (so the switch to event-driven still nets a
large cost reduction) while giving a concrete way to catch the specific
failure mode this design is most at risk of.

## Wrap-Up

Final flow: enrollment data changes triggering a debounced, on-demand
forecasting job, smoothed output published to a forecast store with
history, with an infrequent baseline recompute running alongside as a
correctness backstop.

With more time: tune the debounce window and smoothing threshold
against real historical data rather than an initial guess; add
alerting when the baseline and event-driven forecasts diverge beyond a
threshold; evaluate whether the significance-check heuristic from Dive
1 is worth adding, or whether debouncing alone was already sufficient.
