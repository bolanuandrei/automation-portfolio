# Turning a quiz form into a routed lead pipeline

**Context:** Coaching business running a quiz-style lead magnet on a WordPress site.
**Role:** Sole designer and implementer
**Stack:** n8n · webhooks · AcyMailing REST API · Fluent Forms · PRO SMTP

---

## The problem

Quiz submissions landed in the form plugin's database and nowhere else.
Adding people to the mailing list was manual. Spotting the ones who had
ticked "I'd like a 15-minute call" meant reading through entries — and those
are the highest-intent leads, so a slow response is the most expensive kind.

**Constraints:**
- The mailing platform is a WordPress plugin with no n8n integration
- Two outcomes needed from one submission: list subscription always,
  owner notification only for call requests
- The business owner is not technical — no new tool to learn

---

## The approach

| Option | Why rejected / chosen |
|---|---|
| Native integration node | Not available — the platform has no n8n node |
| Migrate to a supported email platform | Rejected — client had an established list and workflow |
| CSV export and manual import | Rejected — that's the problem, not the fix |
| **Direct REST API calls against the platform's documented endpoints** | **Chosen — no migration, no new tooling for the client** |

**Parallel branches, not sequential.** List subscription and owner
notification are independent outcomes. Chaining them would mean a failure in
one blocks the other — and the notification is the time-critical half.
Running them in parallel from the webhook means a mailing platform hiccup
never costs a hot lead.

**Reply-to routing:** the notification email sets reply-to as the lead's own
address. The owner replies from her inbox and it reaches the lead directly,
with no copy-paste step. Small detail, removes the last manual action in the chain.

---

## Implementation

1. Webhook receives the form POST
2. Two parallel branches:
   - **Always:** create-or-update the contact, then subscribe to the target list
     (two calls — the API separates identity from subscription)
   - **Conditionally:** if the call-request checkbox is affirmative, email the
     owner with name, email and phone, reply-to set to the lead
3. Conditional check uses `contains` rather than strict equality, since form
   plugins serialize checkbox values inconsistently

**Key technical decision:** create-or-update before subscribe, as two calls
rather than assuming one endpoint does both. A returning lead
who retakes the quiz updates cleanly instead of erroring on a duplicate.

📄 `workflow.json` — importable, API key, endpoint and client email removed

---

## Result

| Metric | Before | After |
| List subscription | manual | instant, automatic |
| High-intent lead notification | on review | within seconds |
| Manual steps per lead | 20 | 0 |
| Duplicate handling | error-prone | repeat-safe |

---

## What I'd do differently

No error handling on the API calls beyond the global handler — if the mailing
platform is down, the subscription is lost silently and the lead is never
added. I'd add a retry with backoff, and a fallback that writes failed
submissions to a spreadsheet so nothing is lost when a dependency is
unavailable. Cheap to add, and it's the difference between "usually works"
and "doesn't lose data."
