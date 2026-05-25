Rules

Declaration comments

Use /** ... */ for every top-level variable, constant, regex, and function — even one-liners. One comment per declaration, one declaration per comment.

Comment voice

- Sentence fragments, not full sentences.
- Start with "Used to...", "Used as...", "The...", or an imperative verb ("Load...", "Add...").
- Describe the role of the binding, not its type or mechanics. The code shows how; the comment shows what for.
- Impersonal and present-tense. No "we", no "I", no "this will...".

Do not narrate obvious code

No // loop over users above a for loop. Comment the non-obvious: workarounds, spec citations, bitmask meanings, why a defensive idiom exists. Include a URL when citing a bug or spec.

TODO and NOTE markers

- TODO: and NOTE: are encouraged for known issues and non-obvious behavior.
- Be honest about rough edges.
- TODO: describes work still to be done — never annotations about old work that was removed.

Document the present, not the past

Comments describe the current state of the code, not the history of how it got there.

Violations include:

- "Previously took..." notes in JSDoc after a signature change.
- "We considered X but went with Y" journey notes.
- // Removed the user-tier check that used to live here style comments.
- JSDoc parameters or return descriptions that no longer match the current signature.

Exception — migration notes. Migration notes are legitimate only when they describe an action the reader still has to take. Test: does this note describe something the reader must
still do? If yes, it stays. If it's purely "for context on how we got here," it goes.

/**
* Used to load a user by id.
*
* NOTE: callers passing legacy numeric ids must convert to string ids first.
*/
function getUser(userId) { /* ... */ }

Control flow block comments

Comment the intent of a block, not the mechanics. If a reader who knows JavaScript can see what the block does, don't restate it.

Blocks that usually need a comment:

- Guarding against a non-obvious edge case.
- Implementing a spec or RFC requirement — cite the section.
- Working around a runtime/browser/dependency bug — link the issue.
- A loop that exits early for a reason that isn't visible from the condition.
- A catch that swallows the error on purpose.
- A switch with intentional fallthrough.

Blocks that usually do NOT need a comment:

- A loop doing the obvious thing its body describes.
- An if whose condition reads as a sentence (if (user.isActive)).
- A try/catch that simply reports and rethrows.
- Any block whose purpose is already named by the enclosing function.

Placement

- Comment goes on the line directly above the keyword (if, for, while, try, switch), not inside the block.
- Use // for these, not /** ... */. JSDoc is for declarations.
- One short sentence fragment, same voice as declaration comments.
- Blank line above the comment, no blank line between the comment and the block it describes.

Special cases

- Empty catch blocks always get a comment. Even one word — // ignore or // best-effort — signals intent. A bare catch (error) {} looks like a bug.
- Intentional fallthrough in switch always gets a // fallthrough comment on the line above the next case. Linters look for this token.
- TODO/NOTE markers attach to the block they qualify, on the line directly above.
- Long conditions: prefer extracting a named boolean (const canEdit = ...) over commenting the condition. The named variable is the comment.

INI files

The same rules apply to config.ini and config.ini.template. INI files use ; as the comment marker; everything else (voice, role-describing fragments, TODO/NOTE markers) is identical.

---
Worked examples

Good — declaration comment

/** Used to validate an incoming order before insert. */
function validateOrder(order) {
    // ...
}

Bad — historical commentary

/**
* Used to validate an incoming order before insert.
* Note: previously also validated the deprecated `userId` field, which has been
* removed in favor of `customerId`.
*/
function validateOrder(order) {
    // Removed the user-tier check that used to live here when we had tiered pricing.
    // ...
}

Good — control flow comments on the why

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

Bad — narrates the mechanics

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

Good — combined example

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
