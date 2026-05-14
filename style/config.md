# Application configuration

Application configuration lives in a `config.ini` file at the project root, parsed at startup with the `ini` module. `config.ini` is gitignored; `config.ini.template` is committed and shows every section and key with reasonable defaults.

## Why `.ini`, not JSON or YAML

Three reasons. INI files have sections, which match how config is naturally grouped (database, locale, features). INI files have first-class comment support, which JSON does not. INI files do not have the YAML-indentation footgun where a stray space changes meaning.

The `ini` module from npm is the project standard. Parsing is one call:

```js
import { readFileSync } from 'node:fs';
import { parse } from 'ini';

/** The parsed application configuration loaded once at startup. */
const config = parse(readFileSync('./config.ini', 'utf-8'));

const dbUser = config.database.user;
const language = config.locale.language;
```

## File layout

Sections are named after the area of the system they configure. Keys within a section are lowercase with underscores or hyphens, matching the parser's output:

```ini
[database]
user = dbuser
password = dbpassword
database = use_this_database

[locale]
language = english
currency = EUR

[search]
max_results = 50
ranking = relevance

[cache]
ttl_seconds = 300

[features]
price_tracking = false
```

Common sections by responsibility: `[database]`, `[locale]`, `[search]`, `[cache]`, `[features]`, `[logging]`, `[email]`, `[storage]`, plus per-integration sections like `[stripe]`, `[openai]`. The rule is: one section per logical area, named after the area, not after the module that reads it.

## `config.ini.template`

`config.ini` is gitignored (it contains values, including secrets). `config.ini.template` is committed and lists every section and key the app uses, with comments explaining what each is for and reasonable defaults:

```ini
[database]
; The Postgres user the app connects as.
user = app

; Required: the password for the database user. Generate with: openssl rand -hex 32
password =

; The database name to connect to.
database = app_dev

[locale]
; The default language for AI-generated content. One of: english, romanian, french.
language = english

; The default currency for prices shown to users. ISO 4217 code.
currency = EUR

[features]
; Whether to enable the experimental price-tracking UI.
price_tracking = false
```

When a new section or key is added to the code, `config.ini.template` gains a corresponding entry with the same comment style. The two files stay in sync; the template is the documented surface area of the project's configuration.

A new developer's setup is `cp config.ini.template config.ini` plus editing the values they care about.

## Comment style in config files

The same comment discipline that governs JS code applies to `config.ini` and `config.ini.template`. INI files use `;` as the comment marker. The rules:

- Comments are sentence fragments, not full sentences. Start with "Used to...", "Used as...", "The...", or an imperative verb ("Load...", "Add...").
- Describe the *role* of the binding, not its type or mechanics. The config shows *how*; the comment shows *what for*.
- Do not narrate obvious config. No `; the database user` above `user = app`.
- Comment the non-obvious: workarounds, why a default is what it is, links to upstream docs explaining the option, security implications.
- One comment per declaration, one declaration per comment.
- Voice is impersonal and present-tense. No "we", no "I", no "this will...".
- `TODO:` and `NOTE:` markers are encouraged for known issues and non-obvious behavior.

Good — explains the choice:

```ini
[search]
; The maximum number of results returned per search query. Higher values increase
; memory pressure on the search index; 50 is the tested sweet spot for current data sizes.
max_results = 50
```

Bad — narrates the variable name:

```ini
[search]
; max_results is the maximum results
max_results = 50
```

Good — flags the gotcha:

```ini
[cache]
; The TTL for cached search results, in seconds. NOTE: changes here invalidate
; the on-disk cache and force a full rebuild on next startup.
ttl_seconds = 300
```

## Application code never reads environment variables

`config.ini` is the *only* source of application configuration. Application code never reads `process.env`, never imports `dotenv`, never references environment variables for anything beyond the bare basics that exist on every Node process (`process.cwd()`, `process.argv`).

The reasoning: `.env` is a deployment artifact. It tells the container orchestration layer how to assemble the stack — what ports to publish, what host paths to mount, what project name to use. It is not present inside the container. The application running inside the container has no `.env` file to read, no environment variables propagated from it, and no way to know what was in it.

Anything the application needs to behave correctly is in `config.ini`. Anything the deployment system needs to assemble the stack is in `.env`. The two sets do not overlap.

Practical consequences:

- **Secrets the application uses go in `config.ini`.** Third-party API keys, database passwords (read by the app to connect), encryption keys — all in `config.ini`. `config.ini` is gitignored; production deployments provide a real `config.ini` via a mounted file, not via environment variables.
- **No `process.env.X` in application code.** If you find yourself reaching for `process.env.SOMETHING`, the value belongs in `config.ini` instead.
- **No `dotenv` as an application dependency.** The package may exist as a tool for local dev scripts that run outside the container, but it must not appear in any code path that runs inside the deployed application.
- **No conditional fallback between env and config.** Patterns like `process.env.DB_HOST ?? config.database.host` are wrong. `config.database.host` is the only source; if it is not set, that is a configuration error to surface, not a hole to fill from the environment.

For scripts that genuinely run outside the container (a deployment helper, a local CLI), reading environment variables is fine — those are not application code. The rule applies to anything that runs inside the deployed stack.

## How `config.ini` reaches a deployed instance

In dev, the developer's `config.ini` sits next to the source and is read directly. In production, the deployment system places `config.ini` at the same path the application expects (typically the working directory or an explicit `/app/config.ini`), populated with production values. The application's code path is identical in both — read `config.ini`, parse, use. Where the file came from is not the application's concern.

## Loading config once at startup

Parse `config.ini` once at startup, freeze the result, pass it explicitly to anything that needs it. Do not re-parse on every read; do not stash it as a mutable global.

```js
import { readFileSync } from 'node:fs';
import { parse } from 'ini';

/** The parsed application configuration, loaded once at startup. */
export const config = Object.freeze(parse(readFileSync('./config.ini', 'utf-8')));
```

Modules that need config import the frozen object. This makes config access greppable, makes test injection straightforward (override the import or pass an alternative object), and prevents the "where did this value come from" problem of scattered re-reads.
