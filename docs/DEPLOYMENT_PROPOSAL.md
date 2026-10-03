# Deployment Proposal

## Document status

- Scope: production hosting for the stamp inventory application
- Original research date: September 1, 2026
- Repository and pricing review: October 3, 2026 (verification limits noted below)
- Currency: USD before tax unless a provider states otherwise
- Current stack: Next.js 16 App Router, React 19, Prisma 7, SQLite, and SuperTokens with Google and Apple login

Provider prices and limits change. Recheck the linked pricing pages before provisioning or approving a recurring budget.

The repository still uses SQLite. `pnpm db:init` already runs
`prisma migrate deploy`, and CI already checks migrations against an empty
database. A hosting provider has not been selected in repository configuration;
[#30](https://github.com/majestic-owl448/stamps-v2/issues/30) tracks the persistent
preview. PostgreSQL and Railway below remain recommendations.

## Recommendation

Use Railway for the first production release:

```text
Railway project
├── Next.js service
└── PostgreSQL service

External services
└── SuperTokens managed Cloud
```

This is the shortest path from the current repository to a persistent deployment. Railway has an official [Next.js and Postgres deployment guide](https://docs.railway.com/guides/nextjs), can run Prisma migrations before a release, and keeps the application and database on private networking within one project. Its Hobby plan starts at $5 per month, includes $5 of resource usage, and then bills actual CPU, memory, storage, and egress.

Migrate Prisma from SQLite to PostgreSQL before deployment. Do not treat a persistent SQLite volume as the production destination: it ties writes to one machine and makes replication, failover, and horizontal application scaling harder.

Keep SuperTokens managed Cloud for the first release. Core features cost $0 below 5,000 monthly active users. Manual account linking is optional and has a $100 monthly minimum. The current PRD requires Google and Apple sign-in, but does not require a person to link both identities. Replacing SuperTokens with direct Google and Apple OAuth would make this project responsible for sessions, cookies, token validation, revocation, and the user-auth data model. Decide whether linked login is a product requirement before implementing it or accepting the add-on budget.

The strongest alternative is Vercel Pro with Neon Postgres. Choose it if native Next.js deployment and preview ergonomics matter more than keeping the app and database under one provider. The expected paid floor is $20 per month for Vercel Pro; Neon can begin on its free tier and become usage-based.

## Required work for the proposed PostgreSQL deployment

1. Replace the Prisma SQLite datasource and `better-sqlite3` adapter with PostgreSQL.
2. Create and test Prisma migrations from an empty production-like database.
3. Reuse `pnpm db:init` (`prisma migrate deploy`) as the release migration command after the provider, adapter, and migration set have been changed to PostgreSQL.
4. Keep separate production, preview, local-development, and test database URLs.
5. Configure the public application URL, SuperTokens connection settings, Google credentials, and Apple credentials as provider secrets.
6. Register the final production and preview callback URLs with Google, Apple, and SuperTokens.
7. Add a health endpoint that checks the application process without exposing secrets. Decide separately whether it should fail when the database is unavailable.
8. Add database backups and perform a restore test before storing irreplaceable inventories.
9. Add budget alerts or hard usage limits where the provider supports them.
10. Run `pnpm lint`, `pnpm test --run`, `pnpm build`, migrations, login, and one inventory create/edit/delete flow against the deployed environment.

## Application hosting comparison

The base costs below describe the least expensive usable tier, not a guaranteed bill. Usage-based platforms charge for the resources actually consumed, and free tiers often sleep or restrict commercial use.

| Platform | Next.js compatibility | Database in the same provider/project | Base cost | How cost grows | Assessment |
| --- | --- | --- | --- | --- | --- |
| Railway | Official Next.js support; use standalone output. Supports a Prisma pre-deploy migration command. | Yes. PostgreSQL is another service in the same project over private networking. | Free has $1 monthly usage credit. Hobby is $5/month including $5 usage. Pro is $20/month including $20 usage. | RAM is $10/GB/month, CPU $20/vCPU/month, volume storage $0.15/GB/month, and egress $0.05/GB. The final bill is the plan minimum or actual metered usage, whichever is higher. | Recommended for the first release. One project, simple topology, and no serverless connection constraints. |
| Vercel | Native platform for Next.js, including App Router and Functions. | No first-party SQL database. Connect a Marketplace or external Postgres provider such as Neon. Local SQLite is unsupported because the runtime filesystem is ephemeral. | Hobby is $0 but restricted to personal, non-commercial use. Pro is $20/developer/month and includes $20 usage credit. | Pro Function overages start at $0.128/CPU-hour, $0.0106/GB-hour of provisioned memory, and $0.60 per million invocations; bandwidth and other products have separate meters. Database cost is additional. | Best Next.js experience. Recommended alternative with Neon, especially for preview deployments. |
| Render | Official Next.js web-service deployment. | Yes. Render Postgres can share a private network with the web service. A persistent disk can hold SQLite, but it is paid-only, attaches to one instance, and prevents multi-instance scaling and zero-downtime deploys. | Free web services sleep after 15 idle minutes. A Starter web service is $7/month and paid Postgres starts at $7/month, for a $14/month paid floor. | Scale by selecting larger fixed-size instances, increasing database compute/storage, or adding instances. Persistent disks and outbound bandwidth are separate. | Straightforward paid alternative. Confirm the live database price in the dashboard because Render's main pricing table is dynamically rendered. |
| Fly.io | Official Next.js Docker deployment; standalone output is recommended. | Yes. It can run Postgres and can mount a volume for SQLite. Volumes are local to a machine and cannot be shared between machines. | A continuously running shared-cpu-1x Machine is about $2.02/month at 256 MB, $3.32 at 512 MB, or $5.92 at 1 GB. Real Next.js memory needs may require the larger sizes. | Add Machines, memory, regions, Postgres resources, $0.15/GB-month volumes, and network egress. North America and Europe egress is $0.02/GB. | Low infrastructure floor and good control, but more Docker, networking, database, and availability work. |
| DigitalOcean App Platform | Official Next.js sample and Node application support. | A development PostgreSQL database can be attached to the same app. Production managed PostgreSQL is a separate DigitalOcean resource. | App container: $5/month for 512 MB or $10/month for 1 GB. Development PostgreSQL is $7/month. Production managed PostgreSQL starts at $15/month for a 1 GB single node. | Increase container size or count. Managed database nodes, standby nodes, and storage add cost; extra Standard Edition database storage is $0.215/GiB-month. | Predictable pricing. A paid app plus production database starts around $20/month before auth and extras. |
| Netlify | Broad Next.js support through OpenNext, including App Router, Server Components, route handlers, and SSR. | Yes on credit-based plans through Netlify Database, a managed Postgres product. External Postgres also works. | Free is $0 with 300 credits. Personal is $9/month with 1,000 credits. Pro starts at $20/month with 3,000 credits. | Production deploys cost 15 credits, compute 10 credits/GB-hour, bandwidth 20 credits/GB, and web requests 2 credits/10,000. Netlify Database compute is 10 credits/unit and database bandwidth 20 credits/GB. Its documentation had not published the post-promotion storage rate by the research date. | Compatible, but the shared credit model and missing storage rate make app and database cost less predictable than Railway for this project. |

Official sources: [Railway plans and resource rates](https://docs.railway.com/pricing/plans), [Railway PostgreSQL](https://docs.railway.com/databases/postgresql), [Vercel plans](https://vercel.com/pricing), [Vercel Function pricing](https://vercel.com/docs/functions/usage-and-pricing), [Vercel SQLite limitation](https://vercel.com/kb/guide/is-sqlite-supported-in-vercel), [Render Next.js](https://render.com/docs/deploy-nextjs-app), [Render persistent disks](https://render.com/docs/disks), [Render service and database starting costs](https://render.com/articles/hosting-n8n-on-render-for-llm-powered-automation), [Fly.io Next.js](https://fly.io/docs/js/frameworks/nextjs/), [Fly.io pricing](https://fly.io/docs/about/pricing/), [Fly.io volumes](https://fly.io/docs/js/the-basics/volumes/), [DigitalOcean App Platform pricing](https://docs.digitalocean.com/products/app-platform/details/pricing/), [DigitalOcean PostgreSQL pricing](https://docs.digitalocean.com/products/databases/postgresql/details/pricing/), [Netlify Next.js](https://docs.netlify.com/build/frameworks/framework-setup-guides/nextjs/overview/), [Netlify plans](https://www.netlify.com/pricing/), and [Netlify Database billing](https://docs.netlify.com/build/data-and-storage/netlify-database/billing-and-usage/).

The October review confirmed the linked Railway, Vercel, Render, DigitalOcean,
Netlify, Supabase, and SuperTokens pricing sources. Fly.io machine estimates vary
by region. Neon's October 2 announcement confirms the updated free storage allowance.
Its paid pricing page could not be retrieved during this review, so the paid
Neon figures below remain the September snapshot and need confirmation before
selection. Netlify still does not
publish a storage rate on its linked database billing page.

## Database-only hosting comparison

These options apply when the application host does not include suitable database hosting, or when separating the database is preferable. All listed products provide PostgreSQL and are compatible with a Prisma migration away from SQLite.

| Provider | Base cost and allowance | How cost grows | Fit for this project |
| --- | --- | --- | --- |
| Neon | Free: 100 compute-unit hours per project per month, 1 GB storage per project, and scale-to-zero. The September snapshot lists Launch at $0.106/CU-hour and $0.35/GB-month; Neon gives about $15/month as an example for an intermittent 1 GB workload. | Pay for active compute, stored data, and network transfer beyond allowances. Larger plans add higher limits and operational features. | Recommended with Vercel. Serverless connection handling, branching, and scale-to-zero suit previews and a small initial workload. |
| Supabase | Free: 500 MB database, 5 GB egress, and two active projects; a project pauses after one inactive week. Pro starts at $25/month and includes $10 compute credit, enough for one Micro instance, 8 GB disk, 250 GB egress, and seven days of daily backups. | A Small compute instance makes the Pro total about $30/month after the $10 credit; Medium is about $75. Storage above 8 GB is $0.125/GB and egress above 250 GB is $0.09/GB. | Strong database product, but its bundled Auth, Storage, and APIs overlap with features this repository does not currently need. |
| Railway PostgreSQL | Shares the Railway account's $5 Hobby or $20 Pro included usage. There is no separate fixed database-plan fee. | Same resource meters as application services: $10/GB-month RAM, $20/vCPU-month CPU, $0.15/GB-month volume storage, and egress when public networking is used. | Best when the app is also on Railway. Private networking avoids public database egress. Backups and production readiness still need explicit configuration. |
| Render Postgres | A separate managed service with a free 30-day evaluation database; paid service starts at $7/month. | Increase compute and disk independently, then add higher availability and retention as needed. | Reasonable with a Render web service. Check the dashboard quote before choosing it because the main public pricing table does not expose a stable text table. |
| DigitalOcean Managed PostgreSQL | $15/month for a 1 GB single node. High availability begins with a $30/month primary plus at least one matching $30/month standby. | Read-only nodes start at $15/month; additional Standard Edition storage is $0.215/GiB-month. From October 15, 2026 for new accounts and November 30 for all accounts, new Standard Edition clusters cannot add standby/read-only nodes or exceed 4 GiB RAM; Advanced Edition begins at $130/month per 8 GiB node. | Predictable and conventional, but more expensive than the likely first-release workload needs. |

Official sources: [Neon pricing](https://neon.com/pricing), [Neon free-plan update](https://neon.com/blog/neon-free-plan-1-gb-per-project), [Supabase pricing](https://supabase.com/pricing), [Railway PostgreSQL](https://docs.railway.com/databases/postgresql), [Render Postgres](https://render.com/docs/postgresql), [Render database starting cost](https://render.com/articles/hosting-n8n-on-render-for-llm-powered-automation), and [DigitalOcean PostgreSQL pricing](https://docs.digitalocean.com/products/databases/postgresql/details/pricing/).

### Database placement rules

- Put the application and database in the same geographic region where possible. Latency between every Prisma query and an external database matters more than static-page delivery latency.
- Use a pooled database URL for serverless application runtimes and a direct URL for Prisma migrations when the provider supplies both.
- Do not run production migrations during `next build`. Run them once as a release or pre-deploy step.
- Do not share a production database with preview deployments. Preview migrations can make production code and schema incompatible.
- Backups are useful only after a restore has been tested.
- If SuperTokens is later self-hosted, it may use the same PostgreSQL server, but should use its own database or schema and credentials. SuperTokens supports PostgreSQL 13 or newer and can prefix its tables.

## Authentication cost comparison

OAuth and SuperTokens are not equivalent layers. Google or Apple OAuth identifies a user and returns provider tokens. SuperTokens also supplies the application session, secure cookies, refresh and revocation behavior, account linking, SDK integration, and an auth-user store.

### SuperTokens managed Cloud

For core features, the current price is $0.02 per monthly active user, with no charge below 5,000 MAU. Manual account linking costs another $0.005/MAU with a $100 monthly minimum. It is needed only if the product must let one account use both social identities; the current PRD does not state that requirement.

| Monthly active users | Core features plus account linking |
| ---: | ---: |
| 1,000 | $100/month minimum |
| 4,999 | $100/month minimum |
| 5,000 | $125/month |
| 10,000 | $250/month |
| 50,000 | $1,250/month before any volume discount |

The $100 account-linking minimum applies below 5,000 MAU. At 5,000 MAU, the page's example adds core and linking rates: `($0.02 + $0.005) * 5,000 = $125`. Confirm the invoice and any volume discount with SuperTokens before purchase. Other features have separate charges. Managed MFA is listed at $0.01/MAU with a $100 monthly minimum, and dashboard access includes three users before charging for more.

Advantages for this project:

- The integration already exists.
- There is no auth hosting or auth database to operate.
- If added, manual linking can keep one primary user ID stable across both social identities.
- A later move to self-hosted SuperTokens can preserve the same application-facing SDK model.

### Self-hosted SuperTokens

The open-source core features have no per-MAU software fee. The project instead pays for:

- An always-available SuperTokens Core service.
- PostgreSQL 13 or newer.
- Monitoring, upgrades, backups, failover, and incident response.
- A paid licence for manual account linking.

On Railway, Render, or Fly.io, the Core can run as an additional container and can use the same PostgreSQL server with isolated tables, schema, or database. Self-hosting removes the managed core hosting charge, but account linking remains a paid feature. Confirm its self-hosted licence quote before comparing totals.

### Direct Google and Apple OAuth

Google publishes setup and quota documentation for basic OpenID Connect login but no per-MAU price for direct Google Sign-In. Apple includes Sign in with Apple in the Apple Developer Program, whose current membership fee is $99 per year in the United States, with local pricing elsewhere. That Apple membership is required whether the app reaches Apple through SuperTokens or through a direct implementation.

The direct-OAuth cash floor is therefore the Apple membership plus the application and database infrastructure. The engineering scope is larger:

- Implement and secure authorization-code callbacks and state/nonce validation.
- Create application sessions and rotate or revoke them safely.
- If linked login is a product requirement, implement explicit linking, alternate-login confirmation, last-login protection, and unlinking without changing the inventory owner ID.
- Handle provider email and profile differences.
- Build auth-user administration and deletion flows.
- Maintain security fixes as provider and framework behavior changes.

Direct OAuth becomes attractive if the team is prepared to own security-sensitive session and account-linking code and wants to avoid the $100 SuperTokens add-on minimum.

### Managed OAuth alternative

Google Cloud Identity Platform is useful as a price reference for managed authentication rather than raw OAuth. At the research date, social and email-based providers included the first 50,000 MAU, then cost $0.0055/MAU from 50,000 through 99,999 with lower unit rates at larger tiers. Migrating to it would still be an authentication rewrite and would make Apple configuration and the Apple developer membership necessary.

Official sources: [SuperTokens pricing](https://supertokens.com/pricing), [SuperTokens manual account linking](https://supertokens.com/docs/post-authentication/account-linking/manual-account-linking), [SuperTokens account-linking concepts](https://supertokens.com/docs/post-authentication/account-linking/important-concepts), [SuperTokens self-hosting](https://supertokens.com/docs/deployment/self-host-supertokens), [SuperTokens architecture](https://supertokens.com/docs/references/how-supertokens-works), [Google OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect), [Google OAuth production readiness](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview), [Apple Developer Program](https://developer.apple.com/programs/whats-included/), and [Google Cloud Identity Platform pricing](https://cloud.google.com/identity-platform/pricing).

## Cost scenarios

These are planning examples, not quotes. They exclude domain registration, taxes, paid observability, email, and the Apple Developer Program fee. Usage-based bills depend on measured runtime resources, not only user count.

| Scenario | Application and database | Authentication | Expected floor |
| --- | --- | --- | --- |
| Development or preview without account linking | Railway Free if the combined services remain inside its limits; otherwise Hobby | SuperTokens Cloud core below 5,000 MAU | $0 to $5/month |
| First paid release, recommended | Railway Hobby with Next.js and Postgres; usage above $5 is charged at resource rates | SuperTokens Cloud core below 5,000 MAU | $5/month, then actual Railway usage |
| Native Next.js alternative | Vercel Pro plus Neon Free | SuperTokens Cloud core below 5,000 MAU | $20/month |
| Predictable fixed-size alternative | DigitalOcean 512 MB app plus production managed PostgreSQL | SuperTokens Cloud core below 5,000 MAU | About $20/month |
| Linked-login add-on below 5,000 MAU | Any of the above | SuperTokens Cloud account linking minimum | Hosting floor plus $100/month auth |
| Linked login at 5,000 MAU | Any of the above | SuperTokens core plus account linking is about $125/month | Hosting floor plus about $125/month auth |

The inventory workload is likely to be database-light: small records, modest write frequency, and simple aggregates. Early cost is more likely to be set by minimum service sizes and always-on memory than by database storage. The first meaningful scaling work should therefore be measuring memory, database connections, query latency, and monthly active users before increasing plan sizes.

## Proposed rollout

### Stage 1: deployment preparation

1. Select the hosting provider in the persistent-preview work and, for the proposed topology, migrate Prisma to PostgreSQL.
2. Configure the existing migration command as a release step; add the health endpoint and provider-specific environment documentation.
3. Decide whether linked login is a product requirement. Approve the SuperTokens account-linking add-on budget only if it is.
4. Confirm Google and Apple production credentials and callback URLs.

### Stage 2: preview environment

1. Create a Railway project and a non-production PostgreSQL service.
2. Deploy the Next.js service from the repository with standalone output.
3. Run migrations using Railway's pre-deploy command.
4. Verify login, profile settings, inventory isolation, valuation, and moderation.
5. Set a low usage alert while measuring idle and active resource consumption.

### Stage 3: production

1. Create a separate production database and production environment variables.
2. Enable and test backups.
3. Deploy, run migrations, and execute the release smoke tests.
4. Add the production domain and update every OAuth allowlist and redirect URI.
5. Record the actual first full-month application, database, and authentication usage.

### Stage 4: review after real usage

Review the hosting choice when any of these occurs:

- Railway spend or operational limitations exceed the value of the single-project setup.
- Preview deployments become a core workflow, making Vercel plus Neon more valuable.
- The application needs multiple instances or regions.
- Database recovery, availability, or compliance requirements exceed the selected plan.
- Before enabling paid account linking and again as SuperTokens approaches 5,000 MAU, compare managed Cloud, self-hosting, and another managed auth service.

## Decision to defer

The repository should not add provider-specific deployment configuration until the hosting provider is selected. This proposal recommends Railway, but the implementation issue can retain the final provider choice as an explicit decision before provisioning. The PostgreSQL migration applies to the proposed PostgreSQL topology. Release migrations, environment separation, and tested backups are required for whichever storage option is selected.
