---
name: find-jobs
description: Find new LinkedIn job postings matching config/criteria.md, apply hard filters, dedupe against jobs/seen.json, and write each survivor's full job description to jobs/inbox/ as markdown. Uses the Claude in Chrome extension against the user's logged-in session. Read-only — never applies to anything.
---

# /find-jobs — pull matching LinkedIn postings into a local inbox

You search LinkedIn for the roles in `config/criteria.md`, drop the obvious
misfits, skip anything already seen on a previous run, and write each survivor's
full job description to `jobs/inbox/<id>.md`.

**This is not an auto-applier.** It stops at "the JD is on your disk." Reading,
deciding, and applying are the user's job. Don't offer to apply, and don't click
Apply buttons.

**Read `reference/linkedin-browser-playbook.md` before running this.** §4 covers
the job-search and job-detail page quirks, several of which will silently corrupt
your output if you skip them.

---

## Step 0 — Preflight

1. Read `config/criteria.md`. If missing:

   > Missing `config/criteria.md` — run `/setup` first.

   Bail.

2. Parse from it: the **Roles** list, the **Locations** list, the **Seniority**
   rules, the **Compensation floor**, the **Job types**.

3. Load (or create empty `{}`) `jobs/seen.json`. Create `jobs/inbox/` if needed.

4. Browser preflight:
   - `list_connected_browsers` — if empty, bail (extension not connected).
   - `tabs_context_mcp{createIfEmpty: true}` for a working tab. Reuse it all run.
   - Navigate to `https://www.linkedin.com/feed/`. If it redirects to login, bail:

     > Chrome is connected but you're not logged into LinkedIn. Log in in Chrome
     > and re-run.

     Never log in for the user.

5. Set `today_iso = YYYY-MM-DD`. Use it for `fetched_date` and the id prefix.

Pace: wait 2–3 seconds after each navigation. Never more than one browser action
per 1–2 seconds. If you hit a CAPTCHA or rate-limit interstitial, **stop**, save
what you have, report honestly, and suggest waiting an hour. Never solve it.

---

## Step 1 — Build the search queries

For each role in `criteria.md`, build one URL per location mode:

- **Geographic**: `https://www.linkedin.com/jobs/search/?keywords=<URL-encoded role>&location=<URL-encoded location>&distance=25&f_TPR=r604800`
- **US Remote**: `https://www.linkedin.com/jobs/search/?keywords=<URL-encoded role>&location=United%20States&f_WT=2&f_TPR=r604800`

`f_TPR=r604800` is "posted in the past week" — the right window for a weekly scan.
`f_WT=2` is the Remote workplace-type filter. Hybrid postings fall out of the
geographic search naturally.

That's `len(roles) × 2` searches. Three roles = 6 loads. If the user has ten roles
configured, say so and suggest trimming — twenty searches is a slow run and a
rate-limit risk.

---

## Step 2 — Extract the result cards

Navigate to each URL, wait ~2–3s for React, then extract:

