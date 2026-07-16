---
name: setup
description: First-run guided configuration. Checks the Claude in Chrome extension and LinkedIn login, then asks three sets of questions (roles, locations/comp, target companies) and writes config/criteria.md and config/target_companies.csv. Takes about five minutes. Run this once after downloading the repo.
---

# /setup — first-run configuration

Get a brand-new user from "I just unzipped this folder" to "I can run a skill" in
about five minutes. Two preflight checks, three questions, two files written.

**Tone matters here.** The person running this was probably laid off recently and
is not necessarily technical. Be warm, be brief, don't lecture, and don't make
them feel like they're configuring software. No jargon in anything you print.

---

## Before you start: detect existing config

Check for `config/criteria.md` and `config/target_companies.csv`.

- **Both exist** → this is a re-run. Say so, show a one-line summary of each, and
  ask via `AskUserQuestion` whether to **Reconfigure from scratch** / **Edit one
  section** / **Cancel**. Never silently overwrite work.
- **Neither exists** → fresh setup. Continue.
- **One exists** → note which, and only run the missing stage.

---

## Stage 0 — The two things that must work

Do these as checks, not questions. The user shouldn't have to know what to answer.

### Chrome extension

Call `list_connected_browsers`.

- **Empty** → stop and say:

  > I can't see Chrome yet. This tool reads LinkedIn through the **Claude in
  > Chrome** extension, so that's the one thing it can't work without.
  >
  > Install it, make sure it's connected, then run `/setup` again. Everything else
  > takes about five minutes.

  Bail. Don't try to work around it — every skill in this repo needs it.

- **Connected** → say so in one line and move on.

### LinkedIn login

Navigate to `https://www.linkedin.com/feed/`.

- **Redirects to login** → stop and say:

  > Chrome's connected, but you're not logged into LinkedIn. Open
  > https://www.linkedin.com in Chrome, sign in normally, then run `/setup` again.

  Bail. **Never log in for them and never handle their password.**

- **Loads the feed** → say so and continue.

---

## Stage 1 — Roles

Ask what job titles they're looking for. Explain the cost in plain terms:

> What job titles should I search for? Two or three is the sweet spot — I run one
> LinkedIn search per title, so ten titles means a slow scan and a higher chance
> LinkedIn rate-limits me.

Collect 2–4 role strings. If they give more than 5, gently suggest trimming and
name the tradeoff, but write whatever they insist on — it's their search.

---

## Stage 2 — Locations, seniority, comp, job types

Ask these together — they're one mental context, and splitting them makes setup
feel longer than it is.

1. **Locations.** "Where do you want to work? A metro area, US-remote, or both."
   Capture a metro string and/or remote.
2. **Comp floor.** "What's the lowest salary you'd seriously consider? I'll drop
   postings that openly list less. Postings that don't say — which is most of them
   — I'll keep."

   That second sentence matters. Without it people set a high floor and wonder why
   the inbox is empty.
3. **Seniority.** Infer sensible keep/drop lists from their roles rather than
   asking. If their roles are Senior/Lead/Principal, drop VP+/Head of/Chief and
   Associate/Junior/Intern, and keep-but-flag Director. **Show them what you
   inferred** and let them correct it — don't make them build the list.
4. **Job types.** Ask only: "Open to contract work, or full-time only?"

Write `config/criteria.md` in the shape of `config/criteria.example.md`. Follow
that file's section headings exactly — `/find-jobs` parses them.

---

## Stage 3 — Target companies

> Which companies would you most like to work at? Even five is enough to start.
> These are what `/map-company-connections` maps your network against — it finds
> who you know who can introduce you to someone inside.

Collect company names and a priority (High/Medium/Low) for each. Don't ask for
LinkedIn company ids — `/map-company-connections` resolves and caches those itself
on first run.

Write `config/target_companies.csv` with the header from
`config/target_companies.example.csv`, leaving `linkedin_company_id` blank:

```
id,company,linkedin_company_id,location,city,contacts,priority,status,roles_of_interest,notes
```

Set `status` to `Researching` and leave `contacts` and `notes` empty.

---

## On completion

Confirm what you wrote, then give them **one** next step — not a menu:

> You're set up. Two files written:
>
> - `config/criteria.md` — 3 roles, Seattle + US-remote, $150K floor
> - `config/target_companies.csv` — 8 companies
>
> **Start here: `/build-connections`.** It pulls your LinkedIn connections into a
> local file (~3–5 minutes for a typical network). Everything interesting depends
> on that list existing.
>
> After that: `/rate-connections` to mark who you actually know well, then
> `/map-company-connections <company>` to find who can introduce you.
>
> Your answers live in `config/` and never leave your machine — that folder is
> gitignored.

Resist listing all five skills here. A new user needs one door, not five.

---

## If the user aborts mid-setup

Write whatever's complete, tell them plainly what's missing, and note that re-running
`/setup` picks up the missing pieces. Never leave a half-written file that
`/find-jobs` will choke on — either a file is complete and valid, or it isn't there.
