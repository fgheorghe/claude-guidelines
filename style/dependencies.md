# Dependencies: prefer well-known modules

For solved problems, use the established npm module. Do not reinvent dates, logging, config parsing, CLI formatting, validation, HTTP clients, or other categories with mature, well-known solutions. The ecosystem has converged on good answers; use them.

## Categories where you should always reach for a library

Never hand-roll these:

- **Dates and times.** Use `date-fns` (functional, tree-shakeable) or `luxon` (object-oriented, timezone-aware). Never write your own date parser, formatter, or arithmetic. `new Date('2024-01-15')` plus manual offset math is a bug factory.
- **Logging.** Use `winston`. Flexible, transport-rich, ecosystem-standard. Never `console.log` in anything beyond a script or a one-off.
- **Config parsing.** Use `ini` for application config (the project standard; see `config.md`). Never write your own config parser, and never read environment variables from application code — `config.ini` is the only source of configuration the application sees.
- **CLI formatting.** Use `chalk` or `picocolors` for colors, `ora` for spinners, `cli-progress` for progress bars, `boxen` for boxes, `cli-table3` for tables. Never emit raw ANSI escape codes.
- **CLI argument parsing.** Use `commander` or `yargs`. Never parse `process.argv` by hand.
- **HTTP clients.** Use `undici` (Node 18+, fast, modern), `ky` (browser/universal, fetch-based), or native `fetch`. Avoid `request` (deprecated) and `axios` unless the project already uses it.
- **Schema validation.** Use `zod` (TypeScript-first, inferrable) or `valibot` (smaller). Never validate object shapes with hand-written `if` ladders.
- **UUIDs.** Use `uuid` or `nanoid`. Never `Math.random().toString(36)`.
- **Hashing and crypto.** Use Node's built-in `node:crypto`. Never roll your own.
- **Retry and backoff.** Use `p-retry` or `async-retry`. Never write a `for` loop with `setTimeout` for retries.
- **Concurrency limiting.** Use `p-limit` or `p-queue`. Never write your own semaphore.
- **Deep equality, cloning, debouncing.** Use `lodash` (or `lodash-es` for tree-shaking, or individual `lodash.debounce` packages for tiny installs).

## Categories where the standard library is enough

Do *not* reach for a library for:

- String trimming, splitting, joining, padding (`String.prototype` covers it).
- Array map/filter/reduce/find/some/every (native).
- Basic object spread, destructuring, `Object.keys/values/entries` (native).
- JSON parse/stringify (native; use the second and third args of `JSON.stringify` for pretty-printing).
- Promise composition: `Promise.all`, `Promise.allSettled`, `Promise.race` (native).
- File I/O (`node:fs/promises`).
- Path manipulation (`node:path`).
- URL parsing (`URL` and `URLSearchParams` are global).

The native APIs have caught up. Adding `is-array`, `left-pad`, `is-odd` to a project is a smell.

## The decision rule

Before writing more than ~20 lines of code that feels like infrastructure rather than business logic, ask:

1. Is this a solved problem? (dates, logging, parsing, retries, CLI presentation)
2. Is there a well-known module with >1M weekly downloads that solves it?
3. Does the project already have a dependency that solves it?

If yes to (1) and (2), use the module. If yes to (3), use what's already there — do not introduce a second library for the same job.

## When proposing a new dependency

When introducing a dependency the project doesn't already have, in the same response:

- Name the package and its weekly download count or a one-line credibility note ("maintained by the Node.js core team", "ecosystem standard for X").
- State which category from the list above it covers.
- Show the install command (`npm install <pkg>` or `npm install -D <pkg>` for dev-only).
- Use the smallest viable option. `picocolors` over `chalk` if all you need is colors. `nanoid` over `uuid` if you don't need RFC 4122.

If unsure whether the user wants a new dependency, ask before adding it — but lead with the recommendation, not a blank question.

## When *not* to add a dependency

- The project has a working solution for the same job. Don't add a second.
- The need is one function call's worth of work and the native API covers it.
- The dependency would be the only one of its kind for a trivial use (e.g., adding `lodash` solely to use `_.isEmpty(x)` when `x == null || (typeof x === 'object' && Object.keys(x).length === 0)` is the whole call site).
- The dependency is unmaintained (last publish >2 years ago, open issues piling up) or has known security advisories without a clean upgrade path.

## Worked examples

**Bad — reinventing date formatting:**

```js
function formatDate(date) {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');

    return `${year}-${month}-${day}`;
}
```

**Good — use the library:**

```js
import { format } from 'date-fns';

const formatted = format(date, 'yyyy-MM-dd');
```

**Bad — hand-rolled progress output:**

```js
for (let index = 0; index < items.length; index++) {
    process.stdout.write(`\rProcessing ${index + 1}/${items.length}`);
    await processItem(items[index]);
}
process.stdout.write('\n');
```

**Good — use a progress bar:**

```js
import cliProgress from 'cli-progress';

const bar = new cliProgress.SingleBar({}, cliProgress.Presets.shades_classic);
bar.start(items.length, 0);

for (const item of items) {
    await processItem(item);
    bar.increment();
}

bar.stop();
```

**Bad — hand-rolled retry:**

```js
async function fetchWithRetry(url) {
    for (let attempt = 0; attempt < 3; attempt++) {
        try {
            return await fetch(url);
        } catch (error) {
            if (attempt === 2) {
                throw error;
            }
            await new Promise((resolve) => setTimeout(resolve, 1000 * (attempt + 1)));
        }
    }
}
```

**Good — use the library:**

```js
import pRetry from 'p-retry';

const response = await pRetry(() => fetch(url), { retries: 3 });
```

**Bad — adding a library for one trivial call:**

```js
import isNil from 'lodash.isnil';

if (isNil(value)) {
    return defaultValue;
}
```

**Good — native is fine:**

```js
if (value == null) {
    return defaultValue;
}
```

(`value == null` is the one place loose equality earns its keep — it matches both `null` and `undefined`, and nothing else.)
