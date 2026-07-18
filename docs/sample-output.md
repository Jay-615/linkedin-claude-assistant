# Sample output

What the tool actually produces. Everything here is **fictional** — the persona is
Sam Chen, a Senior PM in e-commerce, the same one used throughout
`config/*.example.*`. No real people or networks appear anywhere in this repo.

---

## `/map-company-connections Lumora Commerce`

### What Claude reports back

> **Lumora Commerce — 34 second-degrees scanned, 12 kept.**
>
> You have 34 second-degree connections there. 12 of them are reachable through a
> bridge you rated **High or Medium** — those are the rows I kept. The other 22
> (reachable only through Low or unrated bridges) I left off, so the list stays a
> scannable "who can actually introduce me."
>
> **Your warm paths:**
>
> | Who | Their role | Warm bridge |
> |---|---|---|
> | Rosa Delgado | Director of Product, Marketplace | **Priya Natarajan** (High) |
> | Tom Whitfield | Group PM, Payments | **Priya Natarajan** (High) |
> | Anika Rao | Principal PM, Growth | **Mira Sato** (High) |
> | Ben Coles | Senior PM, Catalog | **Brendan Hughes** (Medium) |
> | Yusuf Adeyemi | Head of Product Ops | **Henrik Olsen** (Medium) |
>
> **Priya Natarajan is your key here** — she bridges to 4 of the 12, including two of
> your strongest paths. One conversation with her covers a third of your warm reach
> into this company.
>
> **Worth knowing:** 3 of these were only found by drilling in. Their cards showed
> "+N other mutual connections" and hid the good one. Rosa Delgado's card named Felipe
> Marquez and Casey Lin — both people you rated Low — while Priya, your actual best
> path to her, sat inside the hidden 21.
>
> **Two notes:**
>
> - 4 second-degrees were dropped because their only bridge was still *unrated* — rate
>   those bridges (`/rate-connections`) and they'll surface next run.
> - 1 person (Leo Barrett) hides their connections, so that row falls back to the card's
>   named mutuals and may be incomplete.

### What lands in `config/company_connections.csv`

```csv
company,person_name,title,high_bridges,medium_bridges,person_linkedin_url,scanned_date,notes
Lumora Commerce,Rosa Delgado,"Director of Product, Marketplace",Priya Natarajan,,https://www.linkedin.com/in/rosa-delgado-product/,2026-07-16,full bridge list via drill-in (card hid +21)
Lumora Commerce,Tom Whitfield,"Group PM, Payments",Priya Natarajan,Henrik Olsen,https://www.linkedin.com/in/tom-whitfield-pm/,2026-07-16,
Lumora Commerce,Anika Rao,"Principal PM, Growth",Mira Sato,Emma Kowalski,https://www.linkedin.com/in/anika-rao-growth/,2026-07-16,full bridge list via drill-in (card hid +6)
Lumora Commerce,Ben Coles,"Senior PM, Catalog",,Brendan Hughes,https://www.linkedin.com/in/ben-coles-pm/,2026-07-16,
Lumora Commerce,Yusuf Adeyemi,Head of Product Operations,,Henrik Olsen,https://www.linkedin.com/in/yusuf-adeyemi-ops/,2026-07-16,
Lumora Commerce,Leo Barrett,Senior Product Designer,,Brendan Hughes,https://www.linkedin.com/in/leo-barrett-design/,2026-07-16,connections hidden — card mutuals only; may be incomplete
```

(A representative slice — the real file holds all 12 kept rows.)

The CSV is the artifact. Open it in a spreadsheet, scan the `high_bridges` column, and
you have a call list. Bridge names are stored plain — no `, MBA` / `, PHR` credential
noise — so they're easy to read.

**Note the `notes` column.** It records how each row was obtained — drilled vs.
card-only, complete vs. hidden. A row that says "connections hidden" is one you
shouldn't trust as complete, and the file says so rather than quietly presenting a
partial answer as a whole one.

---

## Why the drill-in is the whole trick

Rosa Delgado's row is the example worth understanding.

LinkedIn's search result card for her showed:

> **Felipe Marquez, Casey Lin** and 21 other mutual connections

Read the card at face value and Sam's best path to a Director of Product is two people
he rated **Low** — near-strangers he'd have to introduce himself to first. Most people
would look at that and move on.

The 21 hidden names included **Priya Natarajan**, who Sam rated **High** and who is
already helping him with the Lumora process.

The card didn't just undersell the connection — it pointed at the wrong person. So when
a card truncates, the skill opens that person's shared-connections page and enumerates
every bridge, then keeps only the warm ones:

```
https://www.linkedin.com/search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22<member_urn>%22%5D
```

`network=["F"]` (first-degree to you) + `connectionOf=[X]` = everyone connected to both
you and X. That's the complete bridge set. The skill drills **every** truncated card —
even ones already showing a warm name — because the hidden "+N" could be someone who
knows this person far better than the names on the card.

---

## `/find-jobs`

```
LinkedIn scan — done. 6 searches (3 roles × 2 location modes).

  31 cards found → 18 passed filters
  12 were dedupes from prior runs — skipped
  6 new postings written to jobs/inbox/

  Filter drops: 8 too senior (VP+), 4 wrong location, 1 comp below floor.
  1 posting's JD is hosted off-LinkedIn — flagged in the file.

Read what looks good in jobs/inbox/. For anything at a company you care about,
run /map-company-connections <company> to see who can introduce you.
```

Each survivor becomes a markdown file with the full job description, so the JD is
readable offline and won't vanish when the posting comes down:

```
jobs/inbox/linkedin-2026-07-16-lumora-commerce-senior-product-manager.md
```

That's where this tool stops. It doesn't score the job, write the cover letter, or
click Apply. It gets the description onto your disk and tells you who you know
inside — the two things that are tedious by hand. The rest is yours.
