# SLOs

## What an SLO is

A Service Level Objective defines a measurable reliability target for a service. Dynatrace calculates SLO compliance continuously and can alert when burn rate is too high.

```
SLO = Success rate ≥ 99.9% over a rolling 30-day window
```

---

## Create an SLO

`Observe & Explore → SLOs → Add new SLO`

### Availability SLO (HTTP success rate)

```
Name:       Checkout service availability
Type:       Service availability
Target:     99.5%
Warning:    99.9%
Window:     Rolling 30 days

Entity:     Service — checkout
Metric:     builtin:service.errors.total.rate
            (inverted: 1 - error_rate)
```

### Performance SLO (response time)

```
Name:       Checkout P95 < 500ms
Type:       Service performance (custom)
Target:     95%
Window:     Rolling 7 days

Success criterion (DQL):
  fetch metrics
  | filter metric.key == "builtin:service.response.time"
    and dt.entity.service == "SERVICE-XXXXXXXX"
  | summarize pct = percentile(value, 95)
  | fieldsAdd success = pct < 500000   -- microseconds
```

---

## SLO metric selectors

```
# Availability — % of requests without errors
(100)*(builtin:service.requestCount.total - builtin:service.errors.total.count)
/(builtin:service.requestCount.total)

# Synthetic availability
builtin:synthetic.browser.availability.location.total

# Custom success rate from log metric
(ext:log.success_count)/(ext:log.request_count)*100
```

---

## SLO error budget

Dynatrace calculates error budget automatically:

```
Error budget = (1 - target) × window duration
Remaining    = budget - time already in violation

Example: 99.9% target over 30 days
  Total budget:   30 days × 0.1% = 43.2 minutes
  If violated 20 min so far → 23.2 min remaining
```

---

## Burn rate alerts

Burn rate = how fast you're consuming your error budget relative to a sustainable rate.

| Burn rate | Meaning |
|---|---|
| 1× | Consuming budget at exactly the sustainable rate |
| 5× | Will exhaust budget in 20% of the window |
| 14.4× | Will exhaust 30-day budget in ~50 hours (page now) |

Set up burn rate alerts in the SLO configuration:

```
Fast burn alert:   14.4× burn rate for 5 minutes → page on-call
Slow burn alert:   6×   burn rate for 30 minutes → ticket
```

---

## View SLOs in dashboards

SLO tiles are a built-in dashboard widget:

```
Add tile → SLO tile → Select your SLO → Configure display
```

Shows: current compliance %, trend, error budget remaining, burn rate.

---

## API — manage SLOs

```bash
# List all SLOs
GET /api/v2/slo

# Get a specific SLO
GET /api/v2/slo/<slo-id>

# Get SLO error budget
GET /api/v2/slo/<slo-id>?from=now-30d
```
