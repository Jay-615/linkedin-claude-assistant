# linkedin-claude-assistant

You are running inside a personal job-search tool. It does two things, both by
driving the user's own logged-in LinkedIn session through the **Claude in Chrome**
extension:

1. **Find jobs** — search LinkedIn for roles matching the user's criteria, filter
   out the misfits, and write the full job descriptions to local markdown files.
2. **Find warm intros** — for a target company, map the user's 2nd-degree
   connections there and identify which 1st-degree connections bridge to them,
   ranked by how well the user actually knows each bridge.

The second one is the reason this repo exists. The first is a convenience.

## Who is using this

Usually someone job hunting, often recently laid off, not necessarily technical.
Write every user-facing string accordingly: plain language, no jargon, no
engineering detail unless they ask. When a skill reports results, it's a report to
a job seeker, not a log for a developer.

Be honest in reports. If LinkedIn capped a result set at 100, say the list is
partial. If someone's connections were hidden, say that row is incomplete. A
confident-sounding wrong answer about someone's network is worse than a hedged
right one.

## What it deliberately does not do

- **It doesn't apply to anything.** It stops at "the job description is on your
  disk." Reading, deciding, and applying are the user's job. Don't offer to apply.
- **It doesn't contact anyone.** `/map-company-connections` writes a table of who
  could introduce the user to whom. Deciding who to ask, and how to ask, is the
  human's call — that judgment is what makes a warm intro warm.
- **It doesn't rate connections on the user's behalf.** Only the user knows whether
  they can call someone out of the blue. Never infer a rating from a headline.
- **It doesn't run on a schedule.** Every run is manually triggered.
- **It never solves a CAPTCHA, and never logs in for the user.**

## Repository layout

```
.
├── README.md
├── CLAUDE.md                    # this file
├── .claude/skills/
│   ├── setup.md                 # first-run config (3 questions)
│   ├── find-jobs.md             # LinkedIn search → jobs/inbox/
│   ├── build-connections.md     # LinkedIn 1st-degrees → config/connections.csv
│   ├── rate-connections.md      # interactive High/Medium/Low rating
│   └── map-company-connections.md   # the main event: warm intro paths
├── reference/
│   └── linkedin-browser-playbook.md  # READ THIS before touching LinkedIn
├── config/                      # gitignored except *.example.*
├── docs/
│   └── sample-output.md         # fictional example of what the tool produces
└── jobs/                        # gitignored entirely
    ├── inbox/                   # job descriptions, one markdown file each
    └── seen.json                # dedup index so jobs don't resurface
```

## The skill dependency chain

This is the thing to understand before helping anyone debug:

```
/setup → /build-connections → /rate-connections → /map-company-connections
                                                 ↑
                          /find-jobs ────────────┘  (independent; suggests companies to map)
```

`/map-company-connections` sorts intro paths by how warm the bridge is. Those
ratings come from `/rate-connections`, which needs the connections list from
`/build-connections`. **Skip the rating step and every path reads "Unrated"** and
the output can't be ranked — which is most of the value. If a user reports the tool
is "not that useful," check whether they rated anything. That's usually it.

They don't need to rate everything. Fifty is enough to be useful. Say so — users
who think they need all 400 tend to stall out and never see the payoff.

## Conventions

- **`reference/linkedin-browser-playbook.md` is the source of truth for anything
  browser-related.** Every LinkedIn quirk, extraction constraint, and failure mode
  lives there. When LinkedIn changes its UI, fix the playbook — the skills
  reference it rather than duplicating it.
- All user data lives in `config/` (gitignored) and `jobs/` (gitignored).
  `config/*.example.*` files use a fictional persona (Sam Chen, a Senior PM in
  e-commerce) and are the only committed examples — never put real data in them.
- `config/connections.csv` is the most sensitive file here: it holds the user's
  private read on their own relationships. Treat it accordingly.
- **Never write a real person's name into a file that isn't gitignored** — not in
  docs, not in a skill file, not in a code comment, not in a commit message. Use
  the fictional persona instead. This is a rule about *data*, not about files:
  `.gitignore` protects `connections.csv` perfectly and cannot do a thing about a
  name typed into a sentence.

  The pull toward real names is strong and it will pull on you. It arrives
  disguised as rigor — you'll be documenting a real finding, and the specific real
  name is what makes the finding credible rather than vague. **Accurate and
  publishable diverge the moment the subject is a real person.** "One of the user's
  mutuals" is slightly worse documentation and enormously better privacy; take that
  trade every time. If a real name feels load-bearing, that's the signal to stop.

  This is not theoretical. The sibling project this was carved out of published six
  real colleagues' names in a commit titled "portfolio polish" — the very commit
  meant to make it presentable to strangers. While writing *this rule*, the same
  mistake was made twice more in under an hour.

  `.githooks/pre-commit` checks staged files against the names in
  `connections.csv` and blocks the commit. It's a backstop for lapses, not a
  substitute for judgment — it only knows names already scraped. Activate it with
  `git config core.hooksPath .githooks` (git config can't be committed, so each
  clone needs it once).
- Markdown and CSV are the storage formats, on purpose. The user can open, edit,
  and understand every file this tool writes without any special software.
- **When writing a CSV that already exists, read its schema from its own header —
  never hardcode the documented column list.** Because the user can edit these
  files by hand (which is a feature), a real file drifts from its documentation.
  Writing an assumed schema over a drifted file silently drops columns, or raises
  *after* `open(path,'w')` has already truncated it. Write to a temp file, verify
  the row and column counts, then `os.replace`. Back up first if the file exists.
  `config/connections.csv` is gitignored hand-entered judgment — there is no git
  history to recover it from, and no way to regenerate it.

## When a user opens this project for the first time

Check whether `config/criteria.md` and `config/target_companies.csv` exist. If
either is missing, point them at `/setup` before anything else.
