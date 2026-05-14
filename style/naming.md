# Naming

- Public functions and variables: camelCase.
- Constants: SCREAMING_SNAKE_CASE.
- Regexes: `re` prefix (`rePathSeparator`, `rePrimitive`).
- Internal workers behind a public function: `base` prefix (`baseClone` for `clone`).
- Factories that produce functions: `create` prefix (`createWrap`).
- Boolean variables and properties read as questions: `isActive`, `hasPermission`, `canEdit`.
- Type-tag strings declared once at the top, reused everywhere.

## Naming functions and methods

**When you find yourself typing `And` in a function name, stop and split. Then name the orchestrator after what the combination *is*, not what it *does step by step*.**

That single rule catches most of what goes wrong with function naming. The rest of this section is how to apply it.

### The naming rule

- Start with a verb. Functions do things; their names say what.
- Be specific about *what* and *to what*. `process(user)` is a placeholder; `validateUserEmail(user)` is a name.
- Match the return shape to the verb:
  - `getX` returns X.
  - `findX` returns X or null.
  - `isX`, `hasX`, `canX`, `shouldX` return booleans.
  - `ensureX` returns X, creating it if absent.
  - `requireX` returns X or throws.
- Don't lie. A function called `getUser` that also writes to a cache is misnamed.

### The conjunction test

Watch for these words in function names: `And`, `Or`, `Then`, `With`, `After`, `Before`.

They are diagnostic. Each one is the function telling you it has more than one responsibility welded together. When you see one, split — and name the resulting orchestrator after what the combination *is*, not the steps it runs.

### Worked examples

**Example 1: the conjunction smell**

Bad:
```js
function getPostCodeAndValidatePhoneNumber(user) {
    const postCode = user.address.postCode.trim().toUpperCase();

    if (!/^\+?[\d\s-]{7,}$/.test(user.phone)) {
        throw new Error('invalid phone');
    }

    return postCode;
}
```

The `And` is the tell. Split, then name the orchestrator after the *intent*:

Good:
```js
/** Used to normalize a UK post code to canonical form. */
function normalizePostCode(postCode) {
    return postCode.trim().toUpperCase();
}

/** Used to check that a phone number matches the expected shape. */
function isValidPhoneNumber(phone) {
    return /^\+?[\d\s-]{7,}$/.test(phone);
}

/** Used to extract a delivery address from a user, rejecting bad phones. */
function getDeliveryAddress(user) {
    if (!isValidPhoneNumber(user.phone)) {
        throw new Error('invalid phone');
    }

    return {
        postCode: normalizePostCode(user.address.postCode),
        phone: user.phone
    };
}
```

`getDeliveryAddress` is the orchestrator. Its name describes *what the combination is* — extracting a delivery address — not *what it does step by step* (`validateAndNormalize...`).

**Example 2: hidden side effects**

Bad:
```js
function getUserAndUpdateLastSeen(userId) {
    const user = db.users.findById(userId);
    db.users.update(userId, { lastSeenAt: Date.now() });
    return user;
}
```

`get` shouldn't write. Split:

Good:
```js
/** Used to load a user by id. */
function getUser(userId) {
    return db.users.findById(userId);
}

/** Used to mark a user as recently active. */
function touchLastSeen(userId) {
    db.users.update(userId, { lastSeenAt: Date.now() });
}
```

If a caller does both routinely, the orchestrator names *the combination*:

```js
/** Used to load a user and record that they were just active. */
function getActiveUser(userId) {
    const user = getUser(userId);

    touchLastSeen(userId);

    return user;
}
```

Not `getUserAndTouchLastSeen` — `getActiveUser`. The combination has a meaning; the name should name the meaning.

**Example 3: the verb-phrase ladder**

Bad:
```js
function fetchParseAndStoreInvoice(url) {
    const raw = http.get(url);
    const invoice = JSON.parse(raw.body);
    db.invoices.insert(invoice);
    return invoice;
}
```

Three verbs, three responsibilities. Split, then name the orchestrator after what the whole thing *is*:

Good:
```js
/** Used to fetch the raw invoice payload from a remote URL. */
function fetchInvoiceRaw(url) {
    return http.get(url).body;
}

/** Used to parse a raw invoice payload into a structured record. */
function parseInvoice(raw) {
    return JSON.parse(raw);
}

/** Used to persist an invoice record to the database. */
function storeInvoice(invoice) {
    db.invoices.insert(invoice);
}

/** Used to run the full invoice ingest pipeline for one URL. */
function ingestInvoice(url) {
    const raw = fetchInvoiceRaw(url);
    const invoice = parseInvoice(raw);

    storeInvoice(invoice);

    return invoice;
}
```

