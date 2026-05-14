# Personal coding conventions

These conventions apply to all projects on this machine. Project-level `CLAUDE.md` files may add additional rules but must not contradict these.

## Code style

The JavaScript style guide is split into topical files under `~/.claude/style/`. Most JS tasks need only a subset; pick by relevance.

**Foundational (apply to all JS work):**

- @~/.claude/style/comments.md — comment style, doc-the-present rule, control flow comments.
- @~/.claude/style/formatting.md — method chaining, call expressions, visual rhythm, formatting baseline, file organization.
- @~/.claude/style/naming.md — variables, functions, files, file watching.

**Tooling and project structure:**

- @~/.claude/style/linting.md — ESLint and Stylelint configurations, when to run the linter.
- @~/.claude/style/npm-scripts.md — script naming and grouping, the lint command.
- @~/.claude/style/config.md — `config.ini`, the no-env rule, application configuration.
- @~/.claude/style/dependencies.md — preferred modules, when to add a dependency.

**Framework-specific:**

- @~/.claude/style/api-framework.md — Fastify, Swagger, REST API conventions. Apply only when the project is a pure REST API with no UI.

## Container conventions

For Docker, compose, and container-related work: @~/.claude/docker.md.

## Documentation

For `README.md` rules: @~/.claude/style/readme.md.

## Reading order

When starting a session, the relevant files for the current task:

- All JS coding work: comments, formatting, naming, linting, npm-scripts, config, dependencies.
- API-only REST projects: add api-framework.
- Container or compose work: add docker.
- README work: add readme.

The split lets each session load only what is relevant to the work at hand.
