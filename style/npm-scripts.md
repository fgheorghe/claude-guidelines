# npm scripts

`package.json`'s `scripts` block is the project's command surface. The rules:

## Scripts are thin wrappers

A script entry is a one-line invocation of a real program — a Node script, a binary, a tool. Multi-line shell logic, conditionals, and inline string manipulation belong in a real script file under `scripts/`, not in `package.json`.

Good:

```json
{
  "scripts": {
    "data:ingest": "node scripts/data-ingest.js",
    "data:migrate": "node scripts/data-migrate.js"
  }
}
```

Bad — logic in `package.json`:

```json
{
  "scripts": {
    "data:ingest": "if [ -f data.json ]; then node scripts/ingest.js --input data.json; else echo 'no data'; fi"
  }
}
```

The shell pipeline belongs inside `scripts/data-ingest.js` where it gets syntax highlighting, tests, version control diffs, and code review. `package.json` carries the name; the file carries the behavior.

## Group scripts by namespace

Use `:` as a namespace separator. Scripts that operate on the same area of the system share a prefix:

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "start": "node dist/server.js",

    "lint": "eslint . --fix && npm audit --audit-level=high",
    "test": "node --test",

    "data:ingest": "node scripts/data-ingest.js",
    "data:migrate": "node scripts/data-migrate.js",
    "data:seed": "node scripts/data-seed.js",
    "data:export": "node scripts/data-export.js",

    "cron:refresh-orders": "node scripts/cron/refresh-orders.js",
    "cron:cleanup-sessions": "node scripts/cron/cleanup-sessions.js",
    "cron:rebuild-search": "node scripts/cron/rebuild-search.js",

    "db:reset": "node scripts/db-reset.js",
    "db:dump": "node scripts/db-dump.js",
    "db:restore": "node scripts/db-restore.js"
  }
}
```

Common namespaces: `data:*` for data ingestion and transformation, `cron:*` for scheduled jobs, `db:*` for database lifecycle, `build:*` if the build has multiple stages, `gen:*` for code generation. The unnamespaced scripts (`dev`, `build`, `start`, `lint`, `test`) are the daily-driver commands that everyone runs constantly; the namespaced ones are operational tooling.

Within a namespace, alphabetize. `data:export`, `data:ingest`, `data:migrate`, `data:seed` reads as a table of contents.

## The `lint` script

Every project has a `lint` script that runs the linters and the audit. The shape depends on whether the project has CSS files.

Without CSS:

```json
{
  "scripts": {
    "lint": "eslint . --fix && npm audit --audit-level=high"
  }
}
```

With CSS (also runs Stylelint, and Astro's check if applicable):

```json
{
  "scripts": {
    "lint": "eslint . --fix && stylelint '**/*.css' --fix && npm audit --audit-level=high && npx astro check"
  }
}
```

The `--fix` flag auto-applies safe fixes. `npm audit --audit-level=high` fails the lint if there are high-severity advisories without a clean fix path. Run this script after every change set and fix the reported issues before considering the work complete; see `linting.md` for the workflow rule.

## What goes in `scripts/` vs. `package.json`

A useful test: if the script can be read aloud as one sentence ("run the dev server", "build the project", "ingest data"), it belongs in `package.json` as a one-line invocation. If it needs commas, conditionals, or multiple verbs, it belongs in a `scripts/` file.
