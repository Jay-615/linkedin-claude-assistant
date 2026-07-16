# Sample output

What the tool actually produces. Everything here is **fictional** — the persona is
Sam Chen, a Senior PM in e-commerce, the same one used throughout
`config/*.example.*`. No real people or networks appear anywhere in this repo.

---

## `/map-company-connections Lumora Commerce`

### What Claude reports back

> **Lumora Commerce — 34 people mapped.**
>
> You have 34 second-degree connections there. That's the complete set, not a cap —
> LinkedIn stops at ~100 and we didn't hit it.
>
> **Your intro paths, by how well you know the person who'd make the intro:**
>
> - **3 High** — you can ask these bridges directly
> - **9 Medium** — you know them, might want to warm up first
> - **14 Low** — you'd be introducing yourself to the bridge, not just the target
> - **8 Unrated** — you haven't rated these bridges yet
>
> **Your best paths:**
>
> | Who | Their role | Ask |
> |---|---|---|
> | Rosa Delgado | Director of Product, Marketplace | **Priya Natarajan** (High) |
> | Tom Whitfield | Group PM, Payments | **Priya Natarajan** (High) |
> | Anika Rao | Principal PM, Growth | **Mira Sato** (High) |
> | Ben Coles | Senior PM, Catalog | **Brendan Hughes** (Medium) |
> | Yusuf Adeyemi | Head of Product Ops | **Henrik Olsen** (Medium) |
>
> **Priya Natarajan is your key here** — she bridges to 11 of the 34, including two
> of your three strongest paths. One conversation with her covers a third of your
> reach into this company.
>
> **Worth knowing:** 4 of these paths were only found by drilling in. Their cards
> showed "+N other mutual connections" and hid the good one. Rosa Delgado's card
> named Felipe Marquez and Casey Lin — both people you rated Low. Priya, your
> actual best path to her, was inside the hidden 21.
>
> **Two gaps:**
>
> - 8 bridges aren't in your ratings yet. Run `/rate-connections` — those 8 could be
>   hiding a better path than anything above.
> - 2 people (Dana Kirsch, Leo Barrett) hide their connections, so those rows are
>   incomplete. There may be a warmer route to them that LinkedIn won't show.

### What lands in `config/company_connections.csv`

```csv
company,person_name,title,degree,bridge_connections,bridge_best_rating,person_linkedin_url,scanned_date,notes
Lumora Commerce,Rosa Delgado,"Director of Product, Marketplace",2nd,Priya Natarajan;Felipe Marquez;Casey Lin,High,https://www.linkedin.com/in/rosa-delgado-product/,2026-07-16,"full bridge list via drill-in (card hid +21); rated: Priya Natarajan (High), Felipe Marquez (Low), Casey Lin (Low)"
Lumora Commerce,Tom Whitfield,"Group PM, Payments",2nd,Priya Natarajan;Henrik Olsen,High,https://www.linkedin.com/in/tom-whitfield-pm/,2026-07-16,"rated: Priya Natarajan (High), Henrik Olsen (Medium)"
Lumora Commerce,Anika Rao,"Principal PM, Growth",2nd,Mira Sato;Emma Kowalski,High,https://www.linkedin.com/in/anika-rao-growth/,2026-07-16,"full bridge list via drill-in (card hid +6); rated: Mira Sato (High), Emma Kowalski (Medium)"
Lumora Commerce,Ben Coles,"Senior PM, Catalog",2nd,Brendan Hughes;Marcus Bell,Medium,https://www.linkedin.com/in/ben-coles-pm/,2026-07-16,"rated: Brendan Hughes (Medium), Marcus Bell (Low)"
Lumora Commerce,Yusuf Adeyemi,Head of Product Operations,2nd,Henrik Olsen,Medium,https://www.linkedin.com/in/yusuf-adeyemi-ops/,2026-07-16,rated: Henrik Olsen (Medium)
Lumora Commerce,Dana Kirsch,"Staff PM, Search",2nd,Greta Lindqvist,Low,https://www.linkedin.com/in/dana-kirsch-search/,2026-07-16,connections hidden — card mutuals only; may be incomplete
Lumora Commerce,Leo Barrett,Senior Product Designer,2nd,Brendan Hughes,Medium,https://www.linkedin.com/in/leo-barrett-design/,2026-07-16,connections hidden — card mutuals only; may be incomplete
```

The CSV is the artifact. Open it in a spreadsheet, sort by `bridge_best_rating`,
and you have a call list.

**Note the `notes` column.** It records how each row was obtained — drilled vs.
card-only, complete vs. hidden. A row that says "connections hidden" is a row you
shouldn't trust as complete, and the file says so rather than quietly presenting a
partial answer as a whole one.

---

## Why the drill-in is the whole trick

Rosa Delgado's row is the example worth understanding.

LinkedIn's search result card for her showed:

> **Felipe Marquez, Casey Lin** and 21 other mutual connections

Read the card at face value and Sam's best path to a Director of Product is two
people he rated **Low** — near-strangers he'd have to introduce himself to first.
Most people would look at that and move on.

The 21 hidden names included **Priya Natarajan**, who Sam rated **High** and who is
already helping him with the Lumora process.

The card didn't just undersell the connection. It pointed at the wrong person. So
when a card truncates, the skill opens that person's shared-connections page and
enumerates every bridge:

```
https://www.linkedin.com/search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22<member_urn>%22%5D
```

`network=["F"]` (first-degree to you) + `connectionOf=[X]` = everyone who is
connected to both you and X. That's the complete bridge set.

It only drills cards that actually truncate. One search per company plus one drill
per truncated card keeps a full company map at well under a hundred page loads.

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
