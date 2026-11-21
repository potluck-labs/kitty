# Backend uses Kysely with hand-written SQL migrations run by dbmate

ADR 0007 keeps the domain in plain TypeScript, has repositories turn aggregates into rows by hand, and runs statistics as SQL that sums Postings. An entity ORM would add nothing to that design, because its entity classes and change tracking would have to stay out of the domain. So `kitty-backend` uses Kysely, a typed query builder, on the standard `pg` driver. The schema is defined by SQL migrations we write by hand, and dbmate runs them. After migrating, `kysely-codegen` reads the database and writes the TypeScript types Kysely uses, and dbmate writes the whole current schema to `db/schema.sql`. Both files are committed. We chose SQL-first because the database enforces the money rules: foreign keys, checks, and grants that keep Postings append-only (ADR 0004). A TypeScript schema can't express those rules, so the SQL files are the single place where the full schema lives, and a PR shows exactly the SQL that production will run.

Migrations only go forward. Down scripts may be left empty. A mistake is fixed by redeploying the previous image, which works because of expand and contract (ADR 0005), or by a new migration. The pre-migration dump (ADR 0006) is for emergencies. A deploy takes the dump, then runs `docker compose run --rm migrate` with the same backend image, then starts the new backend. Migrations never run when the app starts. The database has two roles. `kitty_owner` owns the tables and is used only by `migrate`. `kitty_app` runs the backend with only the rights it needs, and has no `UPDATE` or `DELETE` on Postings. A table's owner can always change its own table, so append-only holds only if the app is not the owner.

CI starts an empty Postgres, runs every migration, and regenerates the types file and `db/schema.sql`. It fails if either differs from what is committed, or if a migration already on `main` was edited. Production never reruns a migration it has already applied, so editing one would make tests and production differ without any error. The unit of work from ADR 0007 is carried by `@nestjs-cls/transactional` with its Kysely adapter, behind our own `allOrNothing` interface, so only that adapter imports the library. Queries use the database's `snake_case` names with no renaming plugin, because the plugin doesn't rename inside raw SQL. Postgres returns sums over integers as `int8`, which `pg` hands back as strings, so we parse `int8` into JS numbers and throw if a value is not a safe integer.

## Considered Options

- TypeORM, MikroORM, or Sequelize. Their main feature is entity classes that save themselves. We would have to write every model twice, once as an aggregate and once as an entity, and these libraries are weakest at the summing queries statistics need.
- Drizzle with `drizzle-kit` generating migrations from a TypeScript schema. Routine changes are faster, but grants, triggers, and data backfills would be hand-written SQL outside the TypeScript schema, so it would look complete without being complete. Drizzle 1.0 was still a release candidate in October 2026.
- Prisma. Its schema is in a separate language. Statistics would go through TypedSQL files, and the database-only rules would live outside the schema, as with Drizzle.
- Kysely's own migrator. It is one less tool, but each migration is SQL inside a TypeScript string, and it has no schema file. A close call.
- node-pg-migrate. It also runs plain SQL files from inside pnpm, but has no schema file.
- Down migrations. They are rarely run, so they break without anyone noticing, and they can't bring back dropped data.
- Running migrations when the app starts. A failed migration would crash the app, and nothing would guarantee the dump comes first.
- One role that both owns the tables and runs the app. It is simpler, but `REVOKE` would not protect Postings.
- Kysely's `CamelCasePlugin`. Raw SQL would use one naming style and builder queries another.
- Returning `int8` as JS `bigint` or as strings. `bigint` is awkward everywhere, including in JSON, and strings join instead of adding (`"1250" + 50` is `"125050"`).

## Consequences

Every schema change is hand-written SQL, and generating the types needs a running Postgres. Any purge that must change Postings, such as the erasure exception in ADR 0004, runs as `kitty_owner` outside the app.
