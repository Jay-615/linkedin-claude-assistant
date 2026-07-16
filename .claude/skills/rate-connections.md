---
name: rate-connections
description: Interactive walk-through to apply High/Medium/Low ratings to Unrated rows in config/connections.csv. Presents one connection at a time with name, headline, and notes; writes each rating back to the CSV immediately so a mid-session crash doesn't lose progress. Reminds the user that bulk-editing the CSV in a spreadsheet is an equally valid path.
---

# /rate-connections — interactive rating walk-through

You are helping the user assign a qualitative rating to each `Unrated` row in
`config/connections.csv`. These ratings are what make `/map-company-connections`
useful: it sorts intro paths by how warm the bridge is, so an unrated network
produces a table that can't be ranked. Even partial progress is valuable — every
rated connection makes the output sharper.

This is a **judgment task, not a data task.** The user is the only one who knows
whether they can call someone out of the blue. Never guess a rating on their
behalf, and never infer one from a headline.

This skill is **one of two paths** the user has for rating. The other is
bulk-editing the CSV directly in a spreadsheet. Both are first-class. Make that
clear in the preamble — don't pretend this is the only way.

---

## Opening message to the user

Print this preamble before doing any work:

> I'll walk you through your Unrated LinkedIn connections one at a time. For each,
> you'll choose **High**, **Medium**, **Low**, or **Skip**. Each rating is written
> back to `config/connections.csv` immediately, so if anything interrupts us, you
> keep what you've rated so far.
>
> Rating rubric:
>
> - **High** — You know these people well. You can ask them for help right off the bat.
> - **Medium** — You know these people, but not very well, or you've been out of touch. You may want to reconnect a bit before asking for help.
> - **Low** — You don't really know these people. You may want to introduce yourself and establish a real connection before asking for help.
>
> Heads up: this is one path, not the only one. If you'd rather batch-rate in a
> spreadsheet, open `config/connections.csv` in Excel, Numbers, or anything else —
> sort by headline (groups people by company) and mass-update the `rating` column.
> Both paths produce the same result, and you can stop here and switch at any time.

---

## Step 1 — Read the CSV and find Unrated rows

1. Read `config/connections.csv`. If it doesn't exist:

   > I don't see `config/connections.csv` yet. Run `/build-connections` first —
   > that pulls your LinkedIn 1st-degree connections into the file. Then come back
   > here to rate them.

   Bail.

2. Parse the CSV using Python's `csv` module (via `Bash`). Keep the original column
   order: `linkedin_url,name,headline,rating,last_interaction,notes`.

3. Filter to rows where `rating` is empty, `"Unrated"`, or any whitespace variant.
   Count them.

4. If zero Unrated rows remain:

   > All your connections are already rated. (High: N / Medium: N / Low: N) —
   > nothing to do here.

   Stop.

5. Tell the user the count and ask via `AskUserQuestion`:
   - **Start rating** (recommended)
   - **Show me the breakdown first** — print the current rating distribution + the
     first 5 Unrated headlines so the user can sanity-check before committing time
   - **Cancel**

---

## Step 2 — Optional sort order

Ask once, before the loop, how the user wants to walk the list:

- **By headline (default)** — groups by company / job title, so they'll often rate
  a cluster of co-workers in a row. This is genuinely faster: it keeps the user in
  one mental context instead of context-switching every row.
- **By name** — alphabetical
- **As-is** — whatever order the CSV is in

Default to "by headline."

---

## Step 3 — The rating loop

For each Unrated row, in the chosen sort order:

1. Show the user:

   ```
   [12 / 487]    <Name>
                 <Headline>
                 Notes: <existing notes, or "(none)">
                 Profile: <linkedin_url>
   ```

   The progress counter matters — it lets the user see how much is left and decide
   whether to push through or stop.

2. Use `AskUserQuestion` to ask: "How well do you know <Name>?"

   Options:
   - **High** — You know them well; could ask for help directly
   - **Medium** — You know them, but not well, or have been out of touch
   - **Low** — You don't really know them; would need to introduce yourself
   - **Skip** — Leave Unrated for now, move to next
   - **Stop** — Save and exit the loop

   The description on each option carries the rubric, so the user never has to
   remember what each rating means.

3. **Immediately after the user answers, write the updated row back to
   `config/connections.csv`.** Do not batch writes — a crash or a context cutoff
   between rows must not lose the user's judgment.

   The safest pattern: read the CSV, mutate the in-memory representation, write it
   all back. Do this on every rating. A few thousand rows write in milliseconds —
   the I/O cost is nothing next to the cost of losing user input.

4. Optional: after every ~25 rated rows, briefly remind the user they can stop any
   time and that bulk-editing is still an option:

   > You've rated 25 so far. (If you'd rather finish in a spreadsheet, choose
   > **Stop** here and open `config/connections.csv` directly — your progress is
   > saved.)

5. If the user chooses **Stop**, break the loop cleanly and go to Step 4.

---

## Step 4 — Summary

When the loop ends, print:

```
Rating session — done.
  Rated this session: 47 (High: 8 / Medium: 14 / Low: 25)
  Skipped this session: 3 (still Unrated)
  Total in config/connections.csv: 487
    High: 39 / Medium: 36 / Low: 39 / Unrated: 373
```

Then a short closing line:

> Re-run `/rate-connections` any time to pick up where you left off — Unrated rows
> are the only ones I revisit. Or open `config/connections.csv` directly if a batch
> session feels easier from here.
>
> You have enough rated to start: try `/map-company-connections <a target company>`.

Say that last line whenever they've rated even a handful. Waiting until the list is
"done" is the main way users stall out — the tool works fine on a partially rated
network, and seeing real output is what makes the rest of the rating feel worth it.

---

## Edge cases

- **CSV with manual edits in flight.** If the user has the CSV open in a
  spreadsheet while running this skill, write conflicts are possible. Don't try to
  detect this — assume the user knows what they're doing. If they're using both
  paths, they should close the spreadsheet first.

- **Rating with bad values.** If a row's `rating` contains something outside
  `{High, Medium, Low, Unrated, ""}` (e.g. a typo from manual editing), treat it as
  `Unrated` for this loop, but **do not overwrite the raw value until the user
  gives a new rating** — preserve the typo in case it was intentional. Surface it:

   > Note: this row's existing `rating` value is `"high "` (with a trailing space).
   > I'll treat it as Unrated for now; choose a new rating to clean it up.

- **Row count changed mid-session.** If `/build-connections` runs concurrently and
  adds rows, you'll still be iterating the snapshot from Step 1. That's fine — the
  next `/rate-connections` picks up the new Unrated rows.

- **Stop during the very first prompt.** Exit cleanly with the standard summary
  (showing "Rated this session: 0").

---

## Why immediate-write matters

Each rating is one quick row update. The user is doing the slow part — the
judgment. This skill's job is to make sure none of that judgment is ever lost to a
refresh, a context cutoff, or an accidental window close. Always write to disk
before asking the next question.
