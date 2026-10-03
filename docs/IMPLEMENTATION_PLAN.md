# Stamp Inventory Implementation Plan

## Current repository baseline

The repository currently contains:

- Next.js 16 App Router with React 19 and TypeScript.
- Tailwind CSS 4.
- SuperTokens authentication with Google and Apple.
- Prisma 7 with committed SQLite migrations and separate development, test, and production connection configuration.
- An authenticated inventory dashboard and a separate personal settings route.
- An authenticated profile route that creates or updates a `UserProfile` keyed by the SuperTokens primary user ID.
- A dashboard postal-entity workflow that lets a new user choose an available entity or create one before inventory access.
- A settings manager that lets each user add and edit postal-entity settings and select the active setting.
- Fixed currency conversions and named face-value schedules, with scheduled values resolved according to the user's dashboard local date.
- User-owned inventory entries with monetary, named/code, and manual postage values; quantity, annulled, expiration, and removal controls; and active-country totals.
- Named/code, fixed-conversion, and postal-entity proposal workflows, including moderator approval, rejection, and duplicate merging.
- JSON account-data export and account deletion controls on personal settings, retaining approved shared contributions without the deleted user's direct references.
- Continuous integration for lint, tests, build, and migration checks, plus isolated local normal-user and moderator test clients.
- Vitest and ESLint configuration, with database tests running against a temporary database.

The baseline repair is complete, and CI runs `pnpm lint`, `pnpm test --run`, and `pnpm build`. The retained test suites contain assertions. The database test applies every committed migration to a disposable SQLite database instead of writing to the development database.

