# LinkedIn browser playbook

Everything the skills in this repo learned the hard way about driving LinkedIn
through the Claude in Chrome extension. Every rule here cost a failed run to find.

**Read this before touching LinkedIn — for the five commands *and* for anything a
user asks for off-script.** It is not skill-specific documentation. It's the
general knowledge of how to read LinkedIn without getting silently truncated data,
mis-parsed fields, or rate-limited, and it applies to any request: "who at this
company went to my university," "which connections changed jobs," "pull these
postings into a file." The commands are five paved paths across this terrain. The
terrain is what's actually here.

If you're a human skimming this repo to see how it works: this file is the
interesting one. The skills are mostly orchestration. This is the knowledge.

---

## 1. Ground rules

**Use the user's real logged-in session.** Don't try to fetch LinkedIn with
`curl` or `fetch`. LinkedIn requires login cookies and runs aggressive bot
detection. The Chrome extension drives a real browser the user is really logged
into — that's the whole reason this approach works.

**Pace everything at human speed.** Wait ~2–3 seconds after each navigation.
Never fire more than one browser action per 1–2 seconds when paging. A fast
crawl is a banned account.

**Never solve a CAPTCHA.** If you hit a CAPTCHA, a security check, or a
"You've reached a temporary limit" interstitial: stop immediately, save whatever
you've already collected, and tell the user honestly what happened. Suggest
waiting an hour. Do not retry in-session. Do not attempt the challenge.

**Read-only, always.** These skills look at pages the user can already see with
their own eyes. Nothing here connects, messages, posts, or endorses. If a skill
ever needs to *do* something on LinkedIn, that's a human's job.

---

## 2. Getting data out of the browser

This is where the first implementation lost the most time. Two hard constraints,
both field-verified. **Do not fight them.**

### `javascript_tool` inline returns truncate at ~1 KB, and redact base64

A `javascript_tool` call that returns a big array gets silently cut off. A
250-connection extraction is ~40 KB of JSON — it will not survive the trip. Worse,
strings that *look* like base64 get redacted, which silently eats LinkedIn member
URNs (the `ACoAA…` ids) — the exact thing you need for drill-ins.

Silently is the operative word. You get a plausible-looking short result, not an
error. If a return value looks suspiciously small or a URN came back empty, this
is why.

### Chrome blocks a page's 2nd+ automatic Blob download

The "trigger a Blob download and read the file from `~/Downloads/`" trick works
exactly **once per page load**. The second automatic download from the same page
is blocked by Chrome without a prompt. Fine for a one-shot dump at the end of a
run; unusable for anything iterative.

### The channel that actually works

Accumulate into `localStorage`, then render the whole payload into a `<pre>` and
read it with `get_page_text`. That returns the full text, unredacted, at any size.

```javascript
document.body.innerHTML = '';
const p = document.createElement('pre');
p.textContent = localStorage.getItem('mcc');   // your accumulator key
document.body.appendChild(p);
'rendered ' + JSON.parse(localStorage.getItem('mcc')).length;
```

Then call `get_page_text` and transcribe the returned JSON **verbatim and
complete** to a scratch file. Never abbreviate, summarize, or reconstruct rows
from memory when transcribing — copy every record exactly, then delete the
scratch file once it's parsed.

`localStorage` also survives navigation and extension disconnects, which is what
makes multi-page crawls resumable. Use it as the accumulator, not `window`.

### Replaying a scraper across navigations

Navigation resets `window`, so a function defined on `window` is gone on the next
page. Stash the function's source in `localStorage` once, then replay it per page
with a tiny `eval` — which stays well under the 1 KB return limit because it only
returns a small progress object:

```javascript
// once, at setup:
localStorage.setItem('mccrun', '(' + window.scrapeCo.toString() + ')()');

// per page, inside a browser_batch:
eval(localStorage.getItem('mccrun'))   // → {added: 10, total: 40, hasNext: true}
```

---

## 3. Scrolling and pagination

### Infinite scroll needs real wheel events

