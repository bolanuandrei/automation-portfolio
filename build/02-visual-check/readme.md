# Catching broken layouts before clients do

**Context:** Small web agency maintaining 10 production WordPress sites across shared infrastructure.
**Role:** Sole designer and implementer
**Stack:** n8n · ApiFlash · Google Gemini (vision) · SMTP

---

## The problem

Plugin updates, theme changes and cache misconfiguration break layouts
silently. HTTP monitoring returns 200 on a page whose stylesheet failed to
load — the server is healthy, the page is unusable. We found out when a
client called, which meant the problem had already been live for hours or days.

**Constraints:**
- No budget for a commercial visual regression platform
- Traditional pixel-diff tools produce constant false positives on sites with rotating content, dynamic banners and A/B tests
- Had to work across 10 sites on different builders (Divi, Bricks)

---

## The approach

Pixel-diffing was the obvious answer and the wrong one. These sites change
legitimately every week — a diff-based tool would have cried wolf until we
stopped reading the alerts.

| Option | Why rejected / chosen |
|---|---|
| Pixel-diff against a baseline | Rejected — dynamic content makes every run a false positive |
| Commercial visual testing SaaS | Rejected — per-site pricing, more capability than needed |
| Synthetic browser tests with assertions | Rejected — brittle selectors, high maintenance across 7 different themes |
| **Screenshot + multimodal LLM judgment** | **Chosen — evaluates "does this look broken" the way a human would, tolerant of legitimate change** |

The insight: I didn't need to detect *change*, I needed to detect *breakage*.
Those are different questions, and a vision model can answer the second one.

**Prompt engineering was the actual work.** The first version flagged every
site with a lazy-loaded image as broken. The working version explicitly
instructs the model to ignore missing images when the overall layout holds,
ignore text content entirely, and return a strict JSON object rather than
prose. Constraining the output shape is what made it usable downstream.

Model choice: a lightweight vision model rather than the largest available.
The task is coarse classification, not fine analysis, and cost per run
matters when it multiplies across sites and weeks.

---

## Implementation

1. Weekly schedule trigger, Saturday 06:00 (before business hours, after
   the week's changes have landed)
2. Site list defined in code — a deliberate choice over a database, since
   it changes rarely and lives with the workflow
3. Loop per site, sequential rather than parallel, to stay inside API rate limits
4. Full-page screenshot at 1920px width via screenshot API
5. Vision model classifies the image, returns `{"broken": bool, "reason": str}`
6. Conditional branch on the parsed boolean
7. On failure: email alert with the reason and the screenshot attached

**Key technical decision:** the model returns structured JSON, and the parse
step strips markdown code fences before parsing. LLMs wrap JSON in fences
unpredictably; handling that is not optional in production.

📄 `workflow.json` — importable, API key removed

---

## Result

| Metric | Before | After |
|---|---|---|
| Detection method | client complaint | automated weekly check |
| Sites covered | 0 | 10 |
| Manual review time | 40 min/week | 0 |
| False positives after prompt tuning | — | 1 |
