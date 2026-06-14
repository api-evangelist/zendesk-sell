# Zendesk Sell GraphQL Schema

## Overview

This directory contains a conceptual GraphQL schema for the Zendesk Sell (formerly Base CRM) Sales CRM API. The schema is derived from the [Zendesk Sell REST API reference](https://developer.zendesk.com/api-reference/sales-crm/introduction/) and [Base CRM developer documentation](https://developers.getbase.com/docs/rest/reference).

Zendesk Sell does not natively expose a GraphQL endpoint; this schema is a faithful conceptual representation of the REST API surface intended to support schema-driven development, API gateway translation layers, and developer tooling.

## Source

- REST API host: `https://api.getbase.com`
- API reference: `https://developer.zendesk.com/api-reference/sales-crm/introduction/`
- Authentication: OAuth 2.0 (authorization code, implicit, password, and refresh token grants)
- GitHub organization: `https://github.com/zendesk`

## Schema File

- `zendesk-sell-schema.graphql` — Full GraphQL SDL with 55+ named types covering the complete Zendesk Sell domain.

## Type Inventory

### Core CRM

| Type | Description |
|------|-------------|
| `Contact` | A person or company that may become a customer |
| `Deal` | An opportunity to sell to a contact or organization |
| `Lead` | An unqualified sales prospect |
| `Organization` | A company or account associated with contacts and deals |
| `Address` | A reusable postal address value object |

### Activity & Communication

| Type | Description |
|------|-------------|
| `Task` | A to-do item associated with a CRM resource |
| `Note` | A note attached to a lead, contact, or deal |
| `Call` | A phone call logged against a CRM resource |
| `CallRecording` | Recording metadata for a logged call |
| `Email` | An email message tracked in the CRM |
| `Text` | A text/SMS message logged in the CRM |
| `Meeting` | A calendar meeting or appointment |

### Pipeline & Stages

| Type | Description |
|------|-------------|
| `Pipeline` | A sales pipeline grouping stages |
| `Stage` | A stage within a sales pipeline |

### Products & Orders

| Type | Description |
|------|-------------|
| `ProductCatalog` | A product catalog containing products for sale |
| `Product` | A product that can be attached to deals |
| `Order` | An order associated with a deal |
| `LineItem` | A line item within an order |

### Users & Teams

| Type | Description |
|------|-------------|
| `User` | A Zendesk Sell user account |
| `UserRole` | A role assigned to a user controlling access level |
| `Permission` | A specific permission granted to a user or role |
| `Team` | A team grouping users for territory or reporting |

### Customization

| Type | Description |
|------|-------------|
| `CustomField` | A custom field definition for a CRM resource |
| `CustomFieldOption` | An option value for dropdown custom fields |
| `CustomFieldValue` | A custom field value for a specific resource instance |
| `Tag` | A tag label used to categorize CRM resources |
| `Label` | A label (alias for Tag in some contexts) |
| `Source` | The original source of a lead or deal |

### Filtering & Search

| Type | Description |
|------|-------------|
| `Filter` | A saved filter for querying CRM resources |
| `FilterCondition` | A single condition clause within a saved filter |

### Sequences (Cadences)

| Type | Description |
|------|-------------|
| `Sequence` | An automated outreach sequence |
| `SequenceStep` | A step within an outreach sequence |
| `SequenceSubscription` | A contact or lead enrolled in a sequence |

### Signals & Prospecting

| Type | Description |
|------|-------------|
| `Signal` | A sales signal event surfaced by intelligence features |
| `SignalEntry` | A signal entry record |
| `SmartLink` | A smart link used for email tracking and engagement |
| `Enrichment` | A prospect enrichment result |
| `ProspectorResult` | A result from lead prospecting tools |
| `CreditUsage` | Tracks credit consumption for prospecting and enrichment |

### Visitor Intelligence

| Type | Description |
|------|-------------|
| `Prospect` | A tracked prospect who visited a monitored page |
| `ProspectVisit` | A single page visit by a tracked prospect |
| `ProspectSource` | A traffic source attribution for a prospect |
| `WhoIs` | WHOIS domain intelligence data |
| `VisitEvent` | A visit event from visitor intelligence tracking |
| `HeatMap` | Heat map analytics for a tracked page |

### Webhooks

| Type | Description |
|------|-------------|
| `Webhook` | A webhook subscription for real-time event delivery |
| `WebhookEvent` | A webhook event delivery record |

### API & Authentication

| Type | Description |
|------|-------------|
| `APICredential` | An OAuth application credential |
| `OAuthToken` | An OAuth access/refresh token pair |

### Integrations & Settings

| Type | Description |
|------|-------------|
| `Integration` | An external integration connected to Zendesk Sell |
| `Sync` | A sync record tracking data synchronization |
| `EmailSetting` | Email-specific settings for a user or account |
| `CallSetting` | Call-specific settings for a user or account |
| `CalendarSetting` | Calendar integration settings for a user or account |

### Associations & Collaboration

| Type | Description |
|------|-------------|
| `Association` | An association between two CRM resources |
| `Collaboration` | A collaboration record linking a user to a CRM resource |
| `Embed` | An embedded resource (file, link) attached to a CRM record |

### Pagination & Connections

| Type | Description |
|------|-------------|
| `PageInfo` | Cursor-based pagination metadata |
| `ContactConnection` / `ContactEdge` | Paginated contact list |
| `DealConnection` / `DealEdge` | Paginated deal list |
| `LeadConnection` / `LeadEdge` | Paginated lead list |

### Result Types

| Type | Description |
|------|-------------|
| `LeadConversionResult` | Result of converting a lead to a contact and/or deal |

## Enums

- `ContactType` — PERSON, COMPANY
- `DealStatus` — ONGOING, WON, LOST
- `LeadStatus` — NEW, WORKING, QUALIFIED, UNQUALIFIED, CONVERTED
- `TaskStatus` — ACTIVE, DONE
- `TaskResourceType` — LEAD, CONTACT, DEAL
- `CallDirection` — INBOUND, OUTBOUND
- `CallOutcome` — CONNECTED, LEFT_MESSAGE, LEFT_LIVE_MESSAGE, WRONG_NUMBER, NO_ANSWER, FAILED
- `AssociationResourceType` — LEAD, CONTACT, DEAL, ORDER
- `WebhookEventType` — POST, PUT, PATCH, DELETE
- `FilterType` — LEAD, CONTACT, DEAL
- `CustomFieldType` — TEXT, NUMBER, DECIMAL, DATE, DATETIME, CHECKBOX, DROPDOWN, URL, EMAIL, PHONE
- `SequenceStatus` — ACTIVE, PAUSED, COMPLETED, OPTED_OUT
- `IntegrationStatus` — ACTIVE, INACTIVE, ERROR
- `OwnerType` — USER, TEAM

## Authentication

The Zendesk Sell REST API uses OAuth 2.0. When building a GraphQL gateway over this API, implement the following flows:

- **Authorization Code** — for web applications
- **Implicit** — for single-page applications
- **Password** — for trusted clients
- **Refresh Token** — for renewing expired access tokens

Bearer tokens are passed in the `Authorization: Bearer <token>` header to `https://api.getbase.com`.

## Usage Notes

- This schema uses cursor-based pagination (`Connection` / `Edge` pattern) for list queries.
- `JSON` scalar is used for flexible structured data (custom field values, webhook payloads, intelligence data).
- `DateTime` scalar follows ISO 8601 format.
- Custom fields are defined at the account level per resource type and their values are represented as `CustomFieldValue` objects with a `JSON` value field.
