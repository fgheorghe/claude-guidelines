# Formatting

## Method chaining

- Break method chains across multiple lines once there are two or more calls.
- One call per line. The dot starts the line, aligned under the receiver.
- Even short chains get broken if any single call has more than one argument or a function argument.
- Single-call expressions stay on one line.

Good:
```js
const result = users
    .filter(u => u.active)
    .map(u => u.name)
    .sort();
```

Bad:
```js
const result = users.filter(u => u.active).map(u => u.name).sort();
```

## Call expressions

Break a function call across multiple lines when it earns the break, leave it compact when it doesn't. The threshold is concrete, not aesthetic.

### When to expand a call

Expand to one-argument-per-line when *any* of these is true:

- The call has three or more arguments.
- Any single argument is a complex expression: an `await`, a ternary, an object or array literal, a nested call, an arrow function with a body.
- The full call wouldn't fit on one line at the project's line width.

Otherwise, keep the call on one line.

### How to expand

- Opening paren stays on the call line.
- Each argument on its own line, indented one level deeper than the statement.
- Trailing comma after the last argument.
- Closing paren on its own line, back at the statement's indent level.

Good:
```js
const body = await fetchPriceBatchBody(chunk, countryCode);

const cached = await readCacheIfFresh(
    DETAILS_CACHE_DIR,
    appId,
    DETAILS_CACHE_TTL_MS
);

await persistPriceBatch(
    body,
    chunk,
    countryCode,
    priceOverviews
);

return response.json(
    await applyAiDescription(cached, appId, countryCode)
);
```

Note that `fetchPriceBatchBody` stays compact (two short arguments), `readCacheIfFresh` and `persistPriceBatch` expand (three and four arguments), and `response.json` expands at the outer level because its single argument is an `await` expression — but the inner `applyAiDescription` stays compact because *its* three arguments are all short identifiers and the break is already paying for itself at the outer level.

Bad — uniformly compact:
```js
const cached = await readCacheIfFresh(DETAILS_CACHE_DIR, appId, DETAILS_CACHE_TTL_MS);
await persistPriceBatch(body, chunk, countryCode, priceOverviews);
return response.json(await applyAiDescription(cached, appId, countryCode));
```

Each line is a left-to-right blob; the argument list and the verb compete for attention.

Bad — uniformly expanded:
```js
const body = await fetchPriceBatchBody(
    chunk,
    countryCode
);
```

Three lines for what was one. The expansion adds vertical weight without adding clarity — `chunk` and `countryCode` were already legible side by side.

### Threshold, not absolute rule

The point of the threshold is that *visual weight should match semantic weight*. A trivial two-argument call is a trivial line. A four-argument call with policy-bearing values is a paragraph. Uniform formatting flattens that signal; threshold formatting preserves it.

### Four-argument calls are a signal

When you find yourself expanding a call to four or more positional arguments regularly, consider whether the function should take an options object instead:

```js
await persistPriceBatch({
    body,
    chunk,
    countryCode,
    priceOverviews
});
```

The call site is now self-labeling, reordering is free, and adding a fifth field doesn't break existing callers. Switch to an options object once positional arguments cross three or four, *unless* the arguments have a strong natural order (coordinates, source/destination, key/value).

## Visual rhythm ("aired" code)

- Blank line between every logical group of statements. A "group" is one idea: a declaration block, a guard clause, a computation, a return.
- Blank line after every variable declaration block that has its own JSDoc.
- Blank line before and after every `if`, `for`, `while`, `try`, and `switch` block.
- Blank line between every method definition, with no exceptions.
- Inside function bodies: guard clauses at the top, blank line, then the main work, blank line, then the return.
- Do not pack multiple statements onto one line, even with semicolons.
- Prefer two short functions over one long one. If a function exceeds ~30 lines, look for a split.

Good:
```js
/** Used to format a user record for display. */
function formatUser(user) {
    if (!user) {
        return null;
    }

    const name = user.name.trim();
    const age = user.age || 0;

    return {
        name: name,
        age: age,
        label: `${name} (${age})`
    };
}
```

Bad:
```js
function formatUser(user) {
    if (!user) return null;
    const name = user.name.trim(); const age = user.age || 0;
    return { name, age, label: `${name} (${age})` };
}
```

## Formatting baseline

- Indent: 2 spaces (enforced by ESLint; see `linting.md`). Worked examples in this guide use 4 spaces for visual clarity in prose, but production code uses 2.
- Single quotes for strings, double for JSX attributes if applicable.
- Semicolons always.
- Trailing commas in multi-line literals.
- One statement per line.
- Spaces around operators, after commas, after keywords (`if (x)` not `if(x)`).
- Max line length: 120 characters (enforced by ESLint).

## File organization

- License/header comment at the top.
- Place constants at the top of their file, function, method, or block.
- Imports or environment detection next, with a blank line after.
- Constants and regexes, each with their own JSDoc.
- Internal helpers.
- Public API.
- Exports at the bottom.
- Use banner comments to separate major sections, and include a very brief description (ie.: CONFIG MANAGEMENT, INPUT VALIDATION, ADDRESS MANAGEMENT):

```js
/*----------------------------------CONFIG MANAGEMENT--------------------------------------*/
/*----------------------------------INPUT VALIDATION---------------------------------------*/
```

## Worked example combining all rules

```js
/** Used to match valid email addresses. */
const reEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

/** The maximum number of retries before giving up. */
const MAX_RETRIES = 3;

/*----------------------------------INPUT VALIDATION---------------------------------------*/

/** Used to validate a single user record before insert. */
function validateUser(user) {
    if (!user) {
        return { ok: false, reason: 'missing' };
    }

    if (!reEmail.test(user.email)) {
        return { ok: false, reason: 'bad-email' };
    }

    const name = user.name
        .trim()
        .toLowerCase();

    return {
        ok: true,
        user: {
            name: name,
            email: user.email
        }
    };
}
```
