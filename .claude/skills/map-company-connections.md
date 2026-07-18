---
name: map-company-connections
description: For one target company, map the user's 2nd-degree LinkedIn connections there and the 1st-degree "bridges" who connect them to each. Runs the 2nd-degree People search, and — critically — when a result card truncates its mutuals ("…and N other mutual connection(s)") drills into that person's shared-connections page to enumerate ALL bridges (a hidden one may be the warmest). Matches every bridge to config/connections.csv ratings and writes to config/company_connections.csv, keeping only people reachable via a High- or Medium-rated bridge (High/Medium bridge names in their own columns, credential suffixes stripped). Company-first and cheap: one search per company plus one drill-in only per truncated card.
---

# /map-company-connections — who do I know at company X, and how warm is the path

You take a company, find the 2nd-degree connections who work there, and for each
one record **which of the user's 1st-degree connections bridges to them** and
**how warm that bridge is** (rated in `config/connections.csv`). The output keeps
only the people reachable through a **High- or Medium-rated** bridge — the warm
paths — with those bridge names split into `high_bridges` and `medium_bridges`
columns. People reachable only through Low or unrated bridges are left off, so the
table stays a scannable "who can actually introduce me" list.

This is the skill the repo exists for. It answers the question that actually
matters: *for a company I want to work at, who can give me a real, warm intro?*

**Read `reference/linkedin-browser-playbook.md` before running this.** Every
browser constraint below is explained there, and it's where fixes go when
LinkedIn changes.

## Why the drill-in matters (the whole point)

A 2nd-degree result card only *names* 2–3 mutual connections and then says
"…and 21 other mutual connections." One of those hidden others may be a
**High-rated** bridge — the user's actual best path. Naming only the visible 2–3
doesn't just undersell the connection, it can point the user at the wrong person
entirely. So when a card is truncated, open that person's shared-connections page
to get the complete bridge list.

(Illustrative, using the fictional persona in `config/*.example.*`: a card showing
"Felipe Marquez, Casey Lin +1" — both **Low** — hid Mira Sato, who is **High**.
Another hid Henrik Olsen and Emma Kowalski, both **Medium**. In each case the card
alone would have sent the user to their weakest path.)

---

## Parameters

- **company** — a company name (matched against `config/target_companies.csv`), OR
  a LinkedIn numeric company id directly (e.g. `18472901`), OR the company from a
  specific job listing the user is considering.
- If no argument, list High-priority rows from `config/target_companies.csv` that
  don't yet have a `company_connections.csv` block and ask via `AskUserQuestion`
  which to map.

---

## Step 0 — Preflight

1. **Files.** Confirm `config/connections.csv` exists (needed for bridge ratings)
   and `config/target_companies.csv` exists. If `connections.csv` is missing, warn
   that **every bridge will read "Unrated"** — which strips out the value entirely,
   because this skill keeps only people reachable via a High/Medium bridge. Recommend
   `/build-connections` then `/rate-connections` first. You can still proceed if the
   user insists.
2. **Rate before you scan.** Read `config/connections.csv` and count rows whose
   `rating` is blank or `Unrated`. If any exist, tell the user how many and recommend
   rating them first (via `/rate-connections`, or by editing the CSV), so the scan
   doesn't silently drop a warm path that sits behind a still-unrated bridge. Then ask
   via `AskUserQuestion`: **I've rated them — proceed** / **Proceed anyway** (people
   reachable only through unrated bridges get left off) / **Cancel**. The in-run half
   of this is Step 5 — a bridge not yet in `connections.csv` pauses the run to be rated
   on the spot.
3. **Chrome.** Call `list_connected_browsers`. If empty, bail: the Claude in Chrome
   extension isn't connected. Then `tabs_context_mcp{createIfEmpty:true}` for a
   working tab. Navigate to `https://www.linkedin.com/feed/` — if it redirects to
   a login page, bail and ask the user to log in themselves. Never log in for them.