On the connections page, **programmatic scrolling does not trigger the load.**
Setting `element.scrollTop` doesn't. `Element.scrollIntoView()` doesn't. LinkedIn
uses an intersection observer that fires on user-driven scroll events only.

Use the `computer` tool with `action: "scroll"` — it emits real wheel events that
the observer picks up. There is no "Show more" button anywhere on that page to
fall back to.

### The scroll container is `<main>`, not `<body>`

`document.body.scrollHeight` is the viewport height — useless. The connections
list lives inside `<main>`, which has its own `overflow-y: auto`. Measure and
scroll against `document.querySelector('main')`.

### Search results don't need scrolling at all

This one cuts real time off a run. The **search** pages (people search, job
search) render all their cards on navigation. Navigate, wait ~3s, read
`main.innerText`. No scroll, no screenshot. Only the infinite-scroll
*connections* page needs the wheel-event treatment.

### Batch the round-trips

Use `browser_batch` to chain `navigate` → `wait` → `javascript_tool` for several
pages in one call. The waits inside a batch are real waits, so this is still
human-paced — it just saves the round-trip overhead per step.

---

## 4. Page-specific notes

### Connections page — `/mynetwork/invite-connect/connections/`

Canonical "my connections" page; stable for years.

- **Total count**: a line reading `"<N> connections"` (lowercase c, comma-separated
  for thousands) near the top. Parse it for a progress denominator.
- **Counting cards**: use `button[aria-label^="More actions for"]` — exactly one
  per card. Do **not** count `/in/` anchors: each card renders two (photo + name).
- **Extracting URLs**: use the `/in/` anchors, dedup'd via a `Map` keyed on the
  normalized URL.
- **Card text order** is stable: name, then headline (may wrap across lines), then
  "Connected on <date>". Class names are obfuscated server-side and change — anchor
  on URL patterns and text, never on class names.
- **Declared total vs. rendered count**: off-by-one or two is normal (ghosts of
  recently removed connections, paused accounts). Don't require an exact match as
  your stop condition.
- **No bulk export via the extension.** LinkedIn's Settings → Data Privacy export
  exists but is CAPTCHA-gated and intentionally slow (up to 24 hours). The
  connections page is the only realistic source.

### People search — 2nd degree at a company

```
https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<id>%22%5D&network=%5B%22S%22%5D&page=<n>
```

Paginate with `&page=<n>`. **LinkedIn caps this at ~10 pages / ~100 results** — if
a company has 400 2nd-degrees, you get 100 of them. Say so in the output rather
than implying completeness.

- **Degree line** may be combined (`Name • 2nd`) or split across lines (`Name`,
  then `• 2nd`). Handle both.
- **Member URN**: each card holds **exactly one** `ACoAA…` id — the person's.
  Mutual-connection avatars carry none, so a single regex per card is unambiguous.

### Shared connections (the drill-in)

```
https://www.linkedin.com/search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22<member_urn>%22%5D
```

`network=["F"]` + `connectionOf=[X]` = people who are 1st-degree to **you** AND
connections of **X**. That's the mutual set — your bridges to X.

**Why this page matters is the whole point of `/map-company-connections`.** A
search result card names only 2–3 mutuals, then truncates: "…and 21 other mutual
connections." One of those hidden 21 may be the person you know best — your
actual warmest path. Reading only the visible names doesn't just undersell the
connection, it can point you at the wrong person entirely. So when a card
truncates, drill in.

Only drill cards that actually truncate (`other > 0`). Cards showing all their
mutuals need no drill — parse the card. One search per company plus one drill per
truncated card keeps the run cheap.

If a drill returns zero names, that person hides their connections. Fall back to
the card's named mutuals and note it — don't silently report a smaller number as
if it were complete.

### Mutual-line shapes

Five forms in the wild:

- `"<A> is a mutual connection"`
- `"<A> and <B> are mutual connections"`
- `"<A>, <B> and N other mutual connections"`
- `"<A>, <B>, and <C> are mutual connections"`
- `"<A>, <B>, and N other mutual connections"`

