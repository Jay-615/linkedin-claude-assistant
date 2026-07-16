# LinkedIn Job Search and Networking Finder

Find the jobs, then find the person who can introduce you to them.

This is a small job-search workspace for [Claude Code](https://claude.com/claude-code).
It drives your own logged-in LinkedIn session through the Claude in Chrome
extension to do two tedious things well:

- **Pull job descriptions.** Search LinkedIn for the roles you want, drop the
  obvious misfits, and save the full descriptions as markdown on your machine — so
  they're readable offline and don't vanish when the posting comes down.
- **Find warm intros.** Pick a company you want to work at. Get back a ranked list
  of everyone you're connected to there, and — the useful part — *which of your
  friends can actually introduce you*, sorted by how well you know them.

Everything stays on your machine. It never applies to a job, never messages anyone,
and never runs on a schedule. You stay in the loop for every decision that matters.

**[→ See what the output actually looks like](docs/sample-output.md)**

---

## The interesting part

LinkedIn shows you a teaser and hides the good stuff. A search result for someone
at your target company says:

> **Felipe Marquez, Casey Lin** and 21 other mutual connections

Take that at face value and your best path to a Director of Product is two people
you barely know. Most people read that card and move on.

But one of those 21 hidden names might be your closest former colleague — the
person who'd take your call today. The card doesn't just undersell the connection.
**It points you at the wrong person.**

So when a card truncates, this tool opens that person's shared-connections page and
enumerates every bridge, then cross-references each one against your own ratings of
how well you know them. It only drills the cards that actually hide something, which
keeps a full company map cheap — one search, plus one drill per truncated card.

That's the whole idea. The rest is plumbing.

---

## Install

You need [Claude Code](https://claude.com/claude-code) and the **Claude in Chrome**
extension, connected and logged into LinkedIn. That's it — no Python, no npm, no
build step. The skills are just markdown.

**[⬇ Download the ZIP](https://github.com/Jay-615/linkedin-job-search-and-networking-finder/archive/refs/heads/main.zip)**

1. Download and unzip it anywhere — your Desktop is fine.
2. Open the folder in Claude Code. (In the desktop app, just point it at the
   folder. From a terminal, `cd` into it and run `claude`.)
3. Type `/setup`.

Setup asks three things — what roles you want, where, and which companies you'd
love to work at — and takes about five minutes.

Or clone it, if that's your thing:

```bash
git clone https://github.com/Jay-615/linkedin-job-search-and-networking-finder.git
cd linkedin-job-search-and-networking-finder
claude
```

---

## Using it

Run these in order the first time. The order matters — see below.

| Command | What it does | Time |
|---|---|---|
| `/setup` | Three questions. Writes your config. | ~5 min |
| `/build-connections` | Pulls your LinkedIn connections into a local CSV. | ~3–5 min |
| `/rate-connections` | You mark who you actually know well. **Don't skip this.** | as long as you like |
| `/map-company-connections <company>` | The main event: who can introduce you, ranked. | ~2 min per company |
| `/find-jobs` | Searches LinkedIn, saves matching job descriptions. | ~3 min |

After the first run, it's just `/find-jobs` and `/map-company-connections` whenever
you feel like it.

### Why the rating step is not optional

`/map-company-connections` ranks your intro paths by how well you know the person
making the intro. That rating can only come from you — LinkedIn doesn't know which
of your 400 connections would actually take your call, and neither does Claude.

Skip the rating step and every path comes back "Unrated." The tool still finds the
people; it just can't tell you which door to knock on, which is the entire point.

**You don't have to rate everyone.** Fifty is plenty to start. Rate your top fifty,
map one company, and see if the output is worth more of your time.

---

## What it won't do

This tool stops at "here's the job description, and here's who can introduce you."
That's deliberate.

- **It doesn't apply to jobs.** Mass-applying is how you get ignored, and a bot
  can't tell you whether a role is right for you.
- **It doesn't message anyone.** It hands you a name and how you know them. What
  you say to an old colleague is not something to automate — the fact that a human
  wrote it is most of why a warm intro works.
- **It doesn't rate your relationships for you.** Only you know who you can call.
- **It doesn't solve CAPTCHAs or log in for you.** If LinkedIn pushes back, it
  stops, saves what it has, and tells you honestly.
- **It doesn't run on a schedule.** Every run is you deciding to run it.

It reads pages you can already see, at roughly the pace you'd read them.

---

## Your data

Nothing leaves your machine. `config/` and `jobs/` are gitignored in full — your
criteria, target companies, connections, and every job description stay local.

`config/connections.csv` deserves special mention. It holds your private read on
your own relationships: who you know well, who you've drifted from, who you'd feel
awkward asking for a favor. That file is the reason the tool works, and it is the
last thing you'd want to leak. It never gets committed.

The only committed examples are `config/*.example.*`, which use a fictional persona
(Sam Chen, a Senior PM in e-commerce). No real person appears anywhere in this repo.

---

## How it's built

Five skills in `.claude/skills/`, each a markdown file describing a procedure.
There's no application code — Claude reads the skill and executes it, using the
Chrome extension for LinkedIn and Python for the CSV work.

The one file worth reading is
**[`reference/linkedin-browser-playbook.md`](reference/linkedin-browser-playbook.md)**.
Everything hard-won about driving LinkedIn lives there, and every rule in it cost a
failed run to find:

- Inline JavaScript returns truncate at ~1 KB and silently redact anything that
  looks like base64 — which eats the member IDs you need. Accumulate in
  `localStorage`, render to a `<pre>`, read it back with `get_page_text`.
- LinkedIn's infinite scroll only fires on real wheel events. Setting `scrollTop`
  does nothing. Search pages, on the other hand, need no scrolling at all.
- The scroll container is `<main>`, not `<body>`.
- Job detail pages put the *company* in the `<h1>`. The job title is in
  `document.title`.
- An accessibility label reading `"<title> with verification"` sits between the
  title and company lines, and will shift every field by one if you don't drop it.
- Chrome silently blocks a page's second automatic Blob download.

Class names are obfuscated server-side and rotate, so every selector keys off URL
patterns, `aria-label` prefixes, and visible text instead. The playbook is separate
from the skills for exactly one reason: when LinkedIn changes — and it will — there's
one file to fix.

---

## Related

The bigger, messier project this was carved out of:
[Job-Seeker](https://github.com/Jay-615/Job-Seeker) — adds multi-source scanning
(Indeed, Dice, ATS APIs), weighted scoring, and tailored resume and cover-letter
drafting.

## License

MIT — see [LICENSE](LICENSE).