4. **Today.** Compute `today_iso = YYYY-MM-DD` for the `scanned_date` column.

Pace all LinkedIn navigation at human speed (~2–3s between actions). If you hit a
CAPTCHA, security check, or rate-limit page, **stop**, save whatever's in
`localStorage`, and report honestly. Never solve a CAPTCHA.

---

## Step 1 — Resolve the company's LinkedIn numeric id

If given a numeric id, skip to Step 2. Otherwise:

1. If the company row in `target_companies.csv` already has a `linkedin_company_id`,
   use it.
2. Else find it: navigate to
   `https://www.linkedin.com/search/results/companies/?keywords=<company>`, pick the
   right entity by name + industry + HQ.
3. Navigate to `https://www.linkedin.com/company/<slug>/people/` and extract the
   **dominant** `fsd_company:<digits>` id (the company's own id appears far more
   often than the "similar companies" in the sidebar):

   ```javascript
   (function(){
     const html=document.documentElement.innerHTML, counts={};
     let m; const re=/(?:fsd_company|company):(\d{3,})/g;
     while((m=re.exec(html))) counts[m[1]]=(counts[m[1]]||0)+1;
     return Object.entries(counts).sort((a,b)=>b[1]-a[1]).slice(0,6);
   })()
   ```

   The top id (by a wide margin) is the company id. Write it back to
   `target_companies.csv` in Step 6.

---

## Step 2 — Scrape the 2nd-degree People search

2nd-degree search URL (paginate with `&page=<n>`; LinkedIn caps at ~10 pages /
~100 results):

```
https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<id>%22%5D&network=%5B%22S%22%5D&page=<n>
```

**Setup once** (defines the scraper, stores its source in `localStorage` so each
page can replay it with a tiny `eval`, and clears any prior run). Run this as a
standalone `javascript_tool` call:

```javascript
localStorage.removeItem('mcc');
window.scrapeCo=function(){
  const COMPANY='<Company Name>';                          // <-- set per run
  const main=document.querySelector('main');
  const lines=main.innerText.split('\n').map(s=>s.trim()).filter(Boolean);
  const noise=/mutual connection|followers|^view |^connect$|^message$|^follow$|results$/i;
  const cards=[];
  for(let i=0;i<lines.length;i++){
    const dm=lines[i].match(/•\s*(1st|2nd|3rd)\b/i); if(!dm) continue;   // handles "Name • 2nd" AND split "• 2nd"
    const before=lines[i].slice(0,dm.index).trim();
    const name=before?before:lines[i-1];
    if(!name||noise.test(name)) continue;
    const h=(lines[i+1]||'').slice(0,180);
    let loc=''; const cand=lines[i+2]||'';
    if(cand&&!noise.test(cand)&&!/•\s*(1st|2nd|3rd)/i.test(cand)) loc=cand;
    let mutual='';
    for(let j=i+1;j<Math.min(i+8,lines.length);j++){ if(/mutual connection/i.test(lines[j])){mutual=lines[j];break;} }
    let other=0; const om=mutual.match(/and\s+(\d+)\s+other\s+mutual connections?/i); if(om) other=parseInt(om[1]);
    cards.push({n:name,h:h,loc:loc,m:mutual,other:other});
  }
  const anchors=Array.from(main.querySelectorAll('a[href*="/in/"]'));
  for(const c of cards){
    let slug=null, anchorEl=null;
    for(const a of anchors){const at=(a.innerText||'').trim();
      if(at===c.n||at.startsWith(c.n+'\n')||at.startsWith(c.n+' ')){const mm=a.href.match(/\/in\/([^\/?]+)/); if(mm){slug=mm[1];anchorEl=a;break;}}}
    c.u=slug; c.urn=null;
    if(anchorEl){let card=anchorEl;for(let i=0;i<6&&card;i++){const t=(card.innerText||'').length;if(t>60&&t<600)break;card=card.parentElement;}
      if(card){const um=(card.outerHTML||'').match(/ACoAA[A-Za-z0-9_-]{10,}/); if(um)c.urn=um[0];}}  // exactly one URN per card = the person's
  }
  const store=JSON.parse(localStorage.getItem('mcc')||'[]');
  const seen=new Set(store.map(e=>e.u));
  let added=0;
  for(const c of cards){if(!c.u||seen.has(c.u))continue;seen.add(c.u);
    store.push({co:COMPANY,n:c.n,h:c.h,loc:c.loc,u:c.u,urn:c.urn,m:c.m,other:c.other,bridges:null});added++;}
  localStorage.setItem('mcc',JSON.stringify(store));
  return {added,total:store.length,hasNext:lines.indexOf('Next')>-1};
};
localStorage.setItem('mccrun','('+window.scrapeCo.toString()+')()');
'setup ok';
```

