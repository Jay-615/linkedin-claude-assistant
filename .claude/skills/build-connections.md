---
name: build-connections
description: Populate config/connections.csv from the user's 1st-degree LinkedIn connections. Uses the Claude in Chrome extension to paginate through linkedin.com/mynetwork/invite-connect/connections/ at human pace, writes each connection to the CSV with rating=Unrated. Re-runnable — preserves existing ratings, only adds new connections.
---

# /build-connections — populate connections.csv from LinkedIn

You are doing a bulk pull of the user's 1st-degree LinkedIn connections into
`config/connections.csv`. This is expected to be expensive in tokens (a network of
500 connections may cost 50–100K). The user runs it once after setup, then again
periodically (monthly is plenty) to pick up new connections. **Existing ratings
must be preserved across re-runs** — they represent real human judgment the user
spent time on, and losing them is the worst thing this skill could do.

The CSV schema is fixed:

```
linkedin_url,name,headline,rating,last_interaction,notes
```

`linkedin_url` is the unique key — that's how an existing row matches a freshly
scraped one.

**Read `reference/linkedin-browser-playbook.md` before running this.** The
scrolling and extraction rules below are explained there.

---

## Opening message to the user

Before doing anything, set expectations. Print:

> I'm about to pull all of your 1st-degree LinkedIn connections into
> `config/connections.csv`. This uses the Claude in Chrome extension to scroll
> through your connections list at human pace — for a network of ~500, expect 3–5
> minutes. Existing ratings in the CSV (if any) will be preserved.
>
> This only reads the connections page you can already see yourself. It doesn't
> connect, message, or follow anyone.

Then ask via `AskUserQuestion`: **Proceed** / **Cancel**. If Cancel, stop cleanly.

---

## Step 1 — Browser preflight

1. Call `list_connected_browsers`. If empty:

   > I can't reach Chrome. Make sure the Claude in Chrome extension is installed
   > and connected, then re-run. If you've never set this up, run `/setup` first.

   Bail.

