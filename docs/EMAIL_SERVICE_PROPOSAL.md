# Email Service Proposal

## Document status

- Scope: transactional email for the stamp inventory application
- Delivery model: application mail service backed by an SMTP or email API provider
- Current authentication: Google and Apple social login only; no account-linking interface
- Implementation status: proposal only; no mail transport, outbox, worker, or notification preferences exist
- Excluded from the initial implementation: email-and-password login, username-and-password login, operating a public SMTP server, marketing campaigns, and physical postal-mail features

## Recommendation

Add a provider-independent mail service using the existing profiles, user settings, and moderation events. Use a transactional email provider to deliver messages. Do not operate the SMTP infrastructure for the first release.

The application should own templates, notification rules, preferences, and delivery records. The provider should handle message transfer, domain authentication, bounce processing, complaint processing, and delivery reputation. This split lets the application change providers without rewriting product features.

Email must report application state, not control it. A failed email must not prevent a social login from being linked, a proposal from being moderated, an account from being deleted, or a scheduled value from taking effect. The application remains the source of truth.

Account linking is a separate product decision, as described in the
[deployment proposal](DEPLOYMENT_PROPOSAL.md#authentication-cost-comparison).
Login-linked and login-removed messages, and selection among multiple linked
addresses, depend on that feature being approved and implemented. The initial
email service can use the verified address of the current social identity; it
must define what happens when that identity supplies no verified address.

## Reasons to add email

### Security changes happen outside normal inventory use

A user may not notice that a Google or Apple login was linked or removed until the next sign-in. An email sent to the account's selected notification address gives the user an independent warning. The same applies when account deletion begins.

### It enables a future verified password login

The first release remains social-login-only. Once reliable transactional email exists, the application can later add a non-social login method using either an email address or a username plus password. Both variants must require a verified recovery email; email-less password accounts are out of scope.

The email service would deliver address-verification links, password-reset links, password-change notices, and recovery warnings. SuperTokens, rather than the delivery provider, would continue to hash and check passwords, manage reset-token state, and create sessions.

This future login method would need short-lived, single-use verification and reset tokens, request rate limits, responses that do not disclose whether an account exists, session revocation after sensitive credential changes, and explicit account-linking rules agreed before implementation. A password login must be deliberately linked to an existing signed-in account; a matching email address alone must not merge accounts.

### Moderation does not finish during the submitting session

A postal entity, named/code value, or fixed conversion can remain pending after the user leaves the application. Email can report approval, rejection, or merging without requiring the user to reopen the proposal list repeatedly.

### Future values have known dates

The application already models named/code changes that become valid on a future calendar date. An optional email can warn affected users before the change. The user's saved dashboard IANA timezone determines the relevant date, while the inventory screen remains authoritative for the current value.

### Moderator work can arrive irregularly

Moderators need not poll an empty queue. A digest can list new proposals and older proposals still awaiting review. A digest is preferable to one email per proposal when submission volume grows.

### Delivery history helps support work

A delivery record can answer whether the application attempted a notification, which template version it used, and whether the provider accepted, bounced, or rejected the message. It must not store OAuth tokens or unnecessary inventory details.

## Proposed messages

| Message | Recipient | Default | Trigger | Content boundary |
| --- | --- | --- | --- | --- |
| Login method linked | Account notification email | Required security notice | A Google or Apple identity is linked | Provider name, time, and a link to account settings; no provider token |
| Login method removed | Account notification email | Required security notice | A linked identity is removed after alternate-login confirmation | Removed provider, time, and support instructions |
| Account deletion started | Account notification email | Required security notice | Immediately before the deletion workflow blocks access | Time and warning that private data will be removed; do not attach an export |
| Proposal submitted | Proposer | Optional | A postal entity, named/code value, or conversion is submitted | Proposal type, country or currency pair, and proposal link |
| Proposal approved | Proposer | On | Moderator approval commits shared data | Result and shared record link |
| Proposal rejected | Proposer | On | Moderator rejection | Result, decision note, and proposal link |
| Proposal merged | Proposer | On | Moderator merges a duplicate | Canonical shared record and decision note |
| Upcoming named/code change | Users with matching inventory | Opt-in | An eligible change enters the notice window | Country, code, current value, future value, and effective date |
| Moderation digest | Moderators | Opt-in | Scheduled digest finds pending work | Counts and queue links, not full proposal payloads |
| Service or privacy notice | Affected users | Required only when necessary | An operator creates a scoped notice | The specific service, security, or policy change |

“On” means enabled initially but configurable. Security and required service notices do not use an unsubscribe control. Optional and ordinary transactional messages must have settings controls and an unsubscribe link where applicable.

## Notification address

The account needs one selected notification email. Initially, use a verified email exposed by the current Google or Apple identity. If account linking is added, let the user select among verified addresses from their linked identities. The application does not assume that matching provider emails represent the same person and does not send each message to every linked address.

If account linking is implemented, removing a login method is blocked when its email is the notification address until the user selects another verified linked address. Supporting an unrelated custom notification address can be added later with a separate verification flow.

## Delivery design

```text
Product transaction
       |
       v
Email outbox row in the same database transaction
       |
       v
Background delivery worker
       |
       v
Transactional email provider
       |
       v
Signed delivery webhook updates status
```

The outbox prevents a database change and its notification from becoming inconsistent. For example, proposal approval and its email job commit together. The worker can retry delivery without approving the proposal twice.

Suggested records:

```text
NotificationSettings
  userId
  notificationEmail
  proposalUpdatesEnabled
  upcomingValueEnabled
  moderatorDigestEnabled
  digestFrequency nullable
  createdAt
  updatedAt

EmailOutbox
  id
  userId nullable
  templateKey
  templateVersion
  recipient
  locale nullable
  payload
  idempotencyKey unique
  status
  attemptCount
  nextAttemptAt nullable
  createdAt
  sentAt nullable

EmailDeliveryEvent
  id
  outboxId
  providerMessageId
  eventType
  occurredAt
```

Store only fields required to render the selected template. Inventory lists, full proposal payloads, OAuth responses, and session data do not belong in the outbox.

## User settings

The settings page should provide:

- Verified notification email; selection among linked-login emails only if account linking is implemented.
- Proposal-status email preference.
- Upcoming named/code value preference.
- Moderator-digest preference for moderators.
- Digest frequency when digests are enabled.
- A record of recent security notifications and their delivery status.

Security notifications remain enabled. If the selected address bounces, the settings page shows that email delivery is unavailable and asks the user to choose another verified address.

## Scheduled value notifications

Upcoming-value email is useful only for users who own a matching named/code stamp in that country. It must follow these rules:

1. Use the active postal-entity setting's country and the user's saved dashboard IANA timezone.
2. Send at most once for a user, schedule value, and effective date.
3. Do not send before the value enters the product's notice window.
4. Include pending data only when the recipient is its proposer.
5. Recheck eligibility when the job runs because the proposal, inventory, or preference may have changed.
6. Do not calculate inventory totals from the email payload. Link to the current inventory instead.

The exact send day within the notice window can be selected when this notification is implemented. The current inventory release does not need this email to calculate or display scheduled values.

## Privacy, security, and deliverability

- Publish SPF and DKIM records for the sending domain and a DMARC policy appropriate to the rollout stage.
- Use a dedicated transactional subdomain so application mail is separate from personal or marketing mail.
- Verify provider webhooks and reject unsigned delivery events.
- Escape user-controlled text in templates. Do not render submitted HTML.
- Use short-lived, single-purpose links for any action that changes account state.
- Do not put authentication tokens, full inventory exports, or sensitive moderation payloads in email.
- Rate-limit user-triggered messages and deduplicate event-triggered messages.
- Maintain bounce and complaint suppression so the system stops repeated delivery to a failing address.
- Set a retention period for recipient addresses, outbox payloads, and delivery events.
- Include notification settings and user-linked delivery history in the JSON user-data export.
- On account deletion, remove queued user messages and private recipient data. Retained aggregate delivery records must not identify the deleted account.

## Why not operate the SMTP server

Running an internet-facing SMTP server would add DNS configuration, IP reputation, reverse DNS, queue management, bounce parsing, complaint handling, abuse monitoring, security patching, and deliverability investigation. A technically successful SMTP response does not guarantee inbox placement.

Those tasks do not improve stamp inventory or moderation. A delivery provider lets the project start with authenticated sending and observable delivery while retaining control of product rules in the application.

## Providers and costs

The prices below were reviewed on October 3, 2026. They are public list prices in US dollars, before taxes. Providers can change prices, included features, and volume bands, so the selected plan must be checked again before launch.

All of these providers can be called from the application through an HTTP API. Most also expose SMTP. None needs to run in the same deployment as the web application or database. The application-owned outbox belongs in the application database, which currently uses SQLite; migrate it too if the proposed PostgreSQL deployment is selected. The provider only handles delivery and delivery events.

| Provider | Entry price | How cost scales | Suitability for this project |
| --- | --- | --- | --- |
| [Resend](https://resend.com/pricing) | Free for 3,000 emails per month, limited to 100 per day. Pro is $20 per month for 50,000. | Pro overage is $0.90 per additional 1,000 emails. Scale is $90 per month for 100,000. | The simplest initial choice for the current Next.js stack. It has official Next.js examples, webhooks, and a low-friction API. The free daily limit may be reached during a moderation burst. |
| [Postmark](https://postmarkapp.com/pricing) | Free for 100 emails per month. Basic is $15 per month for 10,000. | Basic overage is $1.80 per additional 1,000. Higher plans reduce the overage rate and add longer retention or more domains. | A strong choice when transactional-email delivery, message history, and support matter more than the lowest price. |
| [Amazon SES](https://aws.amazon.com/ses/pricing/) | The Essentials plan has no monthly account fee and charges $0.16 per 1,000 outgoing emails for the first 10 million. New SES accounts start on this plan. | At that rate, 10,000 emails cost $1.60, 50,000 cost $8, and 100,000 cost $16, excluding attachment data and other AWS services. Pro adds $105 per account and Region each month plus $0.22 per 1,000. | The lowest listed delivery cost in this comparison. It requires more AWS configuration and operational knowledge for identities, permissions, event routing, suppression, and moving an account out of the SES sandbox. |
| [Mailgun](https://www.mailgun.com/pricing/) | Free for up to 100 emails per day. Basic is $15 per month for 10,000. | Basic extra emails start at $1.80 per 1,000. Foundation starts at $35 per month for 50,000; check the calculator for the intended volume. | Useful when both API and SMTP delivery, inbound routing, or Mailgun-specific tooling are wanted. It offers no clear advantage over Resend for the first release. |
| [Brevo](https://www.brevo.com/products/transactional-email/) | Free for 300 emails per day. Starter begins at $9 per month for 5,000 monthly emails. | Standard begins at $18 per month for 5,000. Professional begins at $499 per month for 150,000; intermediate volumes depend on the selected band. | The free daily allowance is generous. Its plans also include marketing and contact-management features that this application does not currently need. |

Resend documents both [Next.js integration examples](https://resend.com/docs/examples) and [signed delivery webhooks](https://resend.com/docs/api-reference/webhooks/create-webhook). Postmark explains how included volume and overage are billed in its [monthly pricing guide](https://postmarkapp.com/support/article/1107-how-does-monthly-pricing-work). Amazon describes the July 2026 SES plan structure in its [pricing-plan announcement](https://aws.amazon.com/blogs/messaging-and-targeting/introducing-amazon-simple-email-service-ses-pricing-plans/). Brevo lists its [plan prices and allowances](https://help.brevo.com/hc/en-us/articles/208589409-About-Brevo-s-pricing-plans) and documents its [transactional email API](https://developers.brevo.com/docs/send-a-transactional-email).

### Recommended starting choice

Use Resend for the first implementation. Its free plan is enough for development and a quiet launch, and its Pro plan gives a clear upgrade point when the application exceeds either 100 messages in a day or 3,000 in a month. Its API and webhook support fit the outbox design without tying product data to the provider.

Amazon SES is the cost-first alternative. Moving to it becomes worthwhile when message volume makes the price difference material or the team is ready to own the extra AWS setup. Because templates, preferences, message state, and provider message IDs remain behind the application service, this change does not require a user-data migration.

Postmark is the alternative if delivery tooling and retained message history are worth a higher per-message price. Brevo is reasonable if its daily free quota fits the traffic pattern. Mailgun should be reconsidered if inbound email or its SMTP tooling becomes a requirement.

The main cost driver is messages sent, not registered accounts. Monthly volume should be estimated from security events, moderation events, opted-in reminders, and digest frequency. A digest can combine several moderation events into one message, reducing both cost and notification noise.

Before selecting a production provider, compare its data-processing terms, available processing regions, retention controls, suppression-list behavior, domain-authentication support, webhook signing, and support response times. The lowest listed price alone is not enough for a security-sensitive email channel.

## Adoption plan

### Stage 1: service foundation

1. Select the transactional email provider and sending subdomain.
2. Configure SPF, DKIM, and DMARC.
3. Add the provider-independent mail interface, outbox, worker, signed webhook, and local test transport.
4. Add notification-email selection and preference controls.
5. Add delivery logs with payload redaction and retention.

### Stage 2: security notifications

1. Add login-linked and login-removed notices only after account linking is implemented.
2. Attempt the account-deletion-started notice before private data removal begins. Delivery is best effort: deletion must not wait for retries, and cleanup must remove any unsent notice and its recipient data.
3. Test failed delivery without rolling back or delaying the account action.

### Stage 3: moderation notifications

1. Send postal-entity, named/code, and fixed-conversion proposal result messages after approve, reject, and merge transactions.
2. Add optional submission confirmations.
3. Add moderator digests only when queue activity warrants them.

### Stage 4: scheduled-value notifications

1. Add the opt-in preference.
2. Select the send day within the existing notice window.
3. Query affected users by country, named/code reference, and current inventory.
4. Recheck eligibility at send time and enforce the idempotency key.

### Later option: verified password login

This is not part of the initial email-service implementation. If selected later:

1. Decide whether users sign in with their verified email address or a separate username.
2. Require a verified recovery email in either case.
3. Add the SuperTokens password recipe without changing the primary user identifier.
4. Add verification, reset, password-change, and recovery-warning templates.
5. Allow deliberate linking and removal under the agreed multiple-login-method rules.
6. Test enumeration resistance, token expiry and single use, rate limits, session revocation, export, and account deletion.

## Verification

- A local test transport captures messages without contacting an external provider.
- Each product event selects the expected template and recipient preference.
- Replaying an event does not create a second message with the same idempotency key.
- A provider failure retries the email without repeating the product transaction.
- A signed webhook updates delivery state; an unsigned webhook is rejected.
- Optional messages respect preferences and unsubscribe changes.
- Security messages are sent even when optional messages are disabled.
- A user never receives another user's inventory, proposal details, email address, or identifiers.
- The JSON account export contains the user's notification settings and linked delivery history without secrets.
- Account deletion removes queued messages and identifying delivery data for that user.

## Decision points before implementation

- Transactional email provider and hosting region.
- Sending domain and subdomain.
- Whether to implement account linking and fund its authentication costs.
- Notification-email behavior when the current identity exposes no verified email.
- Template languages for the first release.
- Retention periods for outbox payloads and delivery events.
- Send day and local send time for upcoming-value notices.
- Moderator digest frequency.
