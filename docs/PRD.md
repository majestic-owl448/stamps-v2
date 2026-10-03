# Stamp Inventory Product Requirements

## Document status

- Product area: stamp inventory
- Release scope: first usable inventory release
- Included systems: authentication, inventory, valuation, crowdsourced reference data, and moderation
- Excluded systems: postage planning, stamp combination selection, and collection-market valuation

## Product summary

The application lets an authenticated user record the stamps they own and calculate their current postage value for a selected postal entity. A postal entity identifies the issuing authority, scope, and country used for postage. Each saved postal-entity setting has a display currency. The user has one dashboard timezone for date-based valuation. A stamp belongs to one postal entity and country and can have a monetary face value, a country-specific name or code, or no face value. The user records how many copies they own, how many are annulled, and whether the stamp has expired.

Named and coded face values use shared, moderated data. Currency conversions that represent established fixed conversions can also become shared data. Users can propose additions and changes. A proposer can use a pending fixed conversion immediately and can use a pending named/code value when its effective date has arrived; other users see either only after approval.

## Product goals

The first release must let a user:

1. Sign in and access only their own inventory.
2. Select or create a postal entity and display currency before using inventory.
3. Save more than one postal-entity setting and select which postal entity is active.
4. Add a stamp identified by a postal entity, country, name, face value, and optional year of issue.
5. Record the owned and annulled quantities for that stamp.
6. Mark a stamp as expired.
7. Edit quantities and remove an inventory entry.
8. See current unit and total postage values in the active postal-entity setting's display currency.
9. See stamps from other countries with postage value zero while they are not active.
10. Use country-specific named or coded face values.
11. Propose postal entities, shared named/code values, scheduled changes, and fixed currency conversions.
12. See an upcoming named/code value during the 10 calendar days before it takes effect.
13. Download a JSON copy of their user-owned and user-linked data.
14. Delete their account and all user-owned data without deleting contributions that have become shared resources.

## Out of scope

The first release will not:

- Find combinations of stamps for a letter or parcel.
- Ask for a planned mailing date.
- Model postal products, weights, dimensions, destinations, or service levels.
- Maintain a user-facing history of old named/code values.
- Track purchase price, sale price, catalog value, condition grade, or collection value.
- Research or pre-populate stamp catalogs.
- Fetch live foreign-exchange rates.
- Mark stamps as annulled when a user consumes them in the application.

An annulled stamp is a stamp that was already cancelled. It remains owned and visible in the collection, but its postage value is zero.

## Terms

| Term | Meaning |
| --- | --- |
| Postal entity | A moderated record for an issuing authority and geographic or office scope in one country. |
| Postal-entity setting | User-owned configuration that pairs a postal entity with its display currency. |
| Active postal entity | Postal-entity setting used for the current inventory valuation. Stamps from other countries have postage value zero. |
| Display currency | Currency used to display postage values and the inventory total for one postal-entity setting. |
| Face value | Monetary amount, country-specific name/code, or absent denomination shown on a stamp. |
| Named face value | A denomination identified by country and a normalized name or code. |
| Value schedule | Shared source of the current value and approved or proposed future changes. |
| Annulled quantity | Owned copies that have been cancelled and cannot contribute postage value. |
| Expired stamp | Stamp identity for which every owned copy has postage value zero. |
| Fixed conversion | Established conversion between currencies, such as a retired currency and its replacement. |
| Proposal | User-submitted addition or change awaiting moderator action. |
| Shared contribution | An approved or merged definition, schedule value, conversion, or future postage-rate change that is no longer owned by its proposer. |

## Roles

### User

A user manages their profile and inventory, reads approved shared data, and submits proposals. They can use their own eligible pending proposal while it awaits moderation.

### Moderator

A moderator reviews proposals, checks the submitted source, merges duplicates, and approves or rejects changes. Moderator actions are recorded with the actor and time.

## Minimum user settings

The authenticated dashboard is the inventory entry point. A new user can view its empty state before completing postal-entity setup. It explains, "Before you can start adding stamps, choose or create at least one postal entity." When approved entities are available, the user chooses one or selects Create; the creation form is open by default only when no entity is available. Selection and creation remain part of the stamp workflow because every stamp belongs to a postal entity.

Authenticated navigation links the dashboard and the personal settings page, with the current page identified. Personal settings contains dashboard timezone preferences, account-data export, and account deletion. It does not contain postal-entity selection or creation. Sign-out remains in authenticated navigation.