**Then crawl the pages.** Use `browser_batch` to chain, per page: `navigate` →
`computer.wait(3)` → `javascript_tool` running `eval(localStorage.getItem('mccrun'))`.
Do ~5 pages per batch. Notes:

- **No scrolling or screenshots needed.** Navigate + a ~3s wait renders all 10
  cards; `main.innerText` captures them. (Scrolling is only needed on the
  infinite-scroll *connections* page, not on search.)
- Each `eval` returns a small `{added,total,hasNext}` — no truncation.
- Stop when `hasNext:false` or a page returns `added:0`.
- Because navigation resets `window`, the scraper is replayed from
  `localStorage['mccrun']` each page.

Example single-page action inside a batch:
`{"name":"javascript_tool","input":{"action":"javascript_exec","tabId":<id>,"text":"eval(localStorage.getItem('mccrun'))"}}`

---

## Step 3 — Drill only the truncated cards

Find the people whose card hid mutuals (`other > 0`) and get their **full** bridge
list:

```javascript
(function(){
  const store=JSON.parse(localStorage.getItem('mcc')||'[]');
  return store.filter(e=>e.other>0).map(e=>({n:e.n,other:e.other,urn:e.urn}));
})()
```

Inline returns **redact** the `ACoAA…` URN (base64 filter — see the playbook §2).
To get the drill URLs in plaintext, render them into the page and read with
`get_page_text`:

```javascript
const trunc=JSON.parse(localStorage.getItem('mcc')||'[]').filter(e=>e.other>0);
document.body.innerHTML='';
const p=document.createElement('pre');
p.textContent=trunc.map(e=>e.n+' ||| https://www.linkedin.com/search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22'+e.urn+'%22%5D').join('\n');
document.body.appendChild(p); 'rendered';
```

`get_page_text` → copy each person's shared-connections URL. For each truncated
person, navigate to their URL (this is the "X mutual connections" page = people
who are connections of BOTH the user and them = all the user's 1st-degree
bridges), wait ~3s, and scrape the 1st-degree names:

```javascript
(function(){
  const main=document.querySelector('main');
  const lines=main.innerText.split('\n').map(s=>s.trim()).filter(Boolean);
  const names=[];
  for(let i=0;i<lines.length;i++){
    const bi=lines[i].indexOf('•'); if(bi<0) continue;
    const after=lines[i].slice(bi+1).trim(); if(after.indexOf('1st')!==0) continue;
    const before=lines[i].slice(0,bi).trim(); const name=before?before:lines[i-1];
    if(name && name.toLowerCase().indexOf('mutual')<0) names.push(name);
  }
  return {count:names.length, names:names};
})()
```

Write each drilled person's `bridges` array back into `localStorage['mcc']` (match
by name or slug). If a drill returns 0 names, that person hides their connections
— leave `bridges` null, fall back to the card's named mutuals, and **note it** so
the user knows that row is incomplete rather than sparse.

