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

This is the leading-operator rule, and it covers property, method, and optional chains (`.`, `?.` start the line). Logical operators (`&&`, `||`) and ternaries break differently — they trail
the line. See Breaking long expressions below.

Call arguments, parameters, and object literals

One uniform rule: a delimited list of more than one item breaks one-item-per-line; a single item stays on one line. "List" means call arguments, function parameters (positional or
destructured), and object-literal properties — wherever they appear, including object literals that are returned or assigned, not only those passed as arguments.

Expand one-item-per-line when any of these is true:

- Two or more items (arguments, parameters, or object properties).
- Any single item is complex: an await, a ternary, an object or array literal, a nested call, or an arrow function with a body.
- The list wouldn't fit on one line at 120 chars.

A single-item list stays on one line: `findUser(username)`, `(value) => value.trim()`, `{id}`, `conn.send(payload)`.

When expanding: the opening paren or brace stays on the line that introduces it, each item on its own line indented one level deeper, a trailing comma after the last item (comma-dangle
requires it), and the closing delimiter back at the statement's indent.

Good — call arguments:
const cached = await readCacheIfFresh(
    DETAILS_CACHE_DIR,
    appId,
    DETAILS_CACHE_TTL_MS,
);

await fetchPriceBatchBody(
    chunk,
    countryCode,
);

Good — function parameters:
const issueSession = (
    username,
    role,
) => {
    // ...
};

Good — object-literal argument and destructured parameter:
await handleMcpRequest({
    request,
    response,
    persona,
    chatApi,
});

const createExecutor = ({
    turnEvents,
    conversationId,
    signal,
    resolve,
    reject,
}) => {
    // ...
};

Good — a single item stays inline:
const body = await fetchPriceBatchBody(chunk);
const wrap = (value) => value;
queue.push({item});

Bad — two or more items crammed onto one line:
await handleMcpRequest({request, response, persona, chatApi});
const createExecutor = ({turnEvents, conversationId, signal, resolve, reject}) => {
const cached = await readCacheIfFresh(DETAILS_CACHE_DIR, appId, DETAILS_CACHE_TTL_MS);
const issueSession = (username, role) => {

When a call regularly takes four or more positional arguments, suggest an options object — unless the arguments have a strong natural order (coordinates, source/destination,
key/value).

Breaking long expressions

When an expression is too long for one line at 120 chars, where the line breaks depends on the operator:

- Property / method / optional chains break with the operator leading the next line (`.`, `?.`), indented one level under the receiver. This is the Method chaining rule above.
- Logical operators (`&&`, `||`) break with the operator trailing the line. Each operand starts a new line at the same indent. Never lead a line with `&&` or `||`.
- A ternary that doesn't fit breaks with `?` trailing the condition line and `:` trailing the true-branch line; the true and false branches each sit on their own line, indented one level
  under the start. Short ternaries stay on one line.

Break after `=`. When a declaration's right-hand side is itself a multi-line expression — a multi-line boolean, a broken ternary, a regex literal, a string split across source lines — put
the `=` at the end of the declaration line and place the whole RHS on the following line(s), indented one level. Don't leave a lone operand or operator dangling on the declaration line.

Good:
const realCompletionTokens =
    Number.isFinite(usage?.completion_tokens) ? usage.completion_tokens : null;

const hasDeps =
    (pkg.dependencies && Object.keys(pkg.dependencies).length > 0) ||
    (pkg.optionalDependencies && Object.keys(pkg.optionalDependencies).length > 0);

const conversationId = typeof body?.conversation_id === 'string' ?
    body.conversation_id.trim() :
    '';

const host =
    request.headers['x-forwarded-host']
        ?.split(',')[0]
        ?.trim() ||
    request.headers.host ||
    'localhost';

Bad — leading logical operator:
const hasDeps = (pkg.dependencies && Object.keys(pkg.dependencies).length > 0)
    || (pkg.optionalDependencies && Object.keys(pkg.optionalDependencies).length > 0);

Bad — RHS dangling on the declaration line:
const host = request.headers['x-forwarded-host']
    ?.split(',')[0]
    ?.trim() || request.headers.host || 'localhost';

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

    return {
        name,
        age,
        label: `${name} (${age})`,
    };
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
- Trailing comma after the last item of any multi-line list — object/array literals, call arguments, and parameter lists (comma-dangle: always-multiline).
- No spaces inside braces: `{a: 1}`, `{name}`, `({id}) => …` — never `{ a: 1 }` (object-curly-spacing: never).
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
        return {
            ok: false,
            reason: 'missing',
        };
    }

    if (!reEmail.test(user.email)) {
        return {
            ok: false,
            reason: 'bad-email',
        };
    }

    const name = user.name
        .trim()
        .toLowerCase();

    return {
        ok: true,
        user: {
            name,
            email: user.email,
        },
    };
}