Before using the inventory, a newly authenticated user must save one postal-entity setting containing:

- Postal entity name.
- ISO 3166-1 alpha-2 country code.
- Issuing authority.
- Geographic or office scope.
- Source URL or source note.
- ISO 4217 display-currency code.

A user can add an approved postal entity or submit another entity for moderation. Each postal-entity setting has its own display currency. The personal Settings page uses the browser timezone by default and lets the user save another valid IANA timezone. The saved timezone is used for server-side date calculations.

One postal-entity setting is active at a time. The first setting becomes active automatically. The user can add another postal entity, edit a setting's display currency, and switch the active postal entity. A user cannot access inventory valuation until at least one usable setting exists.

## Stamp identity and ownership

An inventory entry represents copies of the same stamp identity owned by one user.

Required and optional fields:

| Field | Requirement |
| --- | --- |
| Postal entity | Required reference to a postal entity available to the user. |
| Country | Required ISO 3166-1 alpha-2 country code. |
| Name | Required user-facing identifier. |
| Year of issue | Optional integer. |
| Face value type | `MONETARY`, `NAMED`, or `NONE`. |
| Monetary amount and currency | Required when the type is `MONETARY`. |
| Named face value reference | Required when the type is `NAMED`. |
| Manual postage amount and currency | Required when the value cannot be resolved from a face value. Zero is valid. |
| Quantity owned | Integer greater than zero. |
| Quantity annulled | Integer from zero through quantity owned. |
| Expired | Boolean that applies to every owned copy in the entry. |

The application must preserve the face value when a stamp is annulled or expired. Postal validity changes the calculated postage value, not the recorded stamp identity.

## Postage valuation

### Unit value

The application first compares the stamp country with the active postal entity's country. If they differ, the unit postage value is zero with the reason `OUTSIDE_ACTIVE_COUNTRY`. The stamp remains owned and visible.

For a stamp in the active postal entity's country, the application resolves the unit postage value for a non-expired, non-annulled copy in this order:

1. A monetary face value in the active postal-entity setting's display currency has the same postage value as its face amount.
2. A monetary face value in another currency uses the applicable fixed currency conversion.
3. A named/code face value uses its applicable value schedule.
4. A face value without an applicable conversion or named/code value requires a manual postage amount.
5. A stamp with no face value requires a manual postage amount.

Money calculations must use decimal arithmetic. JavaScript floating-point numbers must not be used for stored values or multiplication.

A manual postage amount retains the currency in which the user entered it. If the user changes that postal-entity setting's display currency, the application converts the amount when an applicable conversion exists. When no conversion exists, the entry remains stored but cannot contribute to the active postal-entity total until the user supplies a conversion or replaces the manual amount.

### Quantity calculation

```text
usable quantity = expired ? 0 : quantity owned - quantity annulled

total postage value = usable quantity * current unit postage value
```

Rules:

- Annulled copies remain included in quantity owned.
- Every annulled copy has postage value zero.
- Every copy of an expired stamp has postage value zero.
- Every stamp outside the active postal entity's country has postage value zero, including stamps from a country that uses the same currency.
- Postage value can be zero but cannot be negative.
- The inventory total is the sum of each entry's total postage value in the active postal-entity setting's display currency.

## Named/code face values

A named/code face value is globally identified by:

```text
country code + normalized name/code
```

The database must preserve a display value separately from the normalized lookup value. The pair `(countryCode, normalizedCode)` must be unique among approved definitions.

Examples supplied during product definition include an Italian face value named `B Zona 1`. The application does not assume or seed its amount without submitted data.

Each named/code definition references a value schedule. The monetary amount is not stored directly on the definition.

## Shared value schedules

A value schedule stores:

- Country
- Currency
- Current baseline value
- Approved future values
- Pending values visible to their proposers
- Effective calendar date for each future value

This separation prepares the data for a later postage-rate feature:

```text
NamedFaceValue -----> ValueSchedule <----- Future PostageRate
```

When a named/code denomination and a postage rate are formally linked, both reference the same schedule. An update to that schedule changes them together. Values that happen to have the same amount but are not formally linked use separate schedules.

The inventory release does not need a postage-rate catalog or postage-rate user interface. A future `PostageRate` record can reference an existing schedule without changing stamp inventory records.

## Current and future named/code values

An effective date is stored as a calendar date, not a universal activation timestamp. Applicability is calculated from the user's dashboard IANA timezone.