`ingestInvoice` is the noun-form of the whole operation. The steps are no longer in the name because they no longer need to be — the name says *what the operation is*, and the body says *how*.

**Example 4: the qualifier creep**

Bad:
```js
function getUserByEmailIncludingDeletedAndWithProfile(email) {
    // ...
}
```

The qualifiers are *options*, not separate responsibilities. Move them to a parameter:

Good:
```js
/** Used to look up a user by email, optionally including soft-deleted and profile. */
function findUserByEmail(email, { includeDeleted = false, includeProfile = false } = {}) {
    // ...
}
```

Splitting is for different responsibilities; parameters are for different modes of the same responsibility. The conjunction test only fires on the former.

### Length-as-diagnostic, not length-as-limit

There is no hard character limit on function names. `serializeUserForPublicApiResponse` is long but honest and singular — leave it alone.

The signal is the *conjunction*, not the length. Long names without conjunctions are usually fine; short names with hidden conjunctions (`save`, `process`, `handle`) are usually doing too much.

## Naming variables

Names are the most important comments in the code. A well-named variable doesn't need a comment; a badly-named one can't be saved by one.

### No single-letter names

Single-letter variables are forbidden except in two narrow cases:
- Loop indices in classical `for` loops: `for (let i = 0; i < n.length; i++)`.
- Mathematical formulas where the letters mirror the math being implemented (`const dx = x2 - x1`).

Everywhere else — including callback parameters, destructured fields, catch bindings, and reduce accumulators — use a name that describes the value.

### Callback parameters

The parameter to `.map`, `.filter`, `.find`, `.forEach`, `.reduce`, `.some`, `.every` names *one element of the collection*. Name it after what that element is, not after the collection or the method.

Good:
```js
const trimmed = inputs
    .map(String)
    .map((value) => value.trim())
    .filter(Boolean);

const activeUsers = users.filter((user) => user.isActive);

const total = lineItems.reduce((sum, item) => sum + item.price, 0);
```

Bad:
```js
const trimmed = inputs.map(String).map((s) => s.trim()).filter(Boolean);
const activeUsers = users.filter((u) => u.isActive);
const total = lineItems.reduce((a, b) => a + b.price, 0);
```

If the collection name is plural, the element name is usually the singular form: `users` → `user`, `messages` → `message`, `lineItems` → `item` or `lineItem`. If there's no obvious singular, name it after the role: `inputs.map((rawValue) => ...)`.

### Catch bindings

The error parameter in `catch` is `error`, never `e` or `err`. The two extra characters disambiguate it from event objects, expression placeholders, and a dozen other `e`s.

```js
try {
    return parse(input);
} catch (error) {
    logger.warn({ error }, 'parse failed');
    return null;
}
```

Exception: if the catch is empty or one-line and the binding is unused, omit it entirely (`try { ... } catch { /* ignore */ }`).

### Reduce accumulators

The accumulator parameter names *what is being accumulated*, not `acc` or `a`.

Good:
```js
const totalCents = orders.reduce((sumCents, order) => sumCents + order.cents, 0);

const byStatus = users.reduce((groups, user) => {
    groups[user.status] ??= [];
    groups[user.status].push(user);
    return groups;
}, {});
```

Bad:
```js
const total = orders.reduce((a, b) => a + b.cents, 0);
const byStatus = users.reduce((acc, u) => { ... }, {});
```

### Event handlers

Event objects are `event`, not `e` or `evt`. If you only need one property, destructure it inline.

```js
button.addEventListener('click', (event) => {
    event.preventDefault();
    submit();
});

input.addEventListener('input', ({ target }) => {
    setValue(target.value);
});
```

### Destructuring renames

When destructuring would produce a too-short or ambiguous name, rename it. `{ id }` is fine when the surrounding context makes "id of what" obvious; `{ id: userId }` is required when it's not.

```js
const { id: userId, name: userName } = user;
const { id: postId } = post;

createComment({ userId, postId, body });
```

### Booleans

Booleans read as questions: `isActive`, `hasPermission`, `canEdit`, `shouldRetry`, `wasModified`. Never bare nouns (`active`, `permission`) — those read as values, not predicates.

### Avoid generic role names

Names like `data`, `info`, `item`, `value`, `result`, `temp`, `obj`, `thing` are placeholders, not names. They're acceptable only when the variable genuinely is generic (a JSON blob whose shape isn't yet known, the return slot of a builder being assembled). When you find yourself reaching for `data`, ask what kind of data — `userPayload`, `parsedResponse`, `cachedRow` — and use that.