```javascript
(function() {
  window.scrollTo(0, document.body.scrollHeight);
  const out = [];
  // Lines that appear inside cards but are chrome/badges, not title/company/location.
  const noise = /^(promoted|easy apply|viewed|be an early applicant|actively reviewing applicants|medical|401\(k\)|· \d+|\d+ applicants?|new|company logo)$/i;
  // LinkedIn inserts an accessibility alt-label "<title> with verification" between
  // the visible title and the company line. Leave it in and company shifts into the
  // title slot, location into the company slot.
  const isVerifLine = (l) => /\bwith verification$/i.test(l);
  const links = Array.from(document.querySelectorAll('a[href*="/jobs/view/"]'));
  const seen = new Set();
  for (const a of links) {
    const m = a.href.match(/\/jobs\/view\/(\d+)/);
    if (!m) continue;
    const id = m[1];
    if (seen.has(id)) continue;
    seen.add(id);
    const card = a.closest('li') || a.closest('div');
    let lines = (card ? card.innerText : '').split('\n').map(s => s.trim()).filter(Boolean);
    lines = lines.filter(l => !noise.test(l) && !isVerifLine(l));
    out.push({
      id, url: `https://www.linkedin.com/jobs/view/${id}`,
      title: lines[0] || '', company: lines[1] || '', location: lines[2] || '',
      text_excerpt: lines.slice(0, 8).join(' | ')
    });
  }
  return out;
})()
```

The card excerpt holds enough to apply the title, seniority, and location filters.
**Don't open every job detail page** — that's the expensive trap. Open a JD only
for cards that pass the cheap filters AND aren't already in `seen.json`.

### When to paginate

Page 1 commonly returns only a handful of organic cards because LinkedIn fills the
top slots with Promoted listings and pushes the real inventory to pages 2–3.

Rule: **if page 1 returns fewer cards than the header's stated total AND a Page 2
link exists, paginate.** Stop at page 3 or when no `Next` is visible.

```javascript
(function() {
  const text = (document.querySelector('main') || document.body).innerText || '';
  const lines = text.split('\n').map(l => l.trim()).filter(Boolean);
  const headerLine = lines.find(l => /^(?:about\s+)?[\d,]+\s+results?$/i.test(l));
  const header_total = headerLine ? parseInt(headerLine.replace(/[^\d]/g, ''), 10) : null;
  const has_page_2 = lines.some(l => /^2$/.test(l)) || /\bPage\s+2\b/.test(text);
  const has_next = /\bNext\b/.test(text);
  return { header_total, has_page_2, has_next };
})()
```

Paginate by appending `&start=<n>` where `n = 25 * (page - 1)`. Accumulate deduped
cards across pages in an outer set keyed on the `/jobs/view/<id>` URL.

---

## Step 3 — Apply the hard filters

Applied to every card, using the rules the user wrote in `criteria.md`. A posting
that fails any of these is **dropped** — it never reaches the inbox. Log every drop
with a one-line reason.

1. **Title category.** The title must plausibly be one of the user's configured
   roles or a close variant. Use judgment, not regex — a "Senior PM, Growth" is a
   Senior Product Manager; a "Product Marketing Manager" is not, unless the JD body
   clearly establishes the right scope.

2. **Seniority.** Apply the keep/drop lists from `criteria.md`. Titles the user
   flagged as "keep but flag" stay in and get a note.

3. **Location.** Keep only postings matching a configured location. Reject
   geo-restricted remote roles that don't match ("Remote — Canada only", "Remote —
   EU"). If the location field is genuinely unclear, read the JD body for a clue
   before deciding.

4. **Compensation floor.** If comp **is disclosed** and is below the floor → drop.
   If comp is **not disclosed** → **keep**, with `comp_floor_met: null`. Most
   postings don't disclose; dropping them would gut the inbox.

5. **Job type.** Apply the keep/drop lists from `criteria.md`. If the type is
   genuinely unknown, keep it.

Track each drop as `{url, reason, title, company}` for the summary.

---

## Step 4 — Dedupe against `jobs/seen.json`

For every survivor:

1. Look up `url` in `jobs/seen.json`. If present → **skip**, count it, move on.
2. Otherwise it's new. Add it:
   ```json
   "<url>": { "first_seen": "<today_iso>", "status": "inbox" }
   ```

Write `jobs/seen.json` back to disk once the run completes, so a mid-run crash
doesn't lose dedup progress. This file is what keeps the same postings from
resurfacing every week.

---

## Step 5 — Fetch the JD for each new survivor

For each new survivor, navigate to its `/jobs/view/<id>` URL, wait ~2s, and extract:

```javascript
(function() {
  const main = document.querySelector('main') || document.body;
  const fullText = main.innerText || '';
  const lines = fullText.split('\n').map(s => s.trim()).filter(Boolean);

  // Title + company from document.title, NOT <h1>. The <h1> on the detail page
  // renders the COMPANY name, not the job title.
  const dtMatch = document.title.match(/^(.+?)\s\|\s(.+?)\s\|\sLinkedIn$/);
  const title = dtMatch ? dtMatch[1] : (lines[0] || '');
  const company = dtMatch ? dtMatch[2] : '';

  const postedLine = lines.find(l => /posted|reposted/i.test(l)) || '';
  const compLine = lines.find(l => /\$[\d,]+(\.\d+)?\s*(\/\s*(hr|hour|yr|year))?(\s*-\s*\$[\d,]+)?/i.test(l)) || '';

  // Promoted postings that route responses off-site host their JD on the hirer's
  // own ATS. LinkedIn renders only a meta header — extracting anyway returns ~1.3KB
  // of navigation noise that looks superficially like a JD.
  const off_linkedin = /responses managed off linkedin/i.test(fullText);

  let body = '';
  if (!off_linkedin) {
    const aboutIdx = lines.findIndex(l => /^about the job$/i.test(l));
    body = aboutIdx >= 0 ? lines.slice(aboutIdx + 1).join('\n\n') : lines.slice(0, 200).join('\n\n');
  }
  return { title, company, postedLine, compLine, body, off_linkedin, all_chars: fullText.length };
})()
```

**If `off_linkedin === true`:** skip body extraction, leave the body empty, and add
this to the file's Notes:

> JD body isn't on the LinkedIn page (Promoted posting, responses managed
> off-LinkedIn). Follow the URL above and click Apply to read the full JD.

Flagging it honestly is better than writing 1.3 KB of navigation chrome into a file
labelled "Job Description."

**Pace:** 2–3 seconds between detail loads. Never more than ~15 per minute. If a
CAPTCHA appears, stop the run and note it in the summary.

---

## Step 6 — Write each survivor to `jobs/inbox/`

Build a stable id:

```
linkedin-<YYYY-MM-DD>-<company-slug>-<role-slug>
```

Slugify: lowercase, ASCII only, hyphens for spaces, drop punctuation. Truncate the
role slug to ~60 chars. On a collision, append `-2`, `-3`.

Write `jobs/inbox/<id>.md`:

```markdown
---
id: linkedin-2026-07-16-lumora-commerce-senior-product-manager
source: linkedin
url: https://www.linkedin.com/jobs/view/4123456789
company: Lumora Commerce
title: Senior Product Manager
location: Seattle, WA
mode: Hybrid
type: full-time
posted_date: 2026-07-12
fetched_date: 2026-07-16
comp_text: "$165,000 - $195,000/yr"
comp_disclosed: true
comp_floor_met: true
---

