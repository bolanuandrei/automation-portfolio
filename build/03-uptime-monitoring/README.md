# Uptime monitoring that doesn't spam you at 3am

**Context:** 10 production endpoints across a single VPS — client sites plus the hosting control panel.
**Role:** Sole designer and implementer
**Stack:** n8n · HTTP checks · workflow static data · SMTP

---

## The problem

No monitoring at all. Outages were discovered by whoever happened to visit a
site first. Free tiers of hosted monitoring services cap out well below 10
endpoints, and the paid tiers cost more than the monitoring is worth for an
agency this size.

The second problem, which is the one most naive implementations get wrong:
a check running every 30 minutes against a site that stays down for 6 hours
generates 12 identical emails. After the third, you stop reading them.

**Constraints:**
- Zero additional monthly cost
- Must distinguish "server is down" from "web server is down" — different fixes
- Alerts had to remain readable during a sustained outage

---

## The approach

| Option | Why rejected / chosen |
|---|---|
| Hosted monitoring SaaS | Rejected — 10 endpoints exceeds free tiers |
| Simple loop with email on non-200 | Rejected — alert fatigue makes it worse than nothing |
| **Checks + per-URL cooldown state** | **Chosen — free, and alerts stay meaningful** |

**The alert suppression decision.** n8n exposes persistent storage scoped to
a workflow. I used it to hold a last-alerted timestamp per URL and suppress
repeat notifications inside a 30-minute window. This is the difference
between a monitor people trust and one they filter to a folder.

**Differential diagnosis, borrowed from network operations.** The endpoint
list includes the hosting control panel port alongside the sites. That one
addition turns a binary alert into a diagnosis:

| Symptom | Conclusion |
|---|---|
| All sites down, panel down | Server or network is down |
| All sites down, panel responds | Web server or CDN layer is the problem |
| One site down, rest fine | Application or configuration issue on that site |

The alert email states this explicitly, so whoever reads it at 3am doesn't
have to reason it out. This came directly from my background handling
network trouble tickets — the first job of an alert is to narrow the
search space, not just to say something is wrong.

---

## Implementation

1. Schedule trigger every 30 minutes
2. Endpoint list defined in code
3. HTTP request per endpoint with `fullResponse`, `neverError` and a 15s
   timeout — errors must become data, not workflow failures
4. Conditional check for non-200 status (timeout normalized to 0)
5. Cooldown step reads and writes per-URL state, decides whether to notify
6. Filter passes only suppression-cleared items
7. Plain-text email with status code, timestamp and the diagnostic table

**Key technical decision:** `neverError: true` on the HTTP node. Without it,
an unreachable host throws and the workflow aborts — meaning the one case
you built the monitor for is the case it can't report. Turning failures into
data is the whole point.

📄 `workflow.json` — importable, endpoints anonymized

---

## Result

| Metric | Before | After |
|---|---|---|
| Endpoints monitored | 0 | 10 |
| Check interval | — | 30 min |
| Emails during a 6h outage | would be 12 | 12 → suppressed to 1 per 30-min window |
| Monthly cost | — | 0 (vps server which is cheap) |
| Time to detect | hours to days | ≤ 30 min |

---

## What I'd do differently

There is no recovery notification — you learn a site is back by checking
manually. I'd add a state transition so the workflow emails on down *and* on
up, which also gives you outage duration for free. I'd also add a second
check from a different network location, because a single vantage point can't
distinguish "the site is down" from "my monitor's network path to it is down."
