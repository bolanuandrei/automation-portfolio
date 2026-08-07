# Weekly content syndication without duplicate posts

**Context:** Health and wellbeing platform publishing articles that also needed to appear on its Google Business Profile.
**Role:** Sole designer and implementer
**Stack:** n8n · RSS · LLM summarization · Google Business Profile API

---

## The problem

Every published article needed a separate, shorter, differently-structured
post on Google Business Profile. Copy-pasting the article doesn't work — the
platform has a character limit and rewards a different format. So each post
meant someone rewriting the article as a summary with a hook and a call to
action. It got skipped whenever the week was busy, which was most weeks.

**Constraints:**
- Posts must never duplicate — GBP has no built-in deduplication
- Character limit on the platform
- Tone had to stay consistent with the brand's editorial voice
- The publishing account manages the profile on behalf of the organization

---

## The approach

| Option | Why rejected / chosen |
|---|---|
| Post the article excerpt directly | Rejected — wrong format, no hook, poor engagement |
| Track posted articles in a spreadsheet | Rejected — a second source of truth that can drift |
| **Deduplicate against the platform's own post history** | **Chosen — the API is the source of truth, nothing to keep in sync** |

**The deduplication decision is the interesting part.** Instead of keeping a
local record of what's been posted, the workflow fetches recent GBP posts and
checks whether each candidate article's URL already appears there — in the
call-to-action link *or* in the post body, since either could contain it.
URLs are normalized for trailing slashes before comparison, because the RSS
feed and the API don't agree on that detail.

The effect: the system has no memory to corrupt. Delete a post manually and
it becomes eligible again. Restore from backup and nothing breaks. State
lives in one place, owned by the platform.

**Selection logic:** walk the 10 most recent articles newest-first, publish
the first one not already posted, stop. One post per run, always the newest
unpublished item, and the loop naturally backfills if a week was missed.

---

## Implementation

1. Weekly schedule trigger, Monday 07:00
2. Read site RSS feed, limit to 10 most recent
3. Fetch recent posts from Google Business Profile
4. Deduplication logic selects the first unposted article, or returns empty
   to halt the run
5. Fetch full article content
6. LLM generates a platform-optimized post from a detailed brief covering
   role, tone, hook, structure, CTA and hashtags
7. Publish via API with a "Learn more" call to action linking the article

**Key technical decision:** the article fetch bypasses the CDN by requesting
the origin directly with an explicit `Host` header. Fetching through the
public URL meant the automation was hitting its own site's protection layer
on a schedule. Going to origin avoids that entirely.

**Status:** built and tested, not currently active. The publishing schedule
is under review pending an editorial calendar decision.

📄 `workflow.json` — importable, account identifiers and origin IP removed

---

## Result

| Metric | Before | After |
|---|---|---|
| Time per post | 15 min | 0 |
| Posts skipped when busy | frequent | none |
| Duplicate posts | possible | structurally prevented |
| Cadence | irregular | weekly |

---

## What I'd do differently

There's no human review step before publishing. For a health-related brand
that's a real risk — an LLM summary of medical content should be checked
before it goes out under the organization's name. I'd add a hold-for-approval
step: generate the draft, email it for a yes/no, publish on approval. Slower,
but appropriate for the subject matter. That's part of why it isn't active.