Phases 1 through 11 organize the inventory features now represented in the code.
Their checklists remain verification criteria, not a claim that every release
acceptance check has been completed. Phase 12 retains the release-verification
checklist; CI is already configured, while the
persistent preview is tracked in [#30](https://github.com/majestic-owl448/stamps-v2/issues/30).
The schema sketches below summarize the model; `prisma/schema.prisma` contains
the complete fields, relations, and constraints.

Next, [#98](https://github.com/majestic-owl448/stamps-v2/issues/98) removes the
separate issuing-authority field, and
[#99](https://github.com/majestic-owl448/stamps-v2/issues/99) introduces postal-entity
currency history, currency proposals, and entity-based valuation. Their schema,
API, interface, export, test, and documentation changes remain outstanding.

The implementation should keep each phase in an atomic conventional commit. Schema migrations and their matching application changes belong in the same feature phase unless splitting them leaves both commits runnable.

## Phase 1: Repair the baseline

Suggested commit:

```text
chore: restore passing project checks
```

Work:

1. Fix the React effect lint errors in the authentication components.
2. Retain the `combRep` helper for future postage-combination work and cover it with unit tests without connecting it to the inventory release.
3. Configure database tests to use a disposable test database.
4. Replace the generated home-page text and metadata with inventory product copy.
5. Confirm `pnpm lint`, `pnpm test --run`, and `pnpm build` pass.

Exit condition: all existing checks pass before inventory tables are introduced.

## Phase 2: Prepare repeatable database setup

Suggested commit:

```text
feat(db): add repeatable setup commands
```

The repository uses SQLite in every environment until a production host is selected. Each environment has a separate database connection, and a production database must be provisioned independently rather than copied from local development. The deployment proposal recommends PostgreSQL for the first release; that migration belongs to deployment work after the hosting decision.

Work:

1. Commit the migrations required to initialize an empty database.
2. Add commands to apply committed migrations and create later development migrations.
3. Keep local, test, and production database connections separate.
4. Run database tests against a disposable database.
5. Document database initialization and deployment migration commands in the README.
6. Verify a clean database can apply every committed migration from the beginning.

Exit condition: local development, isolated tests, and independently provisioned production storage each have documented connection and migration procedures.

## Phase 3: Connect authentication to application profiles

Suggested commits:

```text
feat(auth): persist user profiles and roles
feat(settings): require initial postal entity settings
feat(settings): manage postal entity settings
```

Add:

```text
UserProfile
  id                            String primary key, SuperTokens primary user ID
  email                         String nullable
  role                          USER or MODERATOR
  timeZone                      IANA timezone, default UTC
  timeZoneMode                  SYSTEM or CUSTOM
  activePostalEntitySettingId   String nullable until initial settings are saved
  createdAt
  updatedAt

PostalEntity
  id
  name
  normalizedName
  countryCode
  issuingAuthority
  scope
  sourceUrl nullable
  sourceNote nullable
  status
  submittedById nullable
  createdAt
  updatedAt

UserPostalEntitySetting
  id
  userId
  postalEntityId
  displayCurrencyCode
  createdAt
  updatedAt

unique(userId, postalEntityId)
```

Work:

1. Replace the unrelated numeric Prisma user with a profile keyed by the SuperTokens primary user ID.
2. Create or update the profile from authenticated server-side code.
3. Add the minimum settings flow for postal entity and display currency. Block inventory routes until the first postal-entity setting exists.
4. Store one user dashboard timezone. Use the browser timezone while the user selects the system option, and allow a custom timezone on the personal Settings page.
5. Let the user add postal-entity settings and select one active setting.
6. Add a server-side moderator-role guard.
7. Remove the generic `/api/data` route.
8. Keep `/api/me` limited to authenticated profile information.

Tests:

- A missing session receives `401`.
- A profile is associated with the session user ID.
- A supplied client user ID is ignored or rejected.
- A normal user cannot call moderator endpoints.
- The first postal-entity setting becomes active.
- Postal-entity settings are isolated by user and unique by user plus postal entity.
- The active setting belongs to the authenticated user.
- Invalid country, currency, and IANA timezone values are rejected.

## Phase 4: Add reference and scheduling tables

Suggested commit:

```text
feat(valuation): add shared value schedules
```

Shared entities (pending submissions live in separate proposal tables):

```text
Currency
  code
  displayName

ValueSchedule
  id
  countryCode
  currencyCode
  createdAt
  updatedAt

ValueScheduleValue
  id
  valueScheduleId
  amount
  effectiveOn nullable
  createdAt

NamedFaceValue
  id
  countryCode
  displayCode
  normalizedCode
  valueScheduleId
  createdAt
  updatedAt

CurrencyConversion
  id
  fromCurrencyCode
  toCurrencyCode
  multiplier
  createdAt
  updatedAt
```

Constraints:

- Approved named definitions are unique by country and normalized code.
- Amounts and multipliers are non-negative decimals; conversion multipliers must be greater than zero.
- Effective dates are date-only values.
- A value schedule's values use the schedule currency.
- Pending values live in `NamedFaceValueDefinitionProposal`, `NamedFaceValueValueProposal`, or `CurrencyConversionProposal`, with a `submittedById` and proposal status. Shared records are written on approval or merge.

Do not add a postage-rate catalog in this phase. A later `PostageRate` table will reference `ValueSchedule`. Named face values and formally linked postage rates will stay synchronized because neither owns a copied amount.

Tests:

- Country and code normalization.
- Duplicate approved-definition rejection.
- Decimal storage and serialization.
- Current and upcoming resolution in different IANA timezones.
- Approved data visibility.
- Pending data visibility only to its proposer.

## Phase 5: Implement the valuation service

Suggested commit:

```text
feat(valuation): calculate current stamp values
```

Keep valuation on the server. `lib/namedFaceValue.ts` resolves current and upcoming
named values, `lib/currencyConversion.ts` resolves fixed conversions, and
`lib/stampInventory.ts` presents stamp values and calculates inventory totals.

Resolution rules:

1. Return zero with `OUTSIDE_ACTIVE_COUNTRY` when the stamp country differs from the active country.
2. Include approved entries and pending entries belonging to the current user.
3. For named values, select the latest entry whose effective date is absent or no later than the user's dashboard local date.
4. Prefer the user's eligible pending named-value proposal when it conflicts with the approved value for the same effective date.
5. Find the next eligible future named-value entry for advance notice.
6. Display the next named value when it is no more than 10 calendar days away.
7. Use the current named value for totals until the effective date arrives.

Fixed currency conversions do not have effective dates. The conversion resolver selects an approved rate or the current user's pending correction for the currency pair.

The service accepts an explicit active country setting and local date for deterministic tests. The inventory route supplies today's date in the user's saved dashboard timezone. There is no date selector in the user interface.

Use the exact decimal helpers in `lib/decimal.ts` for calculations. They operate
on decimal strings with `BigInt`; amounts are stored and serialized as strings.
Format only at the presentation layer.

Tests must cover:

- Home-currency identity conversion.
- Fixed currency conversion.
- Named/code resolution.
- Pending and approved precedence.
- Future values before, during, and after the notice window.
- Date changes at timezone boundaries.
- Country mismatch even when both countries use the same currency.
- Active-country switching without inventory rewrites.
- Zero and fractional monetary values without floating-point artifacts.

## Phase 6: Add stamp inventory persistence and APIs

Suggested commit:

```text
feat(stamps): add user-owned inventory endpoints
```

Add:

```text
StampInventoryEntry
  id
  userId
  postalEntityId
  countryCode
  name
  yearOfIssue nullable
  faceValueType       MONETARY, NAMED, or NONE
  faceAmount nullable
  faceCurrencyCode nullable
  namedFaceValueId nullable
  namedFaceValueProposalId nullable
  manualPostageAmount nullable
  manualPostageCurrencyCode nullable
  quantityOwned
  quantityAnnulled
  expired
  createdAt
  updatedAt
```

Enforce type-specific fields in shared request validation and with database constraints supported by the selected database.

Manual postage fields can accompany any face-value type. They are required for `NONE` and act as a fallback when a `MONETARY` or `NAMED` value cannot be resolved. A resolvable face value takes precedence over the stored manual fallback.

API routes:

```text
GET    /api/stamps
POST   /api/stamps
PATCH  /api/stamps/:id
DELETE /api/stamps/:id
GET    /api/settings
PATCH  /api/settings/timezone
POST   /api/settings/postal-entities
PATCH  /api/settings/postal-entities/:settingId
PATCH  /api/settings/active-postal-entity
POST   /api/settings/postal-entities/:settingId/replacement
```

Every stamp query includes the session user ID. Updates and deletions for records outside that ownership boundary return `404`.

The list response includes:

- Raw inventory fields.
- Active country and active display currency.
- Resolved current unit value.
- Usable quantity.
- Total postage value.
- Upcoming named/code value when it falls inside the notice window.
- An explanation of whether the value came from face amount, conversion, named schedule, manual entry, annulled status, or expired status.

Tests:

- Create each face-value type.
- Store and apply a manual fallback for unresolved monetary and named/code values.
- Reject invalid field combinations.
- Accept postage value zero.
- Reject negative amounts.
- Enforce owned and annulled quantity constraints.
- Apply expired and annulled zero-value rules.
- Edit quantity and remove an entry.
- Prevent cross-user read, update, and deletion.
- Recalculate after conversion and schedule changes.
- Preserve the currency of a manual value when a country setting's display currency changes.
- Require a country on every stamp and require named/code references to match it.
- Return zero with `OUTSIDE_ACTIVE_COUNTRY` for stamps from other countries.
- Recalculate values when the user switches active country without rewriting inventory entries.

## Phase 7: Add moderation workflows

Suggested commits:

```text
feat(moderation): add valuation proposals
feat(moderation): add proposal review tools
```

Add proposal and audit entities or extend the reference entities with a separate immutable proposal payload. A proposal must preserve exactly what the user submitted even if the approved record later changes.

API operations:

- Submit a named/code definition or correction.
- Submit a current or future scheduled named/code value.
- Submit a fixed currency conversion or correction.
- List the current user's proposals.
- List the moderator queue.
- Approve a proposal.
- Reject a proposal.
- Merge a duplicate proposal.

Approval behavior:

- Run the shared-data update and proposal-status update in one transaction.
- Link the proposer's pending definition to the approved definition.
- Preserve stamp references.
- Make approved data visible to every user.
- Record moderator, decision time, and decision note.
- Allow the proposer reference on approved shared records and retained moderation history to become null after account deletion.

Merge behavior:

- Select the existing canonical definition or schedule.
- Repoint the proposer's references inside the transaction.
- Mark the proposal `MERGED`.
- Reject a merge that would create incompatible named-value country or effective-date data, or an incompatible conversion currency pair.

Rejected-proposal behavior was resolved in [issue #24](https://github.com/majestic-owl448/stamps-v2/issues/24): stop using the rejected data, mark linked private records as requiring action, and let the proposer resubmit or choose an eligible replacement. Do not select an approved or manual fallback automatically. Resubmission creates a new proposal and preserves the rejected submission.

Moderator interface:

- Queue filtered by proposal type and status.
- Submitted source and proposed values.
- Existing possible matches.
- Approve, reject, and merge actions with confirmation.
- Decision note.

Tests must exercise authorization, correction proposals, duplicate handling, transaction rollback, and pending-data visibility after every decision type.

## Phase 8: Build proposal interfaces

Suggested commits:

```text
feat(valuation): add named value and conversion proposals
```

Named/code selector:

- Country selector.
- Search by display or normalized code.
- Approved results.
- The current user's pending results.
- Form to propose a missing definition or a correction to an existing definition or value.
- Current and upcoming values with effective dates.

The monetary workflow also permits a missing fixed conversion or correction to an existing conversion. Proposal forms must explain that pending data is private to the proposer until approved.

## Phase 9: Build the inventory interface

Suggested commits:

```text
feat(stamps): add inventory creation form
feat(stamps): add inventory list controls
```

Add-stamp flow:

1. Select the stamp postal entity and its country, defaulting to the active selection.
2. Enter a stamp name.
3. Optionally enter the year of issue.
4. Select monetary, named/code, or no face value.
5. Enter the fields for that type.
6. Enter a manual postage fallback when the selected face value cannot be resolved.
7. Enter quantity owned and quantity annulled.
8. Set expired when applicable.
9. Preview the resolved unit and total postage value.
10. Save the entry.

Inventory list:

- Name and optional year.
- Stamp country.
- Face value.
- Owned, annulled, and usable quantities.
- Expired indicator.
- Current unit value.
- Line total.
- Upcoming value and effective date within the notice window.
- Edit quantity and status controls.
- Remove action with confirmation.
- Overall total in the active country's display currency.
- A zero-value explanation for entries outside the active country.

The interface must use `annulled`; it must not label cancelled stamps as `used`.

Every form and control introduced in Phases 8 and 9 must be keyboard accessible, have visible labels and associated errors, and communicate status without relying on color. Format monetary values with `Intl.NumberFormat` and display stored calendar dates in the user's locale.

## Phase 10: Add JSON user data export

Suggested commit:

```text
feat(settings): add user data export
```

Add an authenticated `GET /api/account/export` endpoint that returns one JSON attachment with `Cache-Control: no-store`. Give the document an explicit schema version and generation time. Serialize decimals and date-only values as strings.

The export service gathers:

- SuperTokens account metadata exposed to the application, excluding tokens and provider secrets.
- Profile, postal-entity settings, inventory, and private valuation records.
- Every proposal submitted by the user, regardless of status.
- Shared definitions, schedule values, conversions, and later postage-rate records that retain a contributor link to the user.
- Moderation and audit entries that refer to the user as proposer, affected account, or moderator.

Do not expose another user's private ID or email through a shared moderation record. Keep record identifiers and non-private decision data so relationships in the export remain understandable.

Maintain one export mapping for every table with a user foreign key or stored authentication ID. Add a schema coverage test that fails when a new user-linked table or field has neither an export mapping nor an explicit secret exclusion.

Tests:

- Reject an unauthenticated export request.
- Return parseable JSON with an attachment filename, schema version, and generation time.
- Prevent browser and intermediary caching through the response headers.
- Include fixtures for every user-owned and user-linked record type.
- Include approved and merged contributions without changing the shared records.
- Include moderation entries linked to the user as proposer and moderator.
- Preserve decimal and date-only strings exactly.
- Exclude sessions, tokens, secrets, password material, and another user's private account fields.
- Prevent one user from exporting another user's private data.

## Phase 11: Add account deletion

Suggested commit:

```text
feat(settings): add account deletion
```

Add an authenticated `DELETE /api/account` operation with explicit confirmation. Treat deletion as an idempotent workflow because application records and the SuperTokens identity cannot be deleted in one database transaction.

Workflow:

1. Create an account-deletion job and mark the profile as deleting.
2. Revoke sessions and block further application access for that profile.
3. In a database transaction, delete inventory, postal-entity settings, pending and rejected proposals, and other private user-owned values.
4. Preserve approved and merged shared records while setting proposer and other direct user references to null.
5. Delete the SuperTokens user identity.
6. Retry any failed external step without recreating deleted private data.
7. Remove the profile and deletion job after every required deletion succeeds.

Database foreign keys must use deliberate deletion behavior. User-owned records cascade from the profile. Shared and audit records use nullable contributor references rather than cascading deletion. Do not put the user's email or authentication ID into preserved proposal payloads or audit text.

Tests:

- Delete the authentication identity, profile, postal-entity settings, inventory, and private proposal data.
- Preserve approved and merged definitions, schedules, conversions, and their source information.
- Remove the deleted user's identity from preserved shared data and moderation history.
- Leave every other user's data unchanged.
- Reject account access while deletion is incomplete.
- Retry an interrupted SuperTokens deletion without duplicating work or failing on already-removed records.

## Phase 12: Verification and deployment

Suggested commits:

```text
test: cover authenticated inventory workflows
chore: document production deployment
```

Automated verification:

- Unit tests for normalization, date resolution, and decimal calculations.
- API tests for authentication, ownership, validation, and moderation.
- Component tests for conditional face-value fields and quantity controls.
- Browser tests covering sign-in, required first-run settings, active-country switching, proposal submission, stamp creation, editing, and deletion.
- Browser tests covering a JSON data export with private, shared, and moderation records.
- Browser tests covering account-deletion confirmation and rejection of the deleted session.
- Two-account tests proving inventory and pending-proposal isolation.
- Migration test from an empty database.
- Keyboard and screen-reader checks across settings, inventory, proposals, moderation, export, and deletion.
- Localization checks for each displayed currency and calendar date.

Deployment work:

1. Configure production database credentials.
2. Configure SuperTokens production domains.
3. Replace demo social credentials with production Google and Apple applications.
4. Run migrations before serving the new application version.
5. Keep the existing CI jobs for lint, tests, build, and clean-database migration checks passing.
6. Test a preview deployment with a normal account and a moderator account.
7. Document backup, restore, and moderator-role assignment.

Release verification:

- All acceptance criteria in the PRD pass.
- The production database persists inventory across deployments.
- A moderator decision changes only the intended shared definition or schedule.
- A scheduled change activates according to each test user's local date.
- No inventory or pending proposal crosses user boundaries.
- The complete workflow passes the accessibility and localization audit.

## Recommended delivery order

Phases 1 and 2 establish reliable checks and repeatable database setup. Phases 3 through 5 establish profiles, shared data, and the valuation engine. Phase 6 adds user-owned inventory records before proposals need to reference them. Phases 7 through 9 add proposal moderation and the user interfaces. Phase 10 adds data export after every user-linked record type exists. Phase 11 adds account deletion after export is available. Phase 12 audits the complete authenticated flow and prepares deployment.

Do not add postage-combination logic, a planned-mailing-date selector, a postage-rate catalog, or collection valuation while implementing these phases. Those features require separate product requirements.