At a single instant, a future value can be current for a user whose local date has advanced and still be upcoming for another user. For that reason, `CURRENT` and `SCHEDULED` are derived states rather than persistent global states.

For a user and local date, the resolver selects the latest eligible value whose effective date is absent or on or before that date. Eligible values are:

- Approved global values.
- Pending values proposed by the current user.

During the 10 calendar days before the next eligible value takes effect, the inventory shows:

- Current amount.
- Upcoming amount.
- Upcoming effective date.

Inventory totals use the current amount only. The release does not provide a postage-date selector.

After a new value applies in every supported timezone, the former amount can leave active valuation data. Moderator audit records remain available to moderators, but users do not receive a historical-rate feature.

### Schedule example

The product discussion supplied this acceptance example; it is not a request to seed a stamp catalog:

- Country: Italy (`IT`).
- Named/code face value: `B Zona 1`.
- Schedule currency: `EUR`.
- Current value: `1.35`.
- Approved future value: `1.40`.
- Effective date: October 1, 2028.

Through September 30, 2028 in the user's dashboard timezone, inventory calculations use `1.35 EUR`. From October 1, they use `1.40 EUR`. During the 10-day notice window, the interface shows both values and the effective date. A future postage-rate record for `B Zona 1` can reference the same schedule and change at the same time.

## Currency conversions

The application distinguishes fixed currency conversions from live market exchange rates. A verified fixed conversion is global because its mathematical rate does not vary by user.

A conversion contains:

- Source currency.
- Target currency.
- Exact decimal multiplier.
- Proposal and moderation data.

Users must be able to enter a missing conversion for their inventory. That user-owned value works immediately. A moderator can approve it as a shared fixed conversion after checking its source. Existing stamps continue to resolve through the same conversion reference after approval.

Whether a stamp denominated in a former currency remains valid is separate from the conversion. The stamp's expired field controls postal validity.

## Crowdsourcing and moderation

Users can propose:

- A new named/code face value.
- A correction to its country, display name, or normalized code.
- A current value.
- A future value and effective date.
- A missing fixed currency conversion.
- A correction to a fixed conversion.
- A postal entity with its issuing authority, scope, and source.

Every proposal records:

- Proposer.
- Submitted values.
- Source URL or source note.
- Submission time.
- Status: `PENDING`, `APPROVED`, `REJECTED`, or `MERGED`.
- Moderator, decision time, and decision note when reviewed.

Pending proposal rules:

- The proposer can use the proposed value when its effective date has arrived.
- A future pending value is displayed during its notice window but does not apply early.
- Other users cannot use or see pending data.
- Approval makes the value global without changing the proposer's stamp records.
- A duplicate proposal can be merged into an existing definition or scheduled value.
- A proposal cannot directly overwrite approved data.
- Rejection never makes the submission global. Linked private records stop using it and require the proposer to resubmit corrected data or select an eligible replacement; the application does not choose an approved or manual fallback automatically.

Moderator tools must support a queue, proposal detail, approve, reject, and merge. The approve operation must be transactional so the proposal and shared data cannot disagree.

## Authentication and data isolation

- Google and Apple login continue through SuperTokens.
- The database profile is keyed by the stable SuperTokens primary user ID.
- All inventory reads and writes derive the owner ID from the server-side session.
- A client cannot select or override the owner ID.
- Requests for another user's inventory record return `404`.
- Moderator endpoints require a server-verified moderator role.
- Pending proposals are filtered by proposer ID outside moderator views.

## User data export

An authenticated user can download one JSON file containing the data associated with their account. The export contains:

- Export schema version and generation time.
- SuperTokens account metadata available to the application, excluding tokens and provider secrets.
- Application profile and postal-entity settings.
- Stamp inventory entries with stored face values, quantities, and status fields.
- Pending, rejected, approved, and merged proposals submitted by the user.
- Private valuation data owned by the user.
- Shared definitions, schedule values, conversions, and future postage-rate records linked to the user as contributor.
- Moderation and audit entries linked to the user as proposer, affected account, or moderator.

The export includes stored decimal amounts and date-only values as strings so JSON parsing does not change them. References between exported records use their stored identifiers.

The export does not contain session tokens, OAuth access or refresh tokens, provider client secrets, API keys, password hashes, or another user's private profile fields. When a user-linked moderation entry also refers to another account, the entry remains understandable but the other account's private identifier and email are omitted.

Every new table or field that stores a direct user reference must be added to the export or marked as a secret excluded by an automated test. A user can download the file before deleting the account; deletion does not retain a private export for later retrieval.

