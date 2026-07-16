# config/

This folder holds your personal configuration. Two kinds of files live here.

## `.example.*` files (committed to git, shared with everyone)

`criteria.example.md`, `target_companies.example.csv`, and
`connections.example.csv` are committed to the repo. They show a fictional
persona — Sam Chen, a Senior PM in e-commerce — so anyone browsing the repo can
see exactly what the data model looks like, and so new users have a working
starting point to copy from.

**Don't put your real data in these files.** They are reference material.

## Real config files (gitignored, stay on your machine)

The real files — `criteria.md`, `target_companies.csv`, `connections.csv`, and
`company_connections.csv` — are gitignored. They never get committed and never
get pushed. Your job criteria, target companies, and LinkedIn network are yours
alone.

This matters more here than in most tools. `connections.csv` holds your private
read on your own professional relationships: who you know well, who you've lost
touch with, who you'd feel awkward asking for a favor. That file is the whole
point of the tool, and it is the last thing you'd want to leak. It stays local.

## How to set them up

Run `/setup` once. It asks a few questions and writes the real files for you.
You can edit them by hand any time afterwards — they're plain markdown and CSV.

To populate them by hand instead, copy each `*.example.*` file to the same name
without `.example` (for example, `cp criteria.example.md criteria.md`) and edit.
The `.gitignore` keeps your real files out of git automatically.

## What lives where

| File | Purpose | Written by |
|---|---|---|
| `criteria.md` | Roles, locations, seniority, comp floor, job types | `/setup` |
| `target_companies.csv` | Companies you're tracking, with priority | `/setup` |
| `connections.csv` | Your LinkedIn 1st-degrees with High/Medium/Low ratings | `/build-connections`, rated by `/rate-connections` |
| `company_connections.csv` | 2nd-degrees at each target company + the bridges to them | `/map-company-connections` |
