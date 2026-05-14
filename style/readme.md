# The `README.md`

The README is the project's front door. It exists to answer three questions for someone who has just opened the repo:

1. **What is this thing?** (one paragraph)
2. **How do I run it?** (concrete commands)
3. **What do I need to know that isn't obvious from the code?** (gotchas, decisions, links to deeper docs)

Anything that does not answer one of those three questions does not belong in the README. The README is not a tour of the codebase, not a copy of the file tree, not a re-narration of what the code already says.

## The structure

````markdown
# <Project name>

<One-paragraph description: what this is, who it's for, what it does. Three to five sentences. Plain English, not marketing.>

## Quick start

<The exact commands, in order, to go from "fresh clone" to "running stack." Nothing else.>

## What's here

<Three to five bullet points naming the top-level concerns and where to find them. Not a file tree — a sketch of the moving parts.>

## Conventions

<One sentence per convention, with a link to the relevant doc.>

## Operating notes

<Anything that would surprise a new reader. Gotchas. Why a weird thing is the way it is. Limits and quirks. Empty if there aren't any.>
````

That's the whole shape. A typical README runs 60 to 150 lines. If yours is approaching 500, you are documenting things that belong elsewhere.

## Voice and style

The comment-style rules from `comments.md` apply to README prose too:

- Sentence fragments where they read naturally; full sentences where they don't. Lists are fragments, prose paragraphs are sentences.
- Describe the role and the use, not the mechanics. "Built with Fastify because the API needs Swagger docs derived from validation schemas" — not "We use Fastify which is a Node.js web framework that..."
- Do not narrate obvious things. No "this is a Node.js project" above a `package.json` reference; the reader can see the `package.json`.
- Comment the non-obvious. If the project uses Podman locally and Docker in prod, say so. If a state service has a quirk that took a day to debug, write it down.
- Voice is impersonal and present-tense. Not "we use winston for logging." Just "Logging via winston." Or "Uses winston for structured logging." First-person plural in a README is corporate-blog tone; avoid it.
- `TODO:` and `NOTE:` markers are encouraged for known issues and rough edges. Be honest about what isn't done.

## What goes in the README vs. elsewhere

The README is one of several documentation surfaces. The split:

- **`README.md`** — orientation. The three questions above.
- **`DOCKER.md`** — container and compose conventions. Linked from README; not duplicated in it.
- **Style guide** — code conventions. Linked from README; not duplicated in it.
- **`config.ini.template`** — every config option with a comment explaining it. The README mentions config exists and points at the template; it does not list every option.
- **`.env.template`** — every environment variable with a comment. Same as above.
- **Swagger UI at `/docs`** — API endpoint reference. The README mentions where to find it; the spec itself is generated from route schemas.
- **JSDoc on exports** — function and module documentation. The README does not summarize function signatures.

When you're tempted to write something in the README, ask: is this information already in one of those other places? If yes, the README links to it. If no, the README is the right home.

## Quick-start commands are the heart of the README

The "Quick start" section is the highest-value content in the README. It must be:

- **Copy-pasteable.** A reader should be able to copy the commands into their terminal and have the project run. No `<replace this>` placeholders that the reader has to figure out.
- **Complete.** Every command needed, in order. If `cp config.ini.template config.ini` is required before `docker compose up`, both commands are listed.
- **Tested.** Run the quick start on a fresh checkout periodically. If it fails, the README is wrong, not the project setup.

Example:

````markdown
## Quick start

```bash
git clone <repo-url>
cd <project>
cp .env.template .env
cp config.ini.template config.ini
# Edit .env to set COMPOSE_PROJECT_NAME and DB_PASSWORD.
# Run the chown steps named in docker-compose.yml for state services.
docker compose up
```

The stack is available at http://127.0.0.1:3000. Swagger UI at /docs.
````

That's eight commands and two pointers. The reader is running the project in under a minute. They do not need to know what Fastify is, what the multi-stage Dockerfile does, or which service is on which port for service-to-service traffic — those are details that become relevant later, after the thing is running.

## What "What's here" looks like

Not a file tree. A sketch of the moving parts:

````markdown
## What's here

- **`src/`** — the Fastify API. Routes under `routes/`, shared schemas under `schemas/`, services that talk to the outside world under `services/`.
- **`scripts/`** — operational tooling. `data:*` scripts for ingestion and migration, `cron:*` scripts for scheduled work. See `package.json` for the full list.
- **`docker-compose.yml`** — the stack: web service, cron, database, search. Conventions documented in `DOCKER.md`.
- **`config.ini.template`** — every configuration option with comments. Copy to `config.ini` and edit.
````

Four bullets. Each one is a sentence fragment naming a directory or file and what role it plays. The reader who wants more opens the directory; the README does not pretend to be the file tree.

## Conventions section

One line per convention, with a link:

````markdown
## Conventions

- Code style: see [style guide](./STYLE.md).
- Container conventions: see [DOCKER.md](./DOCKER.md).
- API documentation: served at /docs, generated from Fastify route schemas.
- Logging: structured JSON via winston.
- Config: `config.ini` (gitignored), template at `config.ini.template`.
````

The reader who wants to contribute knows where to look without the README explaining each convention itself.

## Operating notes

This is the catch-all for things that would surprise a new reader. Empty if there are none.

````markdown
## Operating notes

- The cron container runs a long-lived Node scheduler, not `crond`. Adding a scheduled job means editing `scripts/scheduler.js`, not a crontab.
- Meilisearch's data directory must be owned by uid 1000 on the host before first run. The compose file documents the chown step inline.
- Local dev uses Podman; production uses Docker. The `userns_mode: keep-id` directive in the compose file exists for Podman and is ignored by Docker with a harmless warning.
- TODO: the data ingestion script does not yet handle the case where Steam's API rate-limits mid-batch. Retries are best-effort.
````

These are the things that would cause a new contributor to file a confused issue if they encountered them without warning. Write them down once, in the README, with the context that makes them make sense.

## What to remove from existing READMEs

When refreshing an existing README, the typical cuts are:

- Detailed installation prose ("First, install Node.js. Then install npm. Then run npm install..."). Replace with the quick-start command block.
- Lists of every npm script. Replace with "See `package.json` for the full list of scripts; the daily-driver commands are `dev`, `start`, `lint`, `test`."
- API endpoint documentation. Replace with "Swagger UI at /docs."
- File-tree dumps. Replace with the four-bullet "What's here" sketch.
- Architectural overviews that restate what's obvious from the code. Replace with "Operating notes" entries for the parts that aren't obvious.
- "About the project" sections that read like marketing copy. Replace with the one-paragraph description at the top.
- "Contributing" sections that say "read the style guide and run the linter before submitting." Replace with one line under "Conventions" pointing at the style guide.

A good rule of thumb: every sentence in the README should answer one of the three questions, or link to a doc that does. Sentences that don't are cuts.