## Account deletion

An authenticated user can request permanent account deletion from settings. The action requires explicit confirmation and revokes the user's active sessions.

Deletion removes user-owned data:

- SuperTokens identity and sessions.
- Application profile and postal-entity settings.
- Stamp inventory entries.
- Pending and rejected proposals.
- Private conversion or named/code values that never became shared resources.

Approved and merged contributions remain because other inventories can depend on them. This includes shared named/code definitions, value schedules, fixed conversions, and postage-rate data added in a later release. Preserved records must not retain the deleted user's ID, email, or another direct account reference. Moderation history required to explain a shared-data change remains with the user identity removed.

The deletion workflow must be idempotent. If deletion of the external SuperTokens identity fails after application data cleanup starts, the account remains blocked and the system retries the incomplete step. A partial failure must not restore access to data scheduled for deletion or remove shared contributions.

## Inventory interface

The authenticated inventory page contains:

- Active postal-entity selector.
- Postal-entity and display-currency settings.
- Add-stamp form.
- Inventory list.
- Quantity owned and annulled controls.
- Expired control.
- Unit postage value and total postage value.
- Overall inventory postage total.
- Edit and remove actions.

The personal Settings page contains the dashboard timezone preference, JSON
data-download action, and account-deletion action with confirmation.

The face-value input changes with the selected type:

- `MONETARY`: amount and currency.
- `NAMED`: name/code search within the stamp country, with an option to propose a missing definition.
- `NONE`: manual postage value.

If a required conversion is missing, the user can enter one without leaving the stamp workflow. If a named/code value has an upcoming change within the notice window, the stamp row displays both amounts and the effective date.

## Validation and error handling

- Quantity owned must be a positive integer.
- Quantity annulled must be an integer between zero and quantity owned.
- Every stamp must have a valid country code.
- Every display currency must have a supported ISO 4217 code.
- Year of issue is optional and must be a plausible four-digit integer when supplied. Exact accepted bounds belong in the implementation specification.
- Monetary amounts and conversion multipliers must be valid decimal strings.
- Postage amounts cannot be negative.
- Face-value fields must match the selected face-value type.
- A named/code reference must belong to the stamp country.
- Effective dates are calendar dates.
- Timezones must be valid IANA timezone identifiers.
- Manual postage values retain their entered currency.
- Validation errors identify the affected field.

## Accessibility and localization

- Forms and controls must be keyboard accessible.
- Every input has a visible label and associated error text.
- Status is not communicated through color alone.
- Monetary values use `Intl.NumberFormat` with the applicable currency.
- Dates are displayed in the user's locale while stored in an unambiguous date format.
- The interface distinguishes `owned`, `annulled`, and `expired` without using `used` as a synonym.

## Acceptance criteria

The inventory release is complete when:

1. A signed-in user can create, edit, list, and remove only their own inventory entries.
2. A new user must select or create a postal entity and display currency before using inventory.
3. A user can save more than one postal-entity setting and switch the active postal entity.
4. Every stamp is assigned to one postal entity and country.
5. Monetary, named/code, and absent face values are supported.
6. The user can set owned and annulled quantities, subject to the quantity constraints.
7. Annulled and expired copies always contribute zero postage value.
8. A stamp outside the active postal entity's country remains visible but contributes zero postage value.
9. Switching the active postal entity recalculates all entries without changing their stored postal entity, country, or face value.
10. A monetary face value in the active postal-entity setting's display currency is valued at face amount.
11. A converted monetary face value uses the applicable fixed decimal rate.
12. A named/code face value uses the current eligible value from its shared schedule.
13. Updating an approved fixed conversion recalculates every linked monetary stamp.
14. Updating an approved named/code schedule recalculates every linked named/code stamp.
15. A future value activates on its effective calendar date in the user's dashboard timezone.
16. Current and upcoming named/code values appear during the 10-day notice window.
17. A proposer can use their eligible pending data while other users cannot see it.
18. A moderator can approve, reject, and merge proposals through protected controls.
19. Inventory totals use exact decimal calculations and the active postal-entity setting's display currency.
20. Lint, automated tests, production build, database migrations, and an authenticated browser flow pass.
21. Changing a postal-entity setting's display currency never reinterprets a stored manual amount as a different currency.
22. A user can download valid JSON containing all user-owned records and all non-secret records linked to their account, including shared contributions and moderation history.
23. Account deletion removes the authentication identity and all user-owned records while approved or merged shared contributions remain without a contributor identity.
