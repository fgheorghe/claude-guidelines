# Comments

- Use `/** ... */` for every top-level variable, constant, regex, and function — even one-liners.
- Comments are sentence fragments, not full sentences. Start with "Used to...", "Used as...", "The...", or an imperative verb ("Load...", "Add...").
- Describe the *role* of the binding, not its type or mechanics. The code shows how; the comment shows what for.
- Do not narrate obvious code. No `// loop over users` above a `for` loop.
- Comment the non-obvious: workarounds, spec citations, bitmask meanings, why a defensive idiom exists. Include a URL when citing a bug or spec.
- One comment per declaration, one declaration per comment.
- Voice is impersonal and present-tense. No "we", no "I", no "this will...".
- `TODO:` and `NOTE:` markers are encouraged for known issues and non-obvious behavior. Be honest about rough edges.

The same rules apply to comments in `config.ini` and `config.ini.template`. INI files use `;` as the comment marker; everything else (voice, role-describing fragments, TODO/NOTE markers) is identical. See `config.md` for examples.

## Document the present, not the past

Comments and docs describe the current state of the code, not the history of how it got there. When code is removed or changed, the documentation about the old version is removed in the same commit — not annotated, not preserved, not turned into a comment explaining what used to be there.

The rule applies identically to JSDoc, inline comments, Dockerfile comments, compose-file comments, `.env.template` comments, and `README.md`.

### What "document the present" means in practice

When a function changes signature, the JSDoc reflects the new signature. The old parameters do not get a "previously took..." note.

When a service is removed from `docker-compose.yml`, every reference to it disappears in the same commit: the service block, any `depends_on` entries, any `.env` variables only it used, any README mention. Nothing is left behind explaining "this used to also run Redis."

When the cron container moves from `crond` to a Node scheduler, the compose file describes the Node scheduler. There is no comment saying "this used to be a crond-based container." Git remembers the previous version; the current file does not.

When a feature flag is removed, the `config.ini.template` loses the entry. The README's "Operating notes" section, if it mentioned the flag, loses that bullet.

### What this looks like

Good — current state only:

```js
/** Used to validate an incoming order before insert. */
function validateOrder(order) {
    if (!order.customerId) {
        return { ok: false, reason: 'missing-customer' };
    }

    if (order.lineItems.length === 0) {
        return { ok: false, reason: 'empty-order' };
    }

    return { ok: true };
}
```

Bad — historical commentary:

```js
/**
 * Used to validate an incoming order before insert.
 * Note: previously also validated the deprecated `userId` field, which has been
 * removed in favor of `customerId`. Old callers may still pass it; the field
 * is now ignored.
 */
function validateOrder(order) {
    // Removed the user-tier check that used to live here when we had tiered pricing.
    // ...
}
```

The bad version preserves the journey. The good version describes the destination. A reader of the bad version has to mentally subtract the history before they can understand what the function does now; a reader of the good version sees the function as it is.

### When migration notes are legitimate

Migration notes are legitimate **only when they describe an action the reader still has to take**:

```js
/**
 * Used to load a user by id.
 *
 * NOTE: callers passing legacy numeric ids must convert to string ids first.
 * The migration script in scripts/migrate-user-ids.js handles existing data.
 */
function getUser(userId) {
    // ...
}
```

This is current-state documentation: the function takes string ids now, and the note tells a reader what they need to do if they have legacy data. It is not "this function used to take numeric ids." It is "this function takes string ids; if your data is from before the migration, here is how to bring it forward."

The test: does this note describe something the reader must still do? If yes, it stays. If it's purely "for context on how we got here," it goes.

### Removing documentation when removing code

Documentation removal is part of the same change set as code removal. If a refactor takes out a function, the same commit removes the JSDoc, removes any callers, removes any README mention of the feature it powered, and removes any config entries that only existed for it.

This is the most common failure mode in practice: code gets deleted cleanly, but the documentation gets left behind because nobody grepped for references. Reviewers should ask, for any deletion: "is the documentation about this thing also gone?" If references remain, the change set is incomplete.

### Applies to dev decisions and "we tried X first" notes

A common temptation is to comment on the path not taken: "We considered using Redis for this but went with SQLite because..." or "TODO: was originally going to support Postgres directly, removed for now." These are journey notes. They do not belong in the code.

If the decision genuinely matters for a future reader — they might be tempted to reintroduce the rejected approach — write it down once in an architecture-decision record (`docs/decisions/0001-storage.md` or similar), not scattered through the codebase. The code itself describes what is, not what was almost.

