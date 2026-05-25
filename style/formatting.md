Rules

Method chaining

Break a chain across multiple lines once it has two or more calls. One call per line, dot starting the line, aligned under the receiver. Break short chains too if any single call has
more than one argument or a function argument. Single-call expressions stay on one line.

Good:
const result = users
    .filter(u => u.active)
    .map(u => u.name)
    .sort();

Bad:
const result = users.filter(u => u.active).map(u => u.name).sort();

Call expressions

Expand to one-argument-per-line when any of these is true:

- Three or more arguments.
- Any single argument is complex: an await, a ternary, an object or array literal, a nested call, or an arrow function with a body.
- The call wouldn't fit on one line at 120 chars.

Otherwise keep the call on one line. When expanding: opening paren on the call line, each argument on its own line indented one level deeper, trailing comma after the last argument,
closing paren back at the statement's indent.

Good:
const body = await fetchPriceBatchBody(chunk, countryCode);

const cached = await readCacheIfFresh(
    DETAILS_CACHE_DIR,
    appId,
    DETAILS_CACHE_TTL_MS
);  

Bad — expansion for a trivial call:
const body = await fetchPriceBatchBody(
    chunk, 
    countryCode
);  

When a call regularly takes four or more positional arguments, suggest an options object — unless the arguments have a strong natural order (coordinates, source/destination,
key/value).

String interpolation

Default to template literals when a string is assembled from more than one piece. + between strings is a violation.

- Replace 'X ' + value + ' Y' with `X ${value} Y`.
- Whole-string literals with no interpolation use single quotes. Backticks without interpolation or a newline are a false signal.
- Multi-line strings use a single template literal with real newlines, not + concatenation.
- Keep interpolations short — they are values, not expressions to compute. If logic is needed, extract a named variable first.

Good:
throw new Error(`llama.cpp ${response.status} ${response.statusText}`);
const url = `https://${host}/api/files/${id}`;

Bad:
throw new Error('llama.cpp ' + response.status + ' ' + response.statusText);
const url = 'https://' + host + '/api/files/' + id;
const reason = `bad-email`;  // backticks without interpolation

+ is permitted in two cases: arithmetic that happens to land in a string context (`${count + 1}`), and joining two known string variables (prefix + suffix).

Visual rhythm ("aired" code)

- Blank line between every logical group of statements. A group is one idea: a declaration block, a guard clause, a computation, a return.
- Blank line after every variable declaration block that has its own JSDoc.
- Blank line before and after every if, for, while, try, and switch block.
- Blank line between every method definition, no exceptions.
- Function body shape: guard clauses → blank → main work → blank → return.
- One statement per line — never pack multiple statements with semicolons.
- Functions over ~30 lines should be split.

Good:
function formatUser(user) {
    if (!user) {
        return null;
    }   
    
    const name = user.name.trim();
    const age = user.age || 0;

    return { name, age, label: `${name} (${age})` };
}

Bad:
function formatUser(user) {
    if (!user) return null;
    const name = user.name.trim(); const age = user.age || 0;
    return { name, age, label: `${name} (${age})` };
}

Formatting baseline

- Indent: 2 spaces.
- Single quotes for strings (double for JSX attributes).
- Semicolons always.
- Trailing commas in multi-line literals.
- One statement per line.
- Spaces around operators, after commas, after keywords (if (x) not if(x)).
- Max line length: 120 characters.

File organization

- Imports at the top, blank line after.
- Constants and regexes next, each with their own JSDoc.
- Internal helpers.
- Public API.
- Exports at the bottom.
- Banner comments separate major sections, with a brief description:

/*----------------------------------CONFIG MANAGEMENT--------------------------------------*/
/*----------------------------------INPUT VALIDATION---------------------------------------*/

Worked example

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
            name,
            email: user.email
        }
    };
}
