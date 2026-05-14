# Linting

Every project has ESLint configured. Projects with CSS also have Stylelint. The configurations are project-standard and committed to the repo as `eslint.config.mjs` and `stylelint.config.mjs`.

## ESLint configuration

The canonical `eslint.config.mjs`:

```js
import google from 'eslint-config-google';
import sonarjs from 'eslint-plugin-sonarjs';

delete google.rules['valid-jsdoc'];
delete google.rules['require-jsdoc'];

export default [
    google,
    sonarjs.configs.recommended,
    {
        files: ['**/*.js', '**/*.jsx'],
        languageOptions: {
            ecmaVersion: 'latest',
            sourceType: 'module',
            parserOptions: {
                ecmaFeatures: {
                    jsx: true,
                },
            },
        },
        rules: {
            'no-console': 'off',
            'no-else-return': 'warn',
            'max-len': ['error', {
                code: 120,
                ignoreUrls: true,
                ignoreStrings: true,
                ignoreTemplateLiterals: true,
            }],
            'indent': ['error', 2],
            'no-unused-vars': ['error', {
                caughtErrors: 'none',
                argsIgnorePattern: '^_',
            }],
            // Surface task markers as warnings rather than errors. Intentional
            // reminders, not defects; still visible in editor gutters and CI output.
            'sonarjs/todo-tag': 'warn',
        },
    },
    {
        ignores: ['dist/', 'utils/venv/', 'node_modules/'],
    },
];
```

The choices, briefly:

- **Google base config** provides the structural rules (spacing, brace style, naming conventions). The two JSDoc rules are removed because the project's documentation style is governed by `comments.md`, not by Google's JSDoc requirements.
- **SonarJS recommended** layers on complexity and bug-detection rules. Catches things like duplicate string literals, redundant conditions, and unreachable code that Google's config alone misses.
- **`max-len: 120`** is the project's line length. URLs, strings, and template literals are exempted because forcing line breaks inside them often hurts readability more than it helps.
- **`indent: 2`** because that's what Google config expects and most of the JS ecosystem has converged on.
- **`no-unused-vars`** with `caughtErrors: 'none'` because `naming.md` (Catch bindings) requires naming the error parameter, even when unused. `argsIgnorePattern: '^_'` allows `_unused` parameters for callback-shape compatibility.
- **`sonarjs/todo-tag` as warning** because TODO and NOTE markers are encouraged (see `comments.md`) — they should appear in CI output as reminders, not as build failures.

## Stylelint configuration

Projects with CSS files also include `stylelint.config.mjs`:

```js
import standard from 'stylelint-config-standard';

export default {
    rules: {
        ...standard.rules,
        'selector-max-compound-selectors': 3,
        'declaration-no-important': true,
        'no-duplicate-selectors': true,
        'block-no-empty': true,
        'selector-class-pattern': null,
    },
    ignoreFiles: ['dist/**/*.css', 'vendor/**/*.scss'],
};
```

The rules layered on top of `stylelint-config-standard`:

- **`selector-max-compound-selectors: 3`** caps selectors at three compound levels. `.card .header .title` is allowed; `.card .header .title .icon` is not. Forces flatter CSS that doesn't depend on deep DOM nesting.
- **`declaration-no-important: true`** bans `!important`. If a rule needs `!important`, the cascade is wrong; fix the cascade.
- **`no-duplicate-selectors: true`** catches accidental rule duplication that produces order-dependent overrides.
- **`block-no-empty: true`** catches empty rule blocks left over from refactors.
- **`selector-class-pattern: null`** disables the standard config's naming pattern check, allowing the project's own naming conventions without fighting the linter.

## Running the linter

The `lint` script (see `npm-scripts.md`) runs both linters and the audit in one command. Editors should be configured to run ESLint on save (and Stylelint on save for CSS) so problems surface immediately rather than at CI time.

## When to run the linter

Run `npm run lint` after every change set, before considering the work done.

A change set is a logical unit of work, not a single keystroke. Examples of change sets:

- Implementing a feature across several files.
- Refactoring a module.
- Fixing a bug and adding a regression test.
- Adding a new dependency and its integration code.

After completing the change set, run `npm run lint`. If it reports errors or warnings, **fix them before the change set is considered complete.** The lint script has `--fix` enabled, so safe fixes are applied automatically; what remains in the output is what needs human attention.

This is not optional and is not deferrable to CI. A change set with unresolved lint issues is incomplete work.

## What "fix the reported issues" means

For each issue the linter reports:

- **Auto-fixable issues** are already fixed by `--fix`. No action needed beyond reviewing the diff and committing.
- **Errors** must be resolved by changing the code, not by disabling the rule. Disabling a rule with an inline comment (`// eslint-disable-next-line ...`) is allowed only when the rule is genuinely wrong for the case at hand, and only with a comment explaining why on the same line or the line above.
- **Warnings** (including `TODO:` and `NOTE:` markers flagged by `sonarjs/todo-tag`) are not blocking but should be reviewed. If a warning represents real future work, leave it; if it represents a defect, fix it now.
- **`npm audit` findings** at high severity must be resolved before the change set ships. The fix is usually `npm update <package>` or a major version bump for the affected dependency. If no fix is available, document the exception inline with the dependency and revisit when an upgrade path exists.

## What "considered complete" means

A change set is complete when:

- The work itself is done (the feature works, the bug is fixed, the refactor is finished).
- `npm run lint` exits cleanly with no errors.
- Tests pass (`npm test`).
- The diff is reviewed and committed.

Skipping the lint step "because it's just a small change" is the failure mode this rule exists to prevent. Small changes accumulate lint debt fast; running the linter at the end of each change set keeps the codebase clean continuously rather than in periodic cleanup sprints.

## Editor-on-save vs. end-of-changeset linting

Editor-on-save linting surfaces issues immediately as they are introduced. End-of-changeset linting catches what the editor missed (rules that only run on the whole project, audit findings, cross-file issues like duplicate exports). Both are needed; neither replaces the other.

The on-save check tightens the inner loop. The `npm run lint` check is the gate before the work is considered done.
