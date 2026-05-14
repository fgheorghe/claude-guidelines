# REST API framework

For projects that are pure REST APIs (no server-rendered UI, no static site generation, no template engine), use **Fastify**. It is the project standard for API-only services.

This section applies only to API-only projects. Projects that also serve UI components have different framework needs not covered here.

## Why Fastify

Fastify gives you four things in one package, with no assembly:

- **A minimal, REST-focused framework.** No MVC scaffolding, no opinionated app structure, no GraphQL or RPC baggage. Register routes, start the server.
- **First-class OpenAPI/Swagger generation** via `@fastify/swagger` and `@fastify/swagger-ui`, both maintained by the Fastify team. The JSON Schema you write for request/response validation *is* the OpenAPI spec — one source of truth, no separate docs file, no comment-parsing.
- **A built-in dev server with file watching** via `fastify-cli`'s `-w` flag. No `nodemon`, no `node --watch` wrapper. The framework's own watcher.
- **Schema-based validation** for free, derived from the same schemas that produce the docs.

This means a Fastify project naturally satisfies several rules from elsewhere in this guide: the dev script is a thin one-liner, file watching uses the framework's built-in mechanism, and there is no separate documentation step that can drift from the code.

## The starting shape

`package.json`:

```json
{
  "scripts": {
    "dev": "fastify start -w src/app.js",
    "start": "fastify start src/app.js",
    "lint": "eslint . --fix && npm audit --audit-level=high"
  },
  "dependencies": {
    "fastify": "^5",
    "@fastify/swagger": "^9",
    "@fastify/swagger-ui": "^5",
    "ini": "^4",
    "winston": "^3"
  }
}
```

`src/app.js` is the Fastify plugin file that `fastify-cli` loads. It registers plugins and routes; it does not call `listen()` itself (the CLI handles that):

```js
import swagger from '@fastify/swagger';
import swaggerUI from '@fastify/swagger-ui';

import { config } from './lib/config.js';
import { logger } from './lib/logger.js';
import { registerRoutes } from './routes/index.js';

/** Used as the Fastify plugin entrypoint, loaded by fastify-cli. */
export default async function app(fastify) {
    fastify.log = logger;

    await fastify.register(swagger, {
        openapi: {
            info: {
                title: config.api.title,
                description: config.api.description,
                version: config.api.version,
            },
        },
    });

    await fastify.register(swaggerUI, {
        routePrefix: '/docs',
    });

    await registerRoutes(fastify);
}
```

The application loads its configuration from `config.ini` at startup, as established in `config.md`. The API title, description, and version come from there:

```ini
[api]
title = Steam Proxy API
description = Internal API for Steam metadata and price proxying.
version = 1.0.0
```

## Routes carry their own schemas

Every route declares a schema describing its parameters, body, query, and response shapes. The schema is the validation *and* the OpenAPI documentation; these are not two separate concerns.

```js
// src/routes/users.js

/** Used to register the users routes on the Fastify instance. */
export async function registerUserRoutes(fastify) {
    fastify.get('/users/:id', {
        schema: {
            description: 'Fetch a single user by id.',
            tags: ['users'],
            params: {
                type: 'object',
                properties: {
                    id: { type: 'string', description: 'The user id.' },
                },
                required: ['id'],
            },
            response: {
                200: {
                    description: 'The user record.',
                    type: 'object',
                    properties: {
                        id: { type: 'string' },
                        name: { type: 'string' },
                        email: { type: 'string', format: 'email' },
                    },
                    required: ['id', 'name', 'email'],
                },
                404: {
                    description: 'No user with the given id exists.',
                    type: 'object',
                    properties: {
                        error: { type: 'string' },
                    },
                },
            },
        },
    }, async (request, reply) => {
        const user = await findUser(request.params.id);

        if (!user) {
            return reply.code(404).send({ error: 'not-found' });
        }

        return user;
    });
}
```

The single route definition above produces: request validation (rejecting malformed `:id` values), response validation (catching the bug where the handler returns the wrong shape), an OpenAPI entry under the `users` tag at `/docs`, and a machine-readable spec at `/docs/json`. No annotations, no JSDoc parsing, no separate spec file.

## Swagger docs are part of the contract

The OpenAPI spec is published by every API project, served at `/docs` (the Swagger UI) and `/docs/json` (the raw spec). Consumers of the API rely on it. The rule:

**The Swagger docs must always exactly match what the API actually does.**

In practice, because Fastify generates the spec from the route schemas at startup, this rule reduces to:

- Every route has a complete schema. Not just the parts the validator needs — every field, every response code, every error shape.
- Every field has a `description`. Schemas without descriptions produce docs that say "string" for everything; the docs are useless. Descriptions are sentence fragments in the same voice as the JS comment style: terse, role-describing, present tense.
- Every route has a `description` and at least one `tag`. Tags group routes in the Swagger UI; without them, the UI is a flat list of paths.
- Every error response is documented. A route that can return 400, 404, and 500 declares all three response schemas, not just the happy path.
- When the implementation changes, the schema changes in the same commit. A route returning a new field gets the field in its `response` schema. A route rejecting a new input case gets the constraint in its `params` or `body` schema. The two are never out of step, because the schema *is* the source of truth that runtime validation enforces.

