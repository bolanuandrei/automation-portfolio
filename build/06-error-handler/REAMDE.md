# One error handler wired into every automation

**Context:** A growing set of production automations across two organizations
and several client accounts.
**Role:** Sole designer and implementer
**Stack:** n8n error trigger · SMTP

---

## The problem

Automations failed silently. A scheduled workflow that errors at 6am doesn't
announce it — you find out when the thing it was supposed to do didn't happen,
often days later. The failure mode for automation isn't a crash, it's absence.

**Constraints:**
- Must not require adding error handling inside every workflow
- Must work for scheduled workflows with no one watching

---

## The approach

n8n lets each workflow nominate an error workflow. Rather than adding
notification logic to every automation, I built one handler and referenced it
from all of them.

| Option | Why rejected / chosen |
|---|---|
| Try/catch branches inside each workflow | Rejected — duplicated logic, and easy to forget on the next one |
| Check execution logs periodically | Rejected — manual, and nobody does it consistently |
| **One error workflow, referenced globally** | **Chosen — write once, applies everywhere, including future workflows** |

This is a small artifact with disproportionate value. It's also the piece
that separates automations that run from automations that are *operated* —
and it's the first thing I now build in any new n8n instance, before the
workflows themselves.

---

## Implementation

1. Error trigger fires on any failure in any workflow that references it
2. Email sent with the failing workflow's name and the error message

Every other workflow in this portfolio sets this as its designated error
workflow. Adding a new automation means one dropdown selection, not new code.

📄 `workflow.json` — n8n template available

---

## Result

| Metric | Before | After |
|---|---|---|
| Failure detection | by absence of outcome, days later | email within seconds |
| Workflows covered | 0 | all |
| Per-workflow setup cost | — | one setting |

---

## What I'd do differently

The email contains the workflow name and error message but not the execution
URL, so diagnosing means finding the run manually. I'd add a direct link, plus
the node that failed and the input that caused it. I'd also add throttling —
a workflow triggered every 30 minutes that starts failing consistently will
produce an alert every 30 minutes, which is the same alert-fatigue problem I
solved in the uptime monitor and didn't solve here.