2. Call `tabs_context_mcp` with `createIfEmpty: true` to get a working tab.
3. Navigate to `https://www.linkedin.com/feed/`. If the page redirects to a login
   screen, bail with:

   > Chrome is connected but you don't appear to be logged into LinkedIn. Log in
   > in Chrome (open https://www.linkedin.com and sign in), then re-run.

   **Don't try to log in for the user.** Never handle their password.

---

## Step 2 — Load the existing CSV

Read `config/connections.csv` if it exists. Parse it into a dict keyed by
`linkedin_url`. If the file doesn't exist yet, start with an empty dict — you'll
create the file from scratch in Step 6.

---

## Step 3 — Navigate to the connections page and discover the total

1. Navigate to `https://www.linkedin.com/mynetwork/invite-connect/connections/`.
   Wait ~3 seconds for the React render.

2. Read the total connection count from the page header. The line reads
   `"<N> connections"` (lowercase 'c', often comma-separated for thousands):

   ```javascript
   (function() {
     const text = (document.body.innerText || '').split('\n').map(l => l.trim());
     for (const line of text) {
       const m = line.match(/^([\d,]+)\s+[Cc]onnections?$/);
       if (m) return { total: parseInt(m[1].replace(/,/g, ''), 10) };
     }
     return { total: null };
   })()
   ```

3. Report the total to the user. If it's large (≥1000), ask whether to proceed in
   full or cap at the first 500. Default suggestion: proceed in full, confirming
   every 250 (Step 4).

   If the total can't be parsed, that's fine — note it and paginate anyway. You
   just won't have a progress denominator.

---

## Step 4 — Paginate through all connections

The connections page uses **infinite scroll triggered by real wheel events**, with
no "Show more" button. Two facts that matter (playbook §3):

1. **The scroll container is `<main>`, not `<body>`.** `document.body.scrollHeight`
   is the viewport height; `document.querySelector('main').scrollHeight` is the
   connections-list height.
2. **Programmatic `scrollTop` assignment does not trigger the load.** Neither does
   `Element.scrollIntoView()`. You must emit a real wheel event. The `computer`
   tool with `action: "scroll"` does this; `javascript_tool` setting `scrollTop`
   does not.

Loop:

1. Use `computer` with `action: "scroll"`, `coordinate: [500, 400]` (anywhere over
   the main column), `scroll_direction: "down"`, `scroll_amount: 10`.

2. Wait ~2 seconds for the next chunk to render. Each batch typically reveals 8–10
   new cards.

3. Check the card count using the `aria-label^="More actions for"` selector — more
   reliable than counting `/in/` anchors, since each card has two anchors but
   exactly one "More actions for <Name>" button:

   ```javascript
   document.querySelectorAll('button[aria-label^="More actions for"]').length
   ```

4. Stop the loop when:
   - The count reaches (or nearly reaches) the total from Step 3 — off-by-one or
     two is normal for ghosts and recently removed connections, so don't require
     an exact match, **or**
   - The count stops increasing across two consecutive scroll batches, **or**
   - You hit a user-defined cap.

5. **Throughput.** Batches of 5 scrolls + waits load ~40–50 cards. For a
   270-connection network: ~5–6 batches, ~3 minutes. Use `browser_batch` to chain
   scroll/wait/scroll/wait into one round-trip — faster, and still human-paced
   (the waits inside a batch are real waits).

6. **Confirmation every 250 cards** for large networks: after 250, 500, 750, …
   pause and ask via `AskUserQuestion`: **Continue** / **Stop here and save what
   I've got**. This lets the user bail without losing progress.

7. **If you see a CAPTCHA, security check, or "You've reached a temporary limit"**
   → stop immediately, save what you've collected, and tell the user honestly what
   happened. Suggest waiting an hour. Never attempt the challenge.

   If the page text shape changes (no card markers found at all) → stop, save
   partial results, and tell the user LinkedIn's UI may have shifted. Point them
   at `reference/linkedin-browser-playbook.md` §4.

---

## Step 5 — Extract per-connection fields

Once the loop finishes (or you bail), extract every connection currently rendered:

```javascript
(function() {
  // Each card contains an <a href="/in/<slug>/"> link to the profile. Class names
  // are obfuscated server-side, so we anchor on the URL pattern.
  // The card root is the first ancestor whose innerText length looks like a single
  // card (50–500 chars) — bigger and we've walked up into the grid container.
  const anchors = Array.from(document.querySelectorAll('a[href*="/in/"]'));
  const seen = new Map();

  for (const a of anchors) {
    let href = a.href.split('?')[0].split('#')[0];
    if (!/\/in\/[^/]+\/?$/.test(href)) continue;
    if (!href.endsWith('/')) href += '/';
    if (seen.has(href)) continue;

    let card = a.parentElement;
    for (let i = 0; i < 6 && card; i++) {
      const tlen = (card.innerText || '').length;
      if (tlen > 50 && tlen < 500) break;
      card = card.parentElement;
    }
    if (!card) continue;

    const lines = (card.innerText || '').split('\n').map(l => l.trim()).filter(Boolean);
    const filtered = lines.filter(l =>
      !/^(message|connected on|view profile|remove connection|more options)/i.test(l)
    );
    const name = filtered[0] || null;
    // Headlines can wrap across multiple lines — join everything after the name and
    // strip a trailing "Message" if it leaked in.
    let headline = filtered.slice(1).join(' ').trim().replace(/\s*Message\s*$/, '').trim();
    if (headline.length > 400) headline = headline.slice(0, 400);

    if (name) seen.set(href, { linkedin_url: href, name, headline });
  }
  return Array.from(seen.values());
})()
```

**Getting the result out.** A 250+ connection extraction is ~40 KB of JSON, and
inline `javascript_tool` returns truncate at ~1 KB (playbook §2). Don't return it
directly. Store it on `window.__connections`, then render it into a `<pre>` and
read it with `get_page_text`:

```javascript
document.body.innerHTML = '';
const p = document.createElement('pre');
p.textContent = JSON.stringify(window.__connections);
document.body.appendChild(p);
'rendered ' + window.__connections.length;
```

Transcribe the returned JSON **verbatim and complete** — never abbreviate or
reconstruct rows from memory.

(A Blob download to `~/Downloads/` also works, but only once per page load —
Chrome blocks the second automatic download. The `<pre>` channel has no such limit
and is the safer default.)

---

## Step 6 — Merge with existing data and write the CSV

For each scraped `{linkedin_url, name, headline}`:

1. **If `linkedin_url` already exists in the loaded dict:** preserve the existing
   `rating`, `last_interaction`, and `notes`. Update `name` and `headline` to the
   scraped values — those legitimately change (name change, new job). This is how
   re-running preserves the user's ratings.

2. **If it's a new row:** add it with `rating=Unrated`, `last_interaction=""`,
   `notes=""`.

3. Write the merged result to `config/connections.csv` using Python's `csv` module
   via `Bash` — safer than hand-building the file, since names contain commas and
   headlines contain quotes. Use this exact column order:
   `linkedin_url,name,headline,rating,last_interaction,notes`. Quote with
   `csv.QUOTE_MINIMAL`.

   ```bash
   python3 - <<'PY'
   import csv, json, sys
   rows = json.loads(sys.stdin.read())  # list of dicts
   with open('config/connections.csv', 'w', newline='') as f:
       w = csv.DictWriter(f, fieldnames=['linkedin_url','name','headline','rating','last_interaction','notes'])
       w.writeheader()
       for r in rows:
           w.writerow(r)
   PY
   ```

   Pipe the merged JSON into stdin. Sort rows by `name` ascending before writing —
   it makes the CSV easier to scan by hand in a spreadsheet.

---

## Step 7 — Report to the user

```
LinkedIn connections sync — done.
  Total scraped: 487
  Already in CSV: 412 (ratings preserved)
  Newly added: 75
  Now in config/connections.csv: 487 rows
    High: 31 / Medium: 22 / Low: 14 / Unrated: 420
```

Then point at next steps, and be honest about why the rating step matters:

> Next: rate these. `/map-company-connections` sorts your intro paths by how well
> you know the bridge, so until connections are rated, every path it finds comes
> back "Unrated" and can't be ranked — which is most of the value.
>
> Two ways to do it, both fine:
>
> 1. `/rate-connections` — interactive, one at a time.
> 2. Open `config/connections.csv` in Excel, Numbers, or any spreadsheet, sort by
>    headline (groups people by company), and bulk-edit the `rating` column.
>
> You don't need to rate all 420. Rating even your top 50 makes the tool useful.

That last line matters. A user staring at 420 Unrated rows will bounce; a user who
knows 50 is enough will start.

---

## Re-running this skill

Re-running weeks or months later is supported and expected. The merge in Step 6
preserves all existing ratings, `last_interaction` values, and notes. On a re-run:

- New connections get appended with `rating=Unrated`.
- Existing connections' `name` and `headline` get refreshed.

Nothing destructive happens. Note that `config/connections.csv` is gitignored, so
there's no git history to recover from — if the user wants a backup, that's on
them. The skill does not back up the CSV before overwriting.

---

## LinkedIn UI dependencies

The specifics — the connections URL, the count header shape, the `<main>` scroll
container, the wheel-event requirement, the per-card structure — all live in
`reference/linkedin-browser-playbook.md` §3–4. When LinkedIn changes, fix it there,
not here.

All browser interaction is confined to Steps 1, 3, 4, and 5. Nothing
browser-related lives elsewhere in this file.
