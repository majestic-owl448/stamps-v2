# Stamp Inventory

Stamp Inventory is a web application for recording owned stamps and calculating
their current postage value for a selected country. The first release is under
development; its requirements and implementation sequence are in [`docs/`](docs/).

## Current functionality

Signed-in users can select one or more postal entities, set a display currency
for each, choose the active setting used for valuation, and set one timezone for
their dashboard. The dashboard lets them record monetary, named/code, and manually
entered postage values; edit owned and annulled quantities; mark an entry as
expired; remove an entry; and see the active-country total.

Users can propose named/code face values and fixed currency conversions. The
proposal workflow keeps proposed entries separate from approved shared data.
Moderators can review proposals, approve or reject them, and merge duplicates.

The personal Settings page provides timezone preferences, a JSON download of the
signed-in user's account data, and an account-deletion flow. Deletion retains approved shared contributions but
removes the deleted user's direct references to them.

## Local setup

Create the local environment file before installing dependencies. Prisma reads
`DATABASE_URL` during its post-install client generation.

```bash
cp sample.env .env
pnpm install
pnpm db:init
pnpm dev
```

Replace the placeholder SuperTokens and social login credentials in `.env` with
values for your own development apps. The local site runs at
[http://localhost:3000](http://localhost:3000).

### Local role testing

Run the development server with two fixed local profiles when you need to test
normal-user and moderator behavior without social login:

```bash
pnpm dev:test-users
```

This command applies pending migrations to the ignored `data.db.test-users` file,
clears every application table, seeds `local-test-user` with the `USER` role and
`local-test-moderator` with the `MODERATOR` role, then starts one server connected
to that database. No stamps, currencies, postal settings, proposals, or moderation
records are seeded. Each server start clears records created during the previous
local test session.

Open [the normal-user login](http://localhost:3000/api/dev-auth/user) in one
browser or browser profile. Open [the moderator login](http://localhost:3000/api/dev-auth/moderator)
in a different browser or browser profile. Each login stores its selected fixture
in that client's HTTP-only cookie, so both clients can use the same server and
database at the same time.

The bypass requires both development-auth flags and `next dev`. Production builds
ignore it and continue to require a SuperTokens session. Use `pnpm dev` when you
want to test the normal Google or Apple login flow.

## Project checks

Run all three checks before committing a functional change:

```bash
pnpm lint
pnpm test --run
pnpm build
```

The test setup creates a temporary SQLite database, applies the committed Prisma
migrations, and deletes the database after the test run. Tests do not write to
the development database configured in `.env`.

## Database

Local development uses [Prisma 7](https://www.prisma.io/docs) with SQLite through
the `better-sqlite3` driver adapter. The schema and migrations live in `prisma/`.
The generated client is written to `lib/generated/prisma` and re-exported as a
singleton from `lib/db.ts`.

Set `DATABASE_URL` to a different database for each environment:

| Environment | Connection setup |
| --- | --- |
| Development | Copy `sample.env` to `.env`. The default is the ignored `data.db` file. |
| Test | `pnpm test --run` creates a temporary database, applies every committed migration, and removes the database afterward. |
| Production | Provision persistent SQLite storage independently and store its file URL in the deployment platform's environment settings. Do not upload a development or test database. |

Initialize an empty database, or apply all pending committed migrations to an
existing database, with:

```bash
pnpm db:init
```

Prisma loads `DATABASE_URL` from `.env` unless the current process already sets
it. Set an explicit `DATABASE_URL` when targeting another environment. The command
uses `prisma migrate deploy`, which applies committed migrations in order without creating a migration or resetting
existing data. Run the same command against the independently provisioned
production database during deployment.

When changing the schema during development, create and apply the next migration:

```bash
pnpm db:migrate --name describe_change
```

Prisma writes the migration to `prisma/migrations/`. Commit the migration with
the schema change so `pnpm db:init` can apply it in test and production.

Database files and records do not move between environments; only the committed
migration files do.

To inspect a development database:

```bash
pnpm prisma studio
```

The current Prisma schema uses SQLite in every environment. A production target
must provide persistent SQLite storage outside the application deployment before
the migration command runs. A target that provides PostgreSQL instead requires a
matching Prisma provider, adapter, and migration set before deployment. Local
database files match the `data.db*` ignore rule and are not committed or included
as deployment inputs.

## Authentication

[SuperTokens](https://supertokens.com) provides Google and Apple authentication
through its managed cloud core and prebuilt React interface. Copy `sample.env` to
`.env`, then add the core connection and OAuth credentials before running the app.

The authentication code is organized as follows:

| Path | Purpose |
| --- | --- |
| `app/config/appInfo.ts` | Domains and base paths shared by frontend and backend |
| `app/config/backend.ts` | Server recipes and SuperTokens initialization |
| `app/config/frontend.tsx` | Client recipes and Next.js router integration |
| `app/api/auth/[[...path]]/route.ts` | Authentication API routes |
| `app/api/me/route.ts` | Authenticated profile endpoint |
| `app/auth/[[...path]]/page.tsx` | Prebuilt sign-in and sign-up interface |
| `app/components/supertokensProvider.tsx` | Client initialization and provider |
| `app/components/sessionAuthForNextJS.tsx` | Server-rendering-safe session guard |
| `app/dashboard/page.tsx` | Protected postal-entity settings, inventory, and proposals |
| `app/settings/page.tsx` | Protected timezone preferences, account export, and account deletion |
| `lib/userProfile.ts` | Profile creation and email updates for signed-in users |

`GET /api/me` creates or updates one `UserProfile` keyed by the SuperTokens
primary user ID. The profile stores an optional email address and a `USER` or
`MODERATOR` role. Requests without a session return `401`, and requests for a
different user ID return `404`.

Email login is not enabled because the project does not yet have an email delivery
service. The credentials in `sample.env` are placeholders and must be replaced
before deployment.