Parse these **dictionary-first**: scan your known connection names (longest-first)
as substrings and remove them, then split the leftover on `,` / ` and `. Splitting
on commas first breaks on credential-suffixed names — "Priya Natarajan, MBA"
becomes two people, one of whom is named "MBA".

### Company id

Navigate to `/company/<slug>/people/` and count `fsd_company:<digits>` occurrences
in the page HTML. The company's own id dominates by a wide margin; the runners-up
are "similar companies" in the sidebar. Take the top by count, not the first match.

### Job search — `/jobs/search/`

- `f_TPR=r604800` = posted in the past week (604800 seconds). The right window for
  a weekly scan.
- `f_WT=2` = the Remote workplace-type filter.
- `&start=<n>` paginates, where `n = 25 * (page - 1)`.
- **Page 1 under-reports.** LinkedIn fills the top slots with "Promoted" cards and
  pushes organic results to pages 2–3. Don't use "page 1 returned fewer than the
  max" as your stop condition — it will miss most of the real inventory. Instead:
  paginate if page 1's card count is below the header's stated total AND a Page 2
  link exists. Stop at page 3.
- **Card line order** is `[title, "<title> with verification", company, location, …]`.
  That accessibility alt-label line is real and it will shift company into the
  title slot and location into the company slot if you don't drop it. Filter it,
  along with badge lines (Promoted, Easy Apply, Viewed, "N applicants", …).

### Job detail — `/jobs/view/<id>`

- **Title and company come from `document.title`**, not `<h1>`. The `<h1>` on the
  detail page renders the *company* name. `document.title` reliably formats as
  `<job title> | <company> | LinkedIn`.
- **Check for "Responses managed off LinkedIn" before extracting a body.** These
  postings host their JD on the hirer's own ATS; LinkedIn renders only a meta
  header and an Apply button. Extracting anyway returns ~1.3 KB of navigation
  noise that looks superficially like a JD. Detect it, set the body empty, and say
  so in the output.
- The JD body is the block after an `About the job` line.
- Pace: 2–3 seconds between detail loads, never more than ~15/minute.

---

## 5. Failure modes

| Symptom | Cause | What to do |
|---|---|---|
| Return value truncated / URN empty | ~1 KB inline cap + base64 redaction | Use `localStorage` + `<pre>` + `get_page_text` (§2) |
| Scroll doesn't load more cards | Programmatic scroll, not a real wheel event | Use `computer` `action: "scroll"` (§3) |
| `scrollHeight` looks like the viewport | Measuring `body` instead of `main` | Measure `document.querySelector('main')` |
| Company in the title field | The `with verification` line wasn't filtered | Drop it before positional parsing (§4) |
| Job body is ~1.3 KB of nav noise | Off-LinkedIn promoted posting | Detect and flag, don't extract |
| Drill-in returns 0 names | That person hides their connections | Fall back to card mutuals, note it |
| "No results" on 2nd-degree search | Wrong company id, or genuinely zero | Re-verify the id from `/company/<slug>/people/` |
| Second Blob download silently fails | Chrome blocks 2nd+ auto-download per page load | One-shot only, or use the `<pre>` channel |
| Extension reports "not connected" | Usually a transient drop | Reconnect via `tabs_context_mcp{createIfEmpty:true}`; `localStorage` survives, resume from the next page |
| CAPTCHA / rate limit | Going too fast, or plain bad luck | **Stop.** Save partial results. Report honestly. Wait an hour. Never solve it. |

---

## 6. When LinkedIn changes

It will. Everything in §4 is a claim about a UI that ships on its own schedule,
verified as of mid-2026. When a skill starts returning nonsense, this file — not
the skill — is where the wrong assumption lives. Fix it here and the skills that
reference it all get better at once.

The load-bearing pattern that makes that survivable: **anchor on URL patterns and
visible text, never on class names.** LinkedIn's class names are obfuscated
server-side and rotate. Every selector in this repo keys off `href` shapes,
`aria-label` prefixes, and text content for exactly that reason.