# Senior Product Manager — Lumora Commerce

## Job Description

<full JD body, paragraph breaks preserved>

## Key Requirements (extracted)

- 3–6 bullets from the JD's must-have list

## Nice-to-haves (extracted)

- from the stated bonus / preferred qualifications (skip the section if none)

## Notes

- free-text observations: hiring manager named in the posting, "posted 14 days ago
  — may be stale", off-LinkedIn flag, etc. (optional)
```

Field notes:

- `mode` — read from the JD's location/workplace line: Remote / Hybrid / Onsite /
  unknown.
- `type` — infer from title and JD. Most cards expose a Job Type chip near the top.
- `comp_*` — parse the comp line. "$185,000 - $235,000" → floor $185,000 annual.
  "$90/hr" → floor $90/hr. No comp line → `comp_disclosed: false`,
  `comp_floor_met: null`.
- Strip boilerplate footers (EEO statements, benefits legalese) only if they crowd
  the file past ~6K words.
- **Extracted requirements are a summary, not a replacement.** Keep the full body
  above them — the user should be able to read the real JD without going back to
  LinkedIn.

---

## Step 7 — Run summary

```
LinkedIn scan — done. 6 searches (3 roles × 2 location modes).

  31 cards found → 18 passed filters
  12 were dedupes from prior runs — skipped
  6 new postings written to jobs/inbox/

  Filter drops: 8 too senior (VP+), 4 wrong location, 1 comp below floor.
  1 posting's JD is hosted off-LinkedIn — flagged in the file.
```

If anything was skipped or blocked, name it and why:

```
  Search 4/6 (Principal PM, Remote): stopped — hit a rate-limit page.
    Wrote what we had. Try again in an hour.
```

Then point at the next step:

> Read what looks good in `jobs/inbox/`. For anything at a company you care about,
> run `/map-company-connections <company>` to see who can introduce you.

---

## Failure modes

- **A search returns 0 cards** → either genuinely nothing new this week, or the
  card selector broke. Check whether *every* search returns 0; if so, the UI shifted
  — see playbook §4.
- **Company name lands in the title field** → the `with verification` line wasn't
  filtered. Playbook §4.
- **JD body is ~1.3 KB of nav noise** → off-LinkedIn posting that wasn't detected.
- **CAPTCHA / rate limit** → stop, save, report, wait an hour. Never solve it.
- **Chrome disconnects mid-run** → reconnect with `tabs_context_mcp{createIfEmpty:true}`
  and resume; `seen.json` makes re-running harmless.

All browser interaction is confined to Steps 0–5.
