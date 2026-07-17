# LinkedIn Claude Assistant

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

You can do all of this by hand. That's the point — it's just miserable.

A search result for someone at your target company shows you a teaser:

> **Felipe Marquez, Casey Lin** and 21 other mutual connections

Click through and LinkedIn will happily show you all 23. Fine. Now do that for the
other 33 people who work there, and hold in your head which of the names on each
screen are people you'd actually feel comfortable calling. That's 34 page visits,
a scratch file, and a lot of copying and pasting — and somewhere around person 20
you've lost the thread on person 3, because the answer you want isn't on any single
screen. It's in the overlap between all of them.

This tool does that pass for you. One search per company, one drill-in for each
person whose mutuals are truncated, every bridge cross-referenced against your own
ratings of who you actually know, out the other end as a sorted table. About two
minutes instead of an evening, and you get the full picture rather than the seven
people you had patience for.

### The part you'd never assemble by hand

Once that table exists, it answers a question you probably weren't asking:
**which of your Medium contacts should you reconnect with first?**

Your Mediums are the people you genuinely know but have drifted from — the ones
where you'd want to catch up properly before asking for anything. Most people have
far more Mediums than Highs, and no obvious way to rank them. They all feel equally
worth a coffee, which means none of them get one.

The table ranks them for you. Sort by reach and you find that Henrik — who you
haven't spoken to in two years, who you'd never have thought of — is the bridge to
three people at the company you most want to work at, one of them on the team
you're targeting.

Three isn't dramatic. It's just specific, and specific is what you were missing.
It turns "I should network more" into "I should email Henrik," which is a thing a
person can actually do on a Tuesday. Catching up with him upgrades those three
paths from *cold* to *warm* — and you'd never have picked him out of 400 contacts
by memory.

A caveat worth setting expectations on: your biggest bridges are often people you
barely know. Reach and closeness aren't the same thing, and the tool will happily
show you that the person connected to 25 people at your dream company is someone
you can't call. That's still worth knowing — it's just not always the answer you
were hoping for.

---

## Install

You need [Claude Code](https://claude.com/claude-code) and the **Claude in Chrome**
extension, connected and logged into LinkedIn. That's it — no Python, no npm, no
build step. The skills are just markdown.

**[⬇ Download the ZIP](https://github.com/Jay-615/linkedin-claude-assistant/archive/refs/heads/main.zip)**

1. Download and unzip it anywhere — your Desktop is fine.
2. Open the folder in Claude Code. (In the desktop app, just point it at the
   folder. From a terminal, `cd` into it and run `claude`.)
3. Type `/setup`.

Setup asks three things — what roles you want, where, and which companies you'd
love to work at — and takes about five minutes.

Or clone it, if that's your thing:

```bash
git clone https://github.com/Jay-615/linkedin-claude-assistant.git
cd linkedin-claude-assistant
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

If you clone this rather than downloading the ZIP, and you plan to commit anything
back, run `git config core.hooksPath .githooks` once. That turns on a pre-commit
check that blocks any commit containing a name from your `connections.csv` — a
guard against accidentally publishing your own network. You don't need it if you
just downloaded the ZIP, since there's no git repo to commit to.

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

## License

MIT — see [LICENSE](LICENSE).