`result` is the most common offender. Prefer the specific noun: `validatedUser`, `normalizedPath`, `filteredRows`.

### Length follows scope

Short names are fine for short scopes; long names are required for long ones.
- A two-line arrow callback: `user` is enough.
- A 30-line function: `currentUser` or `requestingUser` if there are multiple users in play.
- A module-level binding: full descriptive name (`defaultRequestingUser`).

Never abbreviate to save typing. `req`, `res`, `msg`, `cfg`, `ctx`, `db`, `mgr` are all worse than their full forms. The only abbreviations allowed are ones that are *more* readable than the expansion: `id`, `url`, `http`, `json`, `html`, `css`, `api`, `uuid`, `i18n`.

### Worked example

Before:
```js
[...new Set(input.map(String).map((s) => s.trim()).filter(Boolean))]
```

After:
```js
const uniqueTrimmed = [...new Set(
    input
        .map(String)
        .map((value) => value.trim())
        .filter(Boolean)
)];
```

Two changes did the work: `s` became `value` (now you know what's flowing through the chain), and the whole expression got a name (`uniqueTrimmed`) instead of being an anonymous spread. The chain breaking across lines makes the transformation legible step by step.

## File naming

File naming is two decisions, not one. The directory says *what kind of thing the file is*; the name says *which specific thing of that kind*. Get the directory right first, then name the file within it.

### Directory tells you the category

Place files by role, not by name prefix. The location is the prefix.

- `lib/` — reusable libraries. Stable APIs intended to be imported from anywhere in the project.
- `utils/` — small, pure helpers. One concept per file; no external state, no I/O.
- `services/` — modules that talk to the outside world (HTTP, database, filesystem, third-party APIs).
- `routes/` or `handlers/` — request handlers, one per route or endpoint.
- `scripts/` — runnable entrypoints invoked via `npm run` or directly. Each file is a program, not a library.
- `bin/` — installed CLI entrypoints (the things listed under `"bin"` in `package.json`).
- `config/` — configuration loading and defaults.
- `types/` — shared type definitions (TypeScript) or JSDoc typedef files.

If a file would fit in two directories, it usually wants to be split.

### File names

Within a directory, name the file after *what it is*, not *what category it is*. The category is the directory.

- `kebab-case.js` — lowercase, words separated with `-`. Never `camelCase`, `snake_case`, or `PascalCase` for filenames.
- Start with a noun for libraries, services, and utilities: `price-cache.js`, `user-repository.js`, `phone-validator.js`.
- Start with a verb for scripts: `build-cache.js`, `migrate-prices.js`, `seed-database.js`. Running the file *does* the verb.
- One concept per file. If the name needs `and`, split the file the same way you'd split a function.
- Singular for the thing the file *is*: `user-repository.js`, not `users-repository.js`. Plural only when the file's subject is genuinely a collection (`countries.js` as a static list).

### When the directory already says it, the name doesn't

Bad:

```
lib/lib-price-cache.js
utils/util-format-date.js
services/service-stripe-client.js
scripts/script-build-cache.js
```

Good:

```
lib/price-cache.js
utils/format-date.js
services/stripe-client.js
scripts/build-cache.js
```

The directory is the prefix. Repeating it in the name is noise.

### When there is no directory to lean on

At the top of a directory where everything is the same category but the *purpose* differs — most often `scripts/` and `bin/` — the verb at the start of the name does the disambiguation work:

```
scripts/build-cache.js
scripts/migrate-prices.js
scripts/check-config.js
scripts/seed-database.js
scripts/clean-temp.js
```

You read the directory listing as a menu of commands. That's the point.

### The length-as-diagnostic rule, applied to files

Same rule as function names: if the honest filename uses `and`, or runs past about four hyphenated words, the file is doing too much. Split it.

Bad:

```
lib/fetch-and-cache-app-details.js
```

Good:

```
lib/app-details-fetcher.js
lib/app-details-cache.js
```

If a third file orchestrates the two:

```
lib/app-details.js          // exports the public API: getAppDetails()
lib/app-details-fetcher.js  // internal: network access
lib/app-details-cache.js    // internal: disk persistence
```

The orchestrator gets the unqualified name; the helpers carry the qualifier that says what slice of the work they own.

### When a file matches the thing it exports

If a file exports exactly one thing as its default or named export, the filename should match the export name in kebab-case:

```js
// lib/price-cache.js
export function priceCache() { ... }

// utils/format-date.js
export function formatDate(date) { ... }

// services/stripe-client.js
export class StripeClient { ... }
```

This is the most useful filename-to-symbol relationship: grep for the symbol, find the file instantly; open the file, know what's inside before reading.

### Index files

Use `index.js` only as a *barrel* — a file that re-exports the public surface of a directory. Never as the implementation. If a directory's public API is `getAppDetails`, `clearCache`, and `prefetch`, then:

```
lib/app-details/
    index.js              // re-exports getAppDetails, clearCache, prefetch
    get-app-details.js
    clear-cache.js
    prefetch.js
    fetcher.js            // internal
    cache.js              // internal
```

Importers write `import { getAppDetails } from 'lib/app-details'` and never touch the internals. The barrel is a deliberate published surface, not a dump of everything in the folder.

### Tests and types

- Test files mirror the file they test, with a `.test.js` (or `.spec.js`) suffix: `price-cache.js` → `price-cache.test.js`. Place them next to the source unless your project convention puts them under `test/`.
- Type-only files for a module: `price-cache.js` → `price-cache.types.ts`.
- Fixtures and mocks for a module: `price-cache.fixtures.js`, `price-cache.mock.js`.

The suffix tells you the file's relationship to its sibling; the base name links them.

### Worked example

A small slice of a real-ish layout:

```
src/
    lib/
        app-details.js              // public API
        app-details-fetcher.js      // internal: remote fetch
        app-details-cache.js        // internal: disk cache
        price-cache.js
        rate-limiter.js
    services/
        itunes-client.js
        openai-client.js
    utils/
        format-date.js
        normalize-postcode.js
        is-valid-phone.js
    routes/
        get-app-details.js
        post-price-batch.js
    scripts/
        build-cache.js
        migrate-prices.js
        check-config.js
    config/
        defaults.js
```

Read top to bottom: every file's *category* is its directory, every file's *identity* is its name, and no name repeats the category.

## File watching and dev servers

When the development environment runs the code in a sandboxed or isolated context (a VM, a remote interpreter, a containerized runtime, a cloud workspace), the *source tree is shared with the runtime via a mount or sync* rather than read directly from the host. The dev process inside the runtime watches the shared tree for changes and reloads when files change.

This shapes the choice of watcher.

### Prefer the framework's built-in watcher when one exists

Most modern Node frameworks ship a dev server with watching, hot module replacement, and route invalidation already wired up:

- Fastify: `fastify start -w`
- Next.js: `next dev`
- Vite (React, Vue, Svelte): `vite dev`
- Nest: `nest start --watch`
- Remix: `remix dev`
- SvelteKit: `vite dev` (Vite under the hood)
- Astro: `astro dev`

Use the framework's `dev` command and do not wrap it in anything else. Framework watchers understand the framework's module graph and reload only what is needed; a wrapping `node --watch` or `nodemon` causes double restarts and fights the incremental reload.

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "start": "node dist/server.js"
  }
}
```

### When no framework watcher exists, use `node --watch`

For plain Node apps without a framework dev server, use `node --watch` (built into Node 18.11+) or `tsx watch` (for TypeScript). Both are native or near-native, with no extra dependency:

```json
{
  "scripts": {
    "dev": "node --watch src/index.js",
    "start": "node dist/index.js"
  }
}
```

Do not add `nodemon` to new projects. `node --watch` does what `nodemon` did, natively. Existing projects with `nodemon` do not need to switch.

### Polling for sandboxed filesystems

In sandboxed runtimes, the watcher may not receive native filesystem events through the mount layer (inotify on Linux, FSEvents on macOS). If file changes are not picked up, enable polling:

```bash
NODE_OPTIONS=--watch=polling npm run dev
```

For framework dev servers, each framework has its own polling flag — `vite dev --force` and config options like `server.watch.usePolling: true` for Vite, `webpack.config.js`'s `watchOptions.poll` for webpack-based stacks. Polling is slightly slower than native events but reliable across mount types.

### Files outside the watched directory

Most watchers track only the files reachable from the entry point. Top-level config files (`package.json`, `tsconfig.json`, `*.config.js`) often fall outside that reach and may require an explicit watch flag or a manual restart when they change. Framework watchers typically handle their own config files; `node --watch` does not.

### Editor "safe write" quirks

Some editors write a temp file and atomically rename it over the original. Watchers that track inode identity rather than path may miss this. If saves are not triggering reloads, check the editor's "atomic writes" or "safe write" setting and disable it for the project.
