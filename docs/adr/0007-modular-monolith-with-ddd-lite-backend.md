# Backend is a modular monolith with DDD-lite and ports and adapters in each module

`kitty-backend` is one NestJS deployable split into five modules named after the glossary: users (User, Channel identity, Account, Default Account), groups (Group, Member, Invite code, Channel chat), ledger (all five kinds of Transaction, Postings, Splits, Shares, Category, Merchant), statistics (read queries only), and notifications (the outbox), plus a small shared folder for `Money` and ID types. All modules sit in one bounded context and speak the language of `GLOSSARY.md`. Inside each module are three layers: a domain layer of aggregates, value objects, and domain services in plain TypeScript; an application layer of use cases that only orchestrate; and adapters for HTTP coming in and Postgres going out. Dependencies point inward only, and a lint rule fails CI if a domain file imports NestJS, the ORM, or an adapter. We chose this because the ledger's rules (ADR 0004) and the permission rules must be testable without a database and kept in one place as features grow.

A module offers other modules one front-desk file of public functions in its application layer, and only other modules' application layers call it. Lint forbids every other import across modules. Each table has one owning module that alone writes to it, and foreign keys still link tables across modules so the database rejects a Posting for an Account that does not exist. Transaction, Group, and User are aggregate roots, each with one repository that saves it whole. A use case makes several saves all or nothing through a unit-of-work interface it owns, and the database adapters join that database transaction on their own, so a Notification lands in the outbox with the change that caused it (ADR 0005).

Writes go through the domain. Reads do not: a statistics request goes from the HTTP adapter to a thin application interface and then to an SQL query adapter that sums Postings in the database, skipping aggregates and repositories. Statistics is the one module allowed to read another module's tables (the ledger's), read-only. It checks Group membership through the groups front desk before running a query, and returns 404 if the acting User is not a Member.

Request mechanics stay at the HTTP edge: a NestJS guard checks the Channel's API key and resolves `Kitty-Actor` to a User, an interceptor replays `Idempotency-Key`, and a pipe parses `If-Match` into an expected revision, which the Transaction aggregate then checks. The domain reports refusals as named errors such as `NotPermitted`, `NotFound`, and `OutdatedRevision`, and one exception filter maps them to 403, 404, and 412. Any other error is a logged 500. Rules receive randomness and the clock from outside so tests are repeatable. Tests run on four levels: domain unit tests with no Nest and no database; use-case tests with in-memory fakes; adapter tests against a real Postgres in Docker; and a few whole-backend tests that check responses against `channel-api.yaml` and record through the API and then read balances back.

## Considered Options

- Layered controllers, services, and repositories across the whole app, the NestJS default. It is simpler, but ledger rules spread through services and every rule test needs the DI container.
- One hexagon for the whole app. It keeps rules pure but draws no boundaries between modules.
- Full DDD with several bounded contexts, translation layers, and domain events everywhere. Too much ceremony for one domain and two developers. Event sourcing is also unnecessary because ADR 0004 already gives append-only history.
- Reads through aggregates, summing in TypeScript. It needs one path, but loads every Transaction to show one number and fills the domain with loops that decide nothing.
- Summing methods on the ledger repository. It is fast, but the repository turns into a mix of saving and reporting code.
- Stored balances updated on every write. Reads are fastest, but the same fact lives twice and contradicts ADR 0004.
- Ports owned by each consuming module instead of calling front desks directly. Purer, but an extra file for every question one module asks another, inside a single context.
- One Postgres schema per module with no foreign keys between them. It makes a later split into services easier, but gives up the database's checks on money for a split we do not plan.
- Result values instead of thrown named errors. More explicit, but every step must pass failures on by hand, against how TypeScript and NestJS work.

## Consequences

Statistics queries depend on how the ledger stores Postings. The ledger's front desk publishes what each Ledger account means, and the write-then-read tests catch drift. Which library carries the unit of work is decided with the ORM, in ORM and migration tooling for the backend.
