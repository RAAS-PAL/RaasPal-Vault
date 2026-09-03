# RAASPAL — Project Vault

Working memory for the **RAASPAL AI Robot Solution & Proposal Generator**: what the system
currently is, why it is that way, and what it cost to find out.

Open this folder as a vault in [Obsidian](https://obsidian.md) so the `[[wiki-links]]` resolve
and the graph view works. It reads fine as plain Markdown in any editor or on GitHub, just
without the links.

## The two files

| File | What it is | When to read it |
|---|---|---|
| [index.md](index.md) | Current state — services, hosting, schema, what is built and what is not | **Start here.** Before doing anything else |
| [history.md](history.md) | Append-only session log: what changed, why, and what broke | When you need the reasoning behind a decision, or the history of a component |

`index.md` opens with a **Deployment Snapshot** giving each repo's HEAD and what is verified live.
It carries the date it was measured — treat anything older than a few days as a hint, not a fact,
and re-check before relying on it.

## The repositories this describes

The vault does not contain code. Three separate repositories do:

- `robot-recommendation-api` — Spring Boot 3.4.5 / Java 21 backend
- `robot-recommendation-web-raaspal` — Next.js console frontend
- `raaspal-rims` — Next.js inventory frontend (RIMS)

The workspace root that holds all three is deliberately **not** a git repository.

## Two things to know before touching the database

Both are written up properly in `index.md`, and both have already caused real incidents.

1. **Local development and production share one Supabase database.** Starting the backend
   locally applies pending migrations to production. Expand and contract: add, deploy, *then*
   drop. A `DROP COLUMN` once took down every inventory query while the deployed code still
   selected it.

2. **A committed `V*.sql` is already applied — never edit one in place.** Flyway checksums
   applied migrations and refuses to start on a mismatch, so editing one breaks the next deploy
   with no local symptom. Add a new migration instead.

## Conventions

- Entries in `history.md` are append-only and dated, newest last. Correct a wrong entry by
  editing it in place and saying it was wrong — a log that quietly rewrites itself is worth less
  than one that shows where it was mistaken.
- Reference files, classes and concepts as `[[wiki-links]]`, not plain text.
- Convert relative dates to absolute ones. "Last Tuesday" is useless six months on.

## Contains no secrets

Checked before publishing: no API keys, passwords, tokens or connection strings. Config is
referenced by variable name with placeholder values (`ANTHROPIC_API_KEY=sk-ant-...`).

It **does** contain internal architecture, customer names, a company tax ID and a record of past
outages. **Keep this repository private.**