> Drill **every** card with `other > 0` — even one whose *visible* mutuals already
> include a High or Medium. The hidden "+N" could be a closer bridge than any of the
> named ones (the user's best friend, who happens to know this person well), and it
> costs one page load to be sure. Cards that already show all their mutuals (no "+N
> other") need no drill — their `bridges` stay null and Step 5 parses the card's
> mutual line instead.

---

## Step 4 — Get the full dataset out of the browser

Render the whole array into a `<pre>` and read it with `get_page_text` (the only
reliable channel — playbook §2 explains why the inline return and Blob-download
alternatives both fail):

```javascript
document.body.innerHTML='';
const p=document.createElement('pre');
p.textContent=localStorage.getItem('mcc');
document.body.appendChild(p); 'rendered '+JSON.parse(localStorage.getItem('mcc')).length;
```

Then call `get_page_text` and write the returned JSON **verbatim and complete** to
`jobs/.mcc-raw.json`. **Never abbreviate, summarize, or invent rows** when
transcribing — copy every record exactly, then delete the scratch file after
Step 5.

---

## Step 5 — Rate bridges, keep only warm people, write company_connections.csv

The output is a **short list of people the user can reach through a bridge they
actually value**. A person earns a row **only if at least one of their bridges is
rated High or Medium**. People whose every bridge is Low or unrated are left off —
that's the point; it keeps the table a scannable warm-intro list.

Run a Python pass (via `Bash`) that:

1. **Load ratings.** Read `config/connections.csv` into `clean_name -> rating`
   (High/Medium/Low/Unrated), where `clean_name` is the name with credential suffixes
   stripped (cleaner below).
2. **Resolve each person's bridges.** `bridges = drilled list if present, else parse
   the card's mutual line`. Parse the mutual line with a **dictionary-first** match so
   credential-suffixed names survive (e.g. "Priya Natarajan, MBA", "Greta Lindqvist,
   PHR, SHRM-CP"): scan the known connection names (longest-first) as substrings and
   remove them, then split the leftover on `,` / ` and ` for any names not in
   `connections.csv`. Splitting on commas first would turn "Priya Natarajan, MBA" into
   two people, one named "MBA".
3. **Clean every name.** Both the 2nd-degree `person_name` and every bridge name go
   through the cleaner, so the table is easy to scan — degrees and credentials are
   noise here:

   ```python
   import re
   NAME_SUFFIXES = {'jr','sr','ii','iii','iv','v'}
   def clean_name(n):
       s = (n or '').strip()
       if not s: return s
       parts = [p.strip() for p in s.split(',')]
       base = parts[0]
       if len(parts) > 1 and parts[1].lower().rstrip('.') in NAME_SUFFIXES:
           base = base + ', ' + parts[1]        # keep a genuine "Jr."/"III"
       base = re.sub(r'\s+-\s+[A-Za-z0-9./&®\-]+$', '', base).strip()   # drop " - CCP" style
       return base
   ```

   So `Greta Lindqvist, PHR, SHRM-CP` → `Greta Lindqvist`. Ratings are looked up on the
   cleaned name (which is also how `connections.csv` stores names — see
   `/build-connections`), so cleaning both sides keeps the match intact.
4. **Rate unknown bridges on the fly.** If a bridge name isn't in `connections.csv` at
   all (neither rated nor Unrated — it's *absent*), don't silently treat it as Unrated.
   Pause the run, show the user the bridge (name + the person they'd introduce) and ask
   via `AskUserQuestion` to rate them **High / Medium / Low**. Append the new bridge to
   `config/connections.csv` with that rating (via `/build-connections`'s write path —
   temp-file-then-swap, back up first), then continue. Batch these if several surface at
   once. This is the in-run half of the rate-as-you-go rule from Step 0.
5. **Classify and filter.** For each person, build:
   - `high_bridges` = semicolon list of their bridges rated **High** (cleaned names).
   - `medium_bridges` = semicolon list of their bridges rated **Medium** (cleaned names).
   **Drop the person unless `high_bridges` or `medium_bridges` is non-empty.**
6. **Emit rows** to `config/company_connections.csv` with schema:
   `company, person_name, title, high_bridges, medium_bridges, person_linkedin_url, scanned_date, notes`
   - `title` = the person's raw LinkedIn headline, verbatim.
   - `notes` = `full bridge list via drill-in (card hid +N)` when drilled; `connections
     hidden` when a drill returned 0; `ex-<former employer>` if the headline flags a
     shared background. Keep it terse.
7. **Idempotent write:** read the existing file, drop any rows where
   `company == <this company>`, add the new (kept) rows sorted by warmest bridge then
   name, write back. Re-runs are safe. Back up first
   (`cp … .bak-$(date +%Y%m%d-%H%M%S)`) — real network data, no git history.

`config/company_connections.csv` is gitignored (via `config/*`), so it's safe for
real network data.

---

## Step 6 — Write-back and report

1. If Step 1 resolved a new company id, write it into that company's
   `linkedin_company_id` in `config/target_companies.csv`, and add a short note
   (row count + warmest bridges + date).
2. Delete `jobs/.mcc-raw.json`.
3. Report to the user in plain language — the audience is a job seeker, not an
   engineer. Include:
   - How many 2nd-degrees were scanned, and **how many made the table** (had a
     High/Medium bridge). Say **whether LinkedIn's ~100 cap was hit** (if so, the scan
     is partial — don't imply completeness).
   - The list of kept people: person — title — warm bridge(s).
   - The top bridges by reach (the 1st-degrees who connect to the most kept people).
   - **Who was dropped and why:** a count of people left off because their only bridges
     were Low, and — separately — anyone dropped because their only bridge was still
     *unrated*, named individually so the user can rate that bridge and recover them
     next run. (Rare now that Step 0 and Step 5 rate bridges up front, but surface it if
     it happens.)
   - Any people whose connections were hidden, so the user knows those rows are
     incomplete.

Then point at the next step: `/rate-connections` if bridges came back unrated, or
reaching out to the top bridge themselves.

---

## Failure modes

- **"No results found" on the 2nd-degree search** → the user has 0 2nd-degrees at
  that company, or the company id is wrong. Re-verify the id from
  `/company/<slug>/people/`.
- **Drill-in returns 0** → that person hides their connections. Fall back to the
  card's named mutuals; note "connections hidden."
- **Chrome disconnects mid-crawl** → reconnect (`tabs_context_mcp{createIfEmpty:true}`);
  `localStorage` persists, so resume from the next page. Dedup by slug makes
  re-scraping harmless.
- **CAPTCHA / rate-limit** → stop, extract what's in `localStorage`, report
  honestly, recommend a cool-down. Do not retry in-session.
- **A URN won't extract from a card** (rare) → visit that person's profile and pull
  the base `ACoAA…` id from the page HTML as a fallback.

All browser interaction is confined to Steps 1–4. Nothing browser-related lives
elsewhere in this file.

---

## Design notes

**Company-first, deliberately.** The alternative — crawl each warm 1st-degree's
whole network to discover which companies they can reach — was tried and rejected:
LinkedIn caps at ~100 per person, most of the surfaced companies are irrelevant,
and many 1st-degrees hide their connections entirely. Starting from a company you
actually want and working backwards to the people is cheaper and lands better.

**This skill finds and documents. It does not contact anyone.** It writes a table.
Deciding who to ask, and how, is the user's job — that judgment is the part a
human is actually good at, and the part that makes a warm intro warm.

**What it consumes:** `config/connections.csv` (1st-degree, rated — built by
`/build-connections`, rated by `/rate-connections`). Without ratings, every path
reads "Unrated" and the table can't be sorted by warmth, which is the entire point.
