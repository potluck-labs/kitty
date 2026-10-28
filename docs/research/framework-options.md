# Backend and bot framework options in Python and TypeScript

Research for [#4](https://github.com/potluck-labs/kitty/issues/4), part of the map [#2](https://github.com/potluck-labs/kitty/issues/2). Gathered on 2026-10-28.

This file lists options and trade-offs. It does not pick a framework. That decision belongs to a later ticket.

## Question

What are the main framework options for a backend service and a Telegram bot in Python and in TypeScript, and how do they compare for two junior engineers who want to learn senior-level architecture?

Candidates:

- Backend, Python: FastAPI, Litestar, Django.
- Backend, TypeScript: NestJS, Fastify.
- Bot, Python: aiogram, python-telegram-bot (PTB).
- Bot, TypeScript: grammY, Telegraf.

Criteria: maturity, maintenance activity, typing, testing, dependency injection (DI), async model, Postgres tooling, and how each holds up as the codebase grows.

## Context from the map

- ADR 0001 makes identity channel-agnostic. The bot is one Channel. Users, Groups, Accounts, and Invite codes live in the backend and never depend on Telegram features.
- So the bot is a thin adapter. It turns Telegram updates into calls to a Channel-neutral backend API and turns results into messages. It should hold no ledger state of its own, only short-lived conversation state (for example, "user is halfway through entering an Expense").
- The backend owns the domain model, Postgres, and the API. Most of the architecture learning happens there.
- ORM, query layer, and migration tooling are listed under "Not yet specified" on the map. This file only notes what each framework ships or pairs with, so that later ticket has the facts.

## Method

- Maintenance numbers come from the GitHub GraphQL API, queried on 2026-10-28. "Commits in last 12 months" counts commits on the default branch since 2025-10-28.
- Feature claims come from each project's own docs or README. Each claim links to its source in the "Sources" section.
- Telegram Bot API dates come from Telegram's own changelog.

## Maintenance snapshot

| Project | Stars | Latest release | Commits in last 12 months | Open issues | License |
|---|---|---|---|---|---|
| FastAPI | 104,236 | 0.162.0, 2026-10-23 | 1,784 | 1 | MIT |
| Litestar | 8,532 | v2.25.0, 2026-10-18 | 304 | 232 | MIT |
| Django | 91,531 | 6.1.2 (tags only, no GitHub releases) | 1,081 | 0 (uses Trac) | BSD-3-Clause |
| NestJS | 76,931 | v12.3.0, 2026-10-28 | 3,150 | 8 | MIT |
| Fastify | 37,312 | v5.12.7, 2026-10-13 (v6.0.0-alpha.5 on 2026-10-16) | 361 | 58 | MIT |
| aiogram | 5,910 | v3.31.0, 2026-08-25 | 80 | 36 | MIT |
| python-telegram-bot | 29,534 | v22.8, 2026-06-12 | 176 | 24 | LGPL-3 |
| grammY | 3,785 | v1.46.0, 2026-08-26 | 87 | 13 | MIT |
| Telegraf | 9,202 | v4.16.3, 2024-02-29 | 13 | 64 | MIT |

Telegram Bot API support, per each README, against the current Bot API 10.3 (2026-08-24):

| Bot library | Bot API supported | Bot API release date |
|---|---|---|
| aiogram | 10.3 | 2026-08-24 |
| grammY | 10.3 | 2026-08-24 |
| python-telegram-bot | 10.0 | 2026-05-08 |
| Telegraf | 7.1 | 2024-02-16 |

Notes on reading the numbers:

- Open issue counts depend on each project's triage habits. Django tracks bugs in Trac, so its GitHub count is 0. FastAPI moves most reports to GitHub Discussions, so its count is near 0. Use the release and commit columns for activity, not the issue column.
- Telegraf's npm `latest` tag is still 4.16.3. It has had no release in 32 months and supports a Bot API version from February 2024. The repo is not archived.

## Backend options

### FastAPI (Python)

- Maturity: very widely used, but still versioned 0.x. The docs say each minor release "could potentially have breaking changes" and advise pinning to a minor range, such as `>=0.45.0,<0.46.0`. Four minor or patch releases shipped between 2026-10-22 and 2026-10-23.
- Typing: request and response models are Pydantic models. Type hints on handler parameters drive validation and OpenAPI output.
- DI: `Depends()` builds a tree of dependencies per request. Dependencies can be sync or async and can `yield` for setup and teardown. The docs show one database session per request through a `yield` dependency. The docs describe `Depends` for path operations only, so services used outside a request (for example, a background job) need their own wiring.
- Testing: the docs have a page on replacing dependencies in tests ("Testing Dependencies with Overrides").
- Async model: ASGI, async or sync handlers.
- Postgres: no ORM built in. The tutorial uses SQLModel, which sits on SQLAlchemy and Pydantic, and says "you can use any database that you want." For migrations it says to use Alembic directly for now. SQLModel itself is at 0.0.48.
- Growth: FastAPI gives routing, validation, and DI, and leaves layout to you. Layering (routes, services, repositories) is a choice the team makes and enforces. That is good practice for learning architecture, and also the main risk: nothing stops a handler from talking to the database directly.

### Litestar (Python)

- Maturity: smaller project, 8,532 stars, 5 maintainers per its docs, "community-driven." Version 2.x, with deprecations already marked for removal in 3.0 (for example, implicit name-based DI is deprecated since 2.24).
- Typing: models can be dataclasses, Pydantic models, `TypedDict`, or `msgspec.Struct`. Built-in DTOs control what fields go in and out.
- DI: `Provide` wraps a callable. Dependencies can be declared at app, router, controller, or handler level, and a lower level overrides a higher one with the same key. `yield` dependencies run cleanup after the handler returns and before the response goes out. Results can be cached with `use_cache=True`.
- Testing: the docs have a Testing section. Layer overrides of DI also help tests.
- Async model: ASGI. Blocking sync dependencies should set `sync_to_thread=True`, and Litestar warns if you leave it unset.
- Postgres: a first-party SQLAlchemy plugin (`litestar[sqlalchemy]`) with session injection, a serialization plugin, and a repository layer. The repository code lives in `advanced-alchemy`, a separate project under the same org (v1.12.0, 2026-10-19).
- Growth: Controllers and layered DI give more built-in structure than FastAPI. The smaller community means fewer answers online and fewer third-party examples, and the open issue count (232) is the highest of the backend candidates.

### Django (Python)

- Maturity: the oldest and most stable option here. It has a published support schedule. 5.2 is LTS with extended support to April 2028. 6.1 is the current release. 6.2 is the next LTS, due April 2027.
- Typing: Django does not ship its own type hints. The third-party `django-stubs` project adds them (v6.1.2, 2026-10-05).
- DI: none built in. Django code usually calls modules and the ORM directly.
- Testing: a built-in test runner and test client, with a test database created per run.
- Async model: async views work, and fully under ASGI. The async ORM covers most queries with `a`-prefixed methods. The docs say "Transactions do not yet work in async mode" and recommend a sync function wrapped with `sync_to_async()` for transactional code. Recording a Transaction with its Shares will need a database transaction, so this limit applies directly to Kitty.
- Postgres: built-in ORM and migrations, psycopg 3 support, and built-in connection pooling through `OPTIONS["pool"]` with psycopg. Supports PostgreSQL 15 and higher.
- API layer: Django does not produce JSON APIs on its own. The common choices are Django REST Framework (3.18.3, 2026-10-06) or Django Ninja (v1.7.1, 2026-09-19), which uses type hints and Pydantic in a FastAPI-like style.
- Growth: Django has strong conventions (apps, models, migrations, admin). This keeps a codebase uniform, but its "fat model" style ties domain logic to the ORM. Separating the domain from persistence, which is a common senior-level pattern, takes deliberate work against the framework's defaults.

### NestJS (TypeScript)

- Maturity: large and very active. v12.0.0 shipped on 27 August 2026 with ESM packages, Node 20.19+ or 22.12+, and a `nest upgrade` command for breaking changes. The v11 line still gets patches (v11.2.8, 2026-10-23).
- Typing: TypeScript throughout. v12 adds Standard Schema validation.
- DI: a full IoC container. Classes marked `@Injectable()` are registered in a module's `providers` and injected through the constructor by type. Scopes are singleton (default), request, and transient. Interfaces are erased at compile time, so injecting by interface needs an explicit token with `@Inject()`.
- Testing: `@nestjs/testing` builds a test module with `Test.createTestingModule()`, and `overrideProvider()` swaps any provider. End-to-end tests use Supertest. New ESM projects use Vitest by default.
- Async model: Node event loop. Runs on Express or Fastify as the HTTP layer.
- Postgres: database-agnostic. Documented integrations include TypeORM, Drizzle, Prisma, MikroORM, Sequelize, and Knex.
- Growth: modules, providers, and DI enforce a layered structure similar to Spring or Angular. This is the most opinionated option and the closest to what large teams use. The cost is a lot of decorators and framework concepts to learn before the first endpoint, and the framework's structure can hide whether you understand why the layers exist.

### Fastify (TypeScript)

- Maturity: a mature, fast HTTP framework under the OpenJS Foundation. It has a written LTS policy: each major gets at least six months of support, and security fixes for six months after the next major. v5 is current. v6 is in alpha.
- Typing: written for TypeScript use. Type providers (`@fastify/type-provider-typebox`, `-json-schema-to-ts`, `-zod`) infer request types from the route's validation schema.
- DI: no DI container. Fastify has an encapsulation model instead: every plugin gets its own context, decorators and hooks are visible only to that context and its children, and `fastify-plugin` exposes a plugin's decorators to its parent.
- Testing: `fastify.inject()` sends fake HTTP requests without opening a port. The testing guide uses Node's built-in `node:test` and recommends a `build()` function that returns the app, kept apart from the `listen()` call.
- Async model: Node event loop, async handlers and hooks.
- Postgres: no ORM. Pair with any Node library (Drizzle, Prisma, Kysely, `pg`).
- Growth: Fastify gives plugins and encapsulation, not layers. The team designs the layering and wiring by hand, as with FastAPI. NestJS can also run on top of Fastify, so the two are not strictly exclusive.

### Backend comparison

| Criterion | FastAPI | Litestar | Django | NestJS | Fastify |
|---|---|---|---|---|---|
| Version and stability | 0.x, minor releases may break | 2.x, 3.0 deprecations announced | LTS schedule | 12.x, upgrade tool | 5.x, written LTS policy |
| Typing | Pydantic, type hints | dataclasses, Pydantic, msgspec, DTOs | third-party stubs | TypeScript | TypeScript, type providers |
| DI | per-request `Depends` | layered `Provide` | none | full IoC container, scopes | none, plugin encapsulation |
| Test helpers | dependency overrides | DI overrides by layer | test runner and client | test module, `overrideProvider` | `inject()` |
| Async | ASGI | ASGI | ASGI, but no async transactions | Node | Node |
| Postgres tooling | bring your own (SQLModel, SQLAlchemy, Alembic) | first-party SQLAlchemy plugin, repositories | built-in ORM, migrations, pooling | bring your own, many documented | bring your own |
| Structure out of the box | low | medium | high, ORM-centric | high, module-centric | low |

## Bot options

All four libraries handle routing of updates, middleware, and both long polling and webhooks. The differences that matter for Kitty are Bot API currency, how they hold conversation state, and typing.

### aiogram (Python)

- Bot API 10.3, Python 3.10+, built on asyncio. Has type hints and states it "can be used with mypy."
- Structure: routers ("Blueprints"), middlewares for updates and API calls, and "magic filters" for matching updates.
- Conversation state: a finite state machine (FSM). Storages are in-memory (default, lost on restart, "not recommended for production"), Redis, and PyMongo, or a custom `BaseStorage` subclass. There is no Postgres storage built in.
- Built-in i18n with gettext or Fluent.
- Maintenance: releases about monthly through 2026-08, 80 commits in the last 12 months, a small core team.

### python-telegram-bot (Python)

- Bot API 10.0, three versions behind aiogram and grammY (10.3), Python 3.11+, asyncio since v20. The README says it is "Fully annotated with static type hints."
- Structure: an `Application` with handlers and handler groups. Multi-step dialogs use `ConversationHandler`.
- Conversation state: persistence classes `PicklePersistence`, `DictPersistence`, or a custom `BasePersistence` subclass. It can persist user, chat, and bot data, conversation states, and callback data.
- Optional extras for webhooks (tornado), a job queue (APScheduler), and rate limiting.
- License is LGPL-3. Using it as a dependency is fine. Modified copies of the library itself must stay LGPL-3.
- Maintenance: the largest Python bot community (29,534 stars), releases every two to four months.

### grammY (TypeScript)

- Bot API 10.3, MIT. Runs on Node.js and Deno, and ships a web bundle for platforms such as Cloudflare Workers.
- Structure: middleware-based core with a router plugin. Plugins come in three tiers: built into the core (sessions, keyboards, media groups, inline queries), official `@grammyjs/*` packages (conversations, menu, runner, auto-retry, rate limiting, i18n, and others), and unvetted third-party plugins.
- Conversation state: sessions with pluggable storage adapters, or a custom adapter, which is any object matching the storage interface. Lazy sessions skip database reads and writes when no handler touches the session. The `conversations` plugin builds multi-step dialogs on top.
- Maintenance: releases about monthly through 2026-08, 87 commits in the last 12 months.

### Telegraf (TypeScript)

- Written in TypeScript and ships full typings. Koa-style middleware with `await next()`, sessions, and scenes for multi-step dialogs. Webhooks can attach to an existing http, Express, or Fastify server.
- Bot API 7.1. Telegram has shipped Bot API 7.2 through 10.3 since then (March 2024 to August 2026), and Telegraf's typed API covers none of them.
- No release since 2024-02-29 and 13 commits in the last 12 months. It is not formally archived, but by release activity it is the least maintained candidate in this file.

### Bot comparison

| Criterion | aiogram | python-telegram-bot | grammY | Telegraf |
|---|---|---|---|---|
| Bot API | 10.3 | 10.0 | 10.3 | 7.1 |
| Last release | 2026-08-25 | 2026-06-12 | 2026-08-26 | 2024-02-29 |
| Typing | type hints, mypy | full type hints | TypeScript | TypeScript |
| Conversation state | FSM with Redis or Mongo storage | `ConversationHandler` with pickle or custom persistence | sessions with storage adapters, `conversations` plugin | sessions and scenes |
| License | MIT | LGPL-3 | MIT | MIT |

## Trade-offs to weigh

These are the trade-offs the decision ticket will need to settle. None of them has one right answer.

1. One language or two. The map says Python and TypeScript are both used across the project. Options:
   - Same language for bot and backend. One toolchain, shared types or models between the bot and the API client, one Dockerfile pattern.
   - Different languages. Forces a real API contract (for example, an OpenAPI schema) between the services, which matches ADR 0001's Channel-neutral API and teaches contract-first design. Costs two toolchains, two test setups, and two sets of dependency updates on the VPS.
2. Structure given versus structure designed. NestJS and Django hand you a layout. FastAPI and Fastify hand you almost none, and Litestar sits between. A given layout is faster to start and keeps two people consistent. Designing your own layering (domain, application services, repositories, adapters) teaches more about why layers exist, but needs discipline and code review to stay intact.
3. DI style. NestJS has a full container with scopes. FastAPI and Litestar have per-request function DI. Django and Fastify have none, so wiring is by hand or through modules and plugin decorators. Container DI is the pattern most large codebases use. Hand wiring keeps every dependency visible in code.
4. Domain model and ORM coupling. Django's ORM is tightly bound to the framework, and its models usually are the domain model. The others leave the ORM choice open, which fits a ledger domain kept separate from persistence but means one more decision (already on the map as "ORM or query layer, and migration tooling").
5. Async transactions. Recording an Expense will likely write the Transaction and its Shares in one database transaction. FastAPI and Litestar with SQLAlchemy's async session, and Node with any of the listed ORMs, support transactions in async code. Django's async ORM does not yet, so transactional code runs sync through `sync_to_async()`.
6. Stability and upgrade work. Django and Fastify publish support windows. FastAPI is 0.x and can break on minor releases, so it needs pinning and tests before each upgrade. NestJS ships majors with an upgrade tool. Litestar has a 3.0 coming with announced removals.
7. Bot library currency. aiogram and grammY track the latest Bot API within days. PTB trails by a few versions. Telegraf has stopped tracking it. By release activity, the maintained choices are aiogram or PTB in Python and grammY in TypeScript.
8. Conversation state storage. aiogram's built-in storages are memory, Redis, and Mongo. PTB's are pickle files and dicts. grammY's are pluggable adapters. If the stack should stay Postgres-only on the VPS, each needs either a Redis container, or a custom storage written against Postgres or the backend API. All three support custom storage.

## Open questions for the decision ticket

- Do we want the bot and backend in the same language, or split to force an API contract?
- Do we want a framework that gives structure (NestJS, Django), or one where we design the layers ourselves (FastAPI, Litestar, Fastify)?
- Are we fine adding Redis to the VPS for bot conversation state, or do we want state in Postgres or behind the backend API?

## Sources

Maintenance data: GitHub GraphQL API, `repository` fields `stargazerCount`, `releases`, `defaultBranchRef.target.history(since:)`, and `issues(states: OPEN)`, queried 2026-10-28. npm registry for Telegraf's `latest` tag.

Telegram:

- [Bot API changelog](https://core.telegram.org/bots/api-changelog)

FastAPI:

- [About FastAPI versions](https://fastapi.tiangolo.com/deployment/versions/)
- [Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [SQL (relational) databases](https://fastapi.tiangolo.com/tutorial/sql-databases/)
- [Testing dependencies with overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)

Litestar:

- [Litestar docs home](https://docs.litestar.dev/latest/)
- [Dependency injection](https://docs.litestar.dev/latest/usage/dependency-injection.html)
- [SQLAlchemy usage](https://docs.litestar.dev/latest/usage/databases/sqlalchemy/index.html)
- [advanced-alchemy repo](https://github.com/litestar-org/advanced-alchemy)

Django:

- [Download and supported versions](https://www.djangoproject.com/download/)
- [Asynchronous support](https://docs.djangoproject.com/en/stable/topics/async/)
- [Databases: PostgreSQL notes](https://docs.djangoproject.com/en/stable/ref/databases/)
- [django-stubs repo](https://github.com/typeddjango/django-stubs)
- [Django REST framework repo](https://github.com/encode/django-rest-framework)
- [Django Ninja repo](https://github.com/vitalik/django-ninja)

NestJS:

- [Providers](https://docs.nestjs.com/providers)
- [Testing](https://docs.nestjs.com/fundamentals/testing)
- [Database](https://docs.nestjs.com/techniques/database)
- [v12.0.0 release notes](https://github.com/nestjs/nest/releases/tag/v12.0.0)

Fastify:

- [Encapsulation](https://fastify.dev/docs/latest/Reference/Encapsulation/)
- [Testing guide](https://fastify.dev/docs/latest/Guides/Testing/)
- [Type providers](https://fastify.dev/docs/latest/Reference/Type-Providers/)
- [LTS policy](https://fastify.dev/docs/latest/Reference/LTS/)

Bot libraries:

- [aiogram README](https://github.com/aiogram/aiogram)
- [aiogram FSM storages](https://docs.aiogram.dev/en/latest/dispatcher/finite_state_machine/storages.html)
- [python-telegram-bot README](https://github.com/python-telegram-bot/python-telegram-bot)
- [python-telegram-bot BasePersistence](https://docs.python-telegram-bot.org/en/stable/telegram.ext.basepersistence.html)
- [grammY README](https://github.com/grammyjs/grammY)
- [grammY plugins](https://grammy.dev/plugins/)
- [grammY sessions](https://grammy.dev/plugins/session)
- [Telegraf README](https://github.com/telegraf/telegraf)