If the docs drift from reality, the validator will catch it — Fastify rejects responses that do not match their declared schemas in non-production environments. A 500 from the validator on a previously-working route is the signal that the schema was not updated alongside the implementation.

**The Swagger docs must be disabled in production.**

## Tags, descriptions, and the Swagger UI

Tags group related routes. Use kebab-case tags named after the resource or area:

```js
schema: {
    tags: ['users'],
    // ...
}
```

Common tag patterns: one tag per resource type (`users`, `orders`, `products`), or one tag per functional area for cross-cutting endpoints (`auth`, `admin`, `health`). A route can carry multiple tags if it genuinely belongs in two areas.

Descriptions at the route level explain *what the endpoint does* in one sentence, in the same voice as JSDoc:

```js
schema: {
    description: 'Fetch a single user by id.',
    // ...
}
```

Not "This endpoint fetches a user." Not "We fetch the user." Just the role: "Fetch a single user by id."

Field descriptions follow the same rules:

```js
properties: {
    id: { type: 'string', description: 'The user id.' },
    email: { type: 'string', format: 'email', description: 'The user contact email.' },
}
```

A field without a description is a defect; the docs page will show "string" with no further context.

## Validation as documentation

Use JSON Schema's full vocabulary where it helps the docs, not just where it helps the validator:

- `format: 'email'`, `format: 'date-time'`, `format: 'uuid'` — both validates and tells the docs reader what shape to expect.
- `enum: ['active', 'pending', 'archived']` — both rejects invalid values and lists the allowed ones in the docs.
- `minimum`, `maximum`, `minLength`, `maxLength` — both enforces bounds and shows them.
- `required: [...]` — both rejects missing fields and marks them required in the docs.
- `description` everywhere — not validation, but the docs reader's only context.

The principle: the schema should be detailed enough that a reader who has never seen the codebase can use the API from the Swagger UI alone, without reading the source.

## File layout

A typical Fastify project follows the file layout already established in `naming.md`:

```
src/
    app.js                      // the Fastify plugin entrypoint
    routes/
        index.js                // registers all route groups
        users.js                // user routes
        orders.js               // order routes
        health.js               // healthcheck
    lib/
        config.js               // loads and freezes config.ini
        logger.js               // winston instance
        db.js                   // database client
    services/
        steam-client.js         // third-party API client
        price-cache.js
    schemas/
        user.js                 // shared schemas, referenced from routes
        order.js
    plugins/
        auth.js                 // Fastify plugins for cross-cutting concerns
        rate-limit.js
config.ini                      // gitignored, real values
config.ini.template             // committed, with comments and defaults
package.json
eslint.config.mjs
```

`routes/index.js` is the orchestrator that registers each route group:

```js
import { registerUserRoutes } from './users.js';
import { registerOrderRoutes } from './orders.js';
import { registerHealthRoutes } from './health.js';

/** Used to register every route group on the Fastify instance. */
export async function registerRoutes(fastify) {
    await registerUserRoutes(fastify);
    await registerOrderRoutes(fastify);
    await registerHealthRoutes(fastify);
}
```

Adding a new route group is one import line and one registration line, alphabetized.

## Shared schemas

When the same shape appears in multiple routes (a `User` object returned by several endpoints, an `Error` envelope used everywhere), define it once in `schemas/` and reference it:

```js
// src/schemas/user.js

/** The shape of a User object as returned by user-facing endpoints. */
export const userSchema = {
    $id: 'User',
    type: 'object',
    properties: {
        id: { type: 'string', description: 'The user id.' },
        name: { type: 'string', description: 'The user display name.' },
        email: { type: 'string', format: 'email', description: 'The user contact email.' },
    },
    required: ['id', 'name', 'email'],
};
```

Register the schema once at startup, reference by `$ref` from routes:

```js
// In app.js, before route registration
fastify.addSchema(userSchema);

// In a route
response: {
    200: { $ref: 'User#' },
}
```

The Swagger UI renders the `$ref` correctly, showing the reused shape under the "Schemas" section of the docs. Repeated inline schemas duplicate the docs and drift independently; shared schemas stay in sync by construction.

## Logging

Use `winston` as the Fastify logger, configured once in `lib/logger.js` and assigned to `fastify.log` at startup. Per `dependencies.md`, `winston` is the project standard; this overrides Fastify's default `pino`.

```js
// src/lib/logger.js
import winston from 'winston';
import { config } from './config.js';

/** The application's winston logger, configured once at startup. */
export const logger = winston.createLogger({
    level: config.logging.level,
    format: winston.format.json(),
    transports: [new winston.transports.Console()],
});
```

```js
// In app.js
fastify.log = logger;
```

Request and response logging falls out of Fastify automatically once the logger is attached — every request is logged with method, path, status code, and duration without further configuration.

## What "API-only" means for this guidance

This section applies when the project's responsibility is serving JSON over HTTP to programmatic clients (other services, mobile apps, SPAs hosted elsewhere). It does *not* apply when the project also serves HTML, renders templates, hosts a static site, or otherwise has UI rendering responsibilities. Projects with UI components have different framework needs that are not covered here.

The test: does this project's output, in production, consist entirely of JSON responses to HTTP requests? If yes, Fastify. If the project also produces HTML, CSS, or rendered pages, a different framework choice applies and is out of scope for this section.