Same for `TODO:` markers that say "previously did X, may reintroduce." If reintroduction is real and planned, it's a `TODO:` about the new work to be done, not an annotation about the old work that was removed. If reintroduction is hypothetical, the TODO doesn't belong at all.

### README and operating notes especially

The README's "Operating notes" section is the most vulnerable to historical accumulation. Every refactor adds a new note; nothing ever removes the old ones. The result is a README full of caveats that no longer apply.

When refreshing or extending the README, audit the operating notes:

- For each existing note, ask: "is this still true?" If not, remove it.
- For each existing note, ask: "would a new reader benefit from knowing this?" If not, remove it.
- For each existing note, ask: "does the rule still apply?" If the underlying behavior has changed, update the note or remove it.

A 30-line "Operating notes" section that all currently applies is more useful than a 200-line one where half the notes describe behavior that was changed a year ago.

## Commenting control flow blocks

The rule: comment the *intent* of a block, not the *mechanics*. If a reader who knows JavaScript can see what the block does, don't restate it. If they can't see *why* the block exists or *what case it handles*, that's where a comment earns its place.

### Blocks that usually need a comment

- A block guarding against a non-obvious edge case (an off-by-one, an API quirk, a known-bad input).
- A block implementing a spec or RFC requirement — cite the section.
- A block working around a bug in a runtime, browser, or dependency — link the issue.
- A loop that exits early for a reason that isn't visible from the condition.
- A `catch` that swallows the error on purpose.
- A `switch` that intentionally falls through.

### Blocks that usually do NOT need a comment

- A loop that does the obvious thing its body describes.
- An `if` whose condition already reads as a sentence (`if (user.isActive)`).
- A `try/catch` that simply reports and rethrows.
- Any block whose purpose is already named by the enclosing function.

### Placement

- Comments for a block go on the line directly above the keyword (`if`, `for`, `while`, `try`, `switch`), not inside the block.
- Use `//` for these, not `/** ... */`. JSDoc is reserved for declarations.
- Keep them to one short sentence fragment, same voice as the declaration comments: terse, role-describing, present tense.
- Blank line above the comment, no blank line between the comment and the block it describes.

### Examples

Good — comments the *why*:
```js
// Skip soft-deleted rows; they reappear if the user undeletes.
for (const row of rows) {
    if (row.deletedAt) {
        continue;
    }
    process(row);
}

// Older Safari throws on cross-origin reads; fall back to a copy.
try {
    return window.parent.location.href;
} catch (error) {
    return document.referrer;
}

// RFC 2812 §3.2.2: channel keys are checked before invite status.
if (channel.key && channel.key !== providedKey) {
    return reject('bad-key');
}
```

Bad — narrates the mechanics:
```js
// Loop through the rows.
for (const row of rows) {
    // Check if deleted.
    if (row.deletedAt) {
        // Skip it.
        continue;
    }
    // Process the row.
    process(row);
}
```

### Special cases

- **Empty catch blocks always get a comment.** A bare `catch (error) {}` looks like a bug. Even one word — `// ignore` or `// best-effort` — signals intent.
- **Intentional fallthrough in `switch` always gets a `// fallthrough` comment** on the line above the next `case`. Linters look for this token specifically.
- **TODO/NOTE markers attach to the block they qualify**, on the line directly above. `// TODO: handle the multi-channel case` above the `if`, not buried inside it.
- **Long conditions can carry a comment instead of being broken across lines.** If `if (user.role === 'admin' && !user.suspended && feature.enabled)` is hard to read, prefer extracting a named boolean (`const canEdit = ...`) over commenting the condition. The named variable *is* the comment.

### Worked example

```js
/** Used to dispatch an incoming IRC command to its handler. */
function dispatch(command, args, client) {
    if (!client.authenticated && !PUBLIC_COMMANDS.has(command)) {
        return client.send(ERR_NOTREGISTERED);
    }

    const handler = handlers[command];

    if (!handler) {
        return client.send(ERR_UNKNOWNCOMMAND, command);
    }

    // Handlers may throw on malformed args; surface a protocol error instead of crashing.
    try {
        return handler(args, client);
    } catch (error) {
        logger.warn({ error, command }, 'handler threw');
        return client.send(ERR_NEEDMOREPARAMS, command);
    }
}
```

Notice what's *not* commented: the two guard `if`s. Their conditions read as English already (`!client.authenticated`, `!handler`), and the return value of the function (`client.send(ERR_...)`) tells you what each branch does. Adding `// check if authenticated` would be noise.

The `try/catch` *is* commented, because the intent — "don't let a buggy handler crash the server" — isn't visible from the code alone. The comment names the policy.
