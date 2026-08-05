# Cutting sponsorship contract turnaround from days to minutes

**Context:** Romanian non-profit receiving corporate sponsorships. Each sponsor
required a signed contract with a unique sequential registry number.
**Role:** Sole designer and implementer
**Stack:** n8n · Gotenberg · Google Sheets · Romanian company registry API · SMTP

---

## The problem

Every sponsorship required someone to look up the company's legal details,
copy them into a Word template, assign the next contract number from a shared
registry, export a PDF, and email it. Roughly [X] minutes of manual work per
sponsor, with three recurring failure modes: transcription errors in fiscal
codes, duplicate contract numbers when two people worked in parallel, and
delays when the person who owned the template was unavailable.

**Constraints:**
- Sponsors had to be able to self-serve from the website, without an account
- Contract numbering had to be strictly sequential and collision-free
- The company registry API is metered — every lookup costs money
- Output had to be a PDF, not HTML, for signing

---

## The approach

Split into two independent webhook entry points rather than one long flow.
The lookup and the contract generation are separate user actions, seconds or
minutes apart, and coupling them would have meant holding state between them.

| Option | Why rejected / chosen |
|---|---|
| Single webhook, everything in one pass | Rejected — user needs to review fetched company data before generating |
| Third-party e-sign platform | Rejected — per-document cost, and overkill for the volume |
| **Two webhooks: lookup, then generate** | **Chosen — matches the actual user flow, each step independently testable** |

**Cost control decision:** validation runs *before* the API call, not after.
The fiscal code is normalized (prefix stripped, non-digits removed) and
checked for plausibility locally. Invalid input never reaches the metered
endpoint. This was a deliberate choice after realizing that most bad requests
are typos, not attacks — and typos are cheap to catch locally.

**Defensive response mapping:** the registry API returns inconsistent field
names across company types. Rather than trusting one shape, the mapping step
tries several candidate keys per field and falls back cleanly. This is the
kind of thing that only shows up in production, on the tenth company.

---

## Implementation

**Flow A — company lookup**
1. Webhook receives fiscal code from the website form
2. Local validation and normalization
3. Conditional branch: invalid input returns an error without calling the API
4. Registry API lookup
5. Defensive field mapping to a stable output shape
6. Response returned synchronously to the form

**Flow B — contract generation**
1. Webhook receives the confirmed company data
2. Read current counter from the Google Sheets registry
3. Build contract HTML with the next sequential number
4. Convert HTML to PDF via a self-hosted Gotenberg container
5. Write the issued number back to the registry
6. Email the PDF to the sponsor

**Key technical decisions:**
- **Gotenberg self-hosted** instead of a PDF SaaS: no per-document cost, no
  data leaving our infrastructure, and full control over rendering
- **Google Sheets as the registry** rather than a database: the non-profit's
  staff needed to audit and correct numbering themselves, in a tool they
  already used
- **Read-then-write on the counter** with the increment inside a single
  execution to keep numbering sequential

📄 `workflow.json` — importable, credentials and document IDs removed

---

## Result

| Metric | Before | After |
|---|---|---|
| Time per contract | [X] min | under 2 min, unattended |
| People required | 1 | 0 |
| Numbering collisions | occasional | none observed |
| Transcription errors | [X] | eliminated (data comes from registry) |

---

## What I'd do differently

The sequential counter is read-then-write within one execution, which is fine
at current volume but is not atomic — two simultaneous submissions could
collide. At higher volume I'd move the counter to a store with an atomic
increment, or add a retry-on-conflict check that re-reads before writing.
I accepted the risk knowingly because contract volume is a few per month,
but I would not ship this pattern at scale.
