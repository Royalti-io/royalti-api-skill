# Royalti.io API Reference

Developer reference for integrating with the [Royalti.io](https://royalti.io) REST API — the royalty management platform for music labels, distributors, and publishers.

**Base URL:** `https://api.royalti.io`
**Current Version:** 2.6.4
**Architecture:** Multi-tenant (workspace-scoped)

---

## Table of Contents

- [Authentication](#authentication)
- [Request Patterns](#request-patterns)
- [Response Patterns](#response-patterns)
- [Core Resources](#core-resources)
- [Royalties & Analytics](#royalties--analytics)
- [Webhooks](#webhooks)
- [WebSocket Events](#websocket-events)
- [Data Models](#data-models)
- [Integration Patterns](#integration-patterns)

---

## Authentication

### Token Types

| Token | Prefix | Expiry | Use Case |
|-------|--------|--------|----------|
| JWT Access Token | — | 6 hours | All API requests |
| JWT Refresh Token | — | 1 day | Obtain new access tokens |
| Workspace API Key | `RWAK` | Never (revocable) | Programmatic workspace access |
| User API Key | `RUAK` | Never (revocable) | Programmatic user access |

### Two-Step JWT Login

```bash
# Step 1: Login — returns refresh token + workspace list
curl -X POST https://api.royalti.io/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "user@example.com", "password": "secret" }'
```

Response:

```json
{
  "refresh_token": "eyJ...",
  "workspaces": [
    { "id": "ws_abc", "name": "My Label" }
  ]
}
```

```bash
# Step 2: Exchange refresh token for access token (scoped to a workspace)
curl https://api.royalti.io/auth/authtoken?currentWorkspace=ws_abc \
  -H "Authorization: Bearer <refresh_token>"
```

Response:

```json
{
  "data": {
    "access_token": "eyJ..."
  }
}
```

### Using the Access Token

All subsequent requests include the access token:

```bash
curl https://api.royalti.io/artist/ \
  -H "Authorization: Bearer <access_token>"
```

### API Key Authentication

API keys can be used instead of JWT tokens:

```bash
# Workspace API key
curl https://api.royalti.io/artist/ \
  -H "Authorization: Bearer RWAK_abc123..."

# User API key
curl https://api.royalti.io/asset \
  -H "Authorization: Bearer RUAK_def456..."
```

### Other Auth Methods

| Endpoint | Purpose |
|----------|---------|
| `POST /auth/loginlink` | Magic link login |
| `POST /auth/forgotpassword` | Request password reset |
| `PATCH /auth/resetpassword?code=CODE` | Apply password reset |
| `GET /auth/google` | Google OAuth |
| `GET /auth/linkedin` | LinkedIn OAuth |
| `GET /auth/facebook` | Facebook OAuth |

### Rate Limiting

- **Login endpoint:** 20 requests per 3 minutes per IP
- **Other endpoints:** No documented limits (subject to fair use)

### RBAC Roles

Ascending privilege:

```
guest < user < admin < owner < super_admin < main_super_admin
```

---

## Request Patterns

### Standard Headers

```
Authorization: Bearer <access_token | API_KEY>
Content-Type: application/json
```

### Pagination

All list endpoints support pagination:

| Param | Default | Description |
|-------|---------|-------------|
| `page` | `1` | Page number |
| `size` | `10`–`20` | Items per page (max `100`) |
| `sort` | `updatedAt` | Sort field |
| `order` | `desc` | `asc` or `desc` |

```bash
GET /artist/?page=2&size=25&sort=artistName&order=asc
```

### Filtering (Analytics Endpoints)

| Param | Type | Description |
|-------|------|-------------|
| `start` | `YYYY-MM-DD` | Date range start |
| `end` | `YYYY-MM-DD` | Date range end |
| `dsp` | CSV string | Filter by DSP/platform |
| `country` | CSV string | Filter by territory (ISO 3166-1 alpha-2) |
| `artists` | CSV string | Filter by artist IDs |
| `upc` | CSV string | Filter by UPC |
| `isrc` | CSV string | Filter by ISRC |
| `aggregator` | CSV string | Filter by distributor |
| `type` | string | Filter by sale type |
| `periodFilterType` | `accounting` \| `sale` | Date filtering mode |
| `includePreviousPeriod` | boolean | Include comparison data |
| `table_name` | string | Filter by royalty file source table |

### Search

Many list endpoints support text-based search:

```bash
GET /artist/?search=drake&page=1&size=10
```

### Bulk Operations

Most resources support bulk create and delete:

```bash
POST /artist/bulk          # Create multiple artists
DELETE /artist/bulk/delete  # Delete multiple artists
POST /asset/bulksplits     # Assign splits to multiple assets
```

---

## Response Patterns

### Success

```json
{
  "status": "success",
  "message": "Operation successful",
  "data": {}
}
```

### Paginated List

```json
{
  "status": "success",
  "data": [],
  "totalItems": 142,
  "totalPages": 15,
  "currentPage": 1
}
```

### Summary (v2.6.4+)

Available on `/artist/summary`, `/asset/summary`, `/product/summary`, `/user/summary`, `/split/summary`, `/payment/summary`, `/expense/summary`, `/revenue/summary`:

```json
{
  "message": "Summary retrieved successfully",
  "summary": {
    "total": 250,
    "byStatus": { "active": 200, "inactive": 50 },
    "byFormat": { "Single": 120, "Album": 80, "EP": 50 },
    "byType": { "Audio": 230, "Video": 20 },
    "revenue": { "total": 15000.50, "currency": "USD" }
  }
}
```

### Error

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": ["optional", "array", "of", "details"]
  }
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `201` | Created |
| `400` | Bad request / validation error |
| `401` | Unauthorized (missing or invalid token) |
| `403` | Forbidden (insufficient role/permissions) |
| `404` | Resource not found |
| `409` | Conflict (duplicate, already exists) |
| `422` | Unprocessable entity |
| `429` | Rate limited (login endpoint) |
| `500` | Server error |

---

## Core Resources

All resources follow the same CRUD pattern and are workspace-scoped (multi-tenant).

### Standard CRUD Pattern

```
GET    /{resource}/              # List (paginated)
POST   /{resource}/              # Create
GET    /{resource}/{id}          # Get by ID
PUT    /{resource}/{id}          # Update
DELETE /{resource}/{id}          # Delete
GET    /{resource}/summary       # Summary stats (v2.6.4+)
GET    /{resource}/download/csv  # CSV export
POST   /{resource}/bulk          # Bulk create
DELETE /{resource}/bulk/delete   # Bulk delete
```

### Users — `/user/`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /user/{id}/stats` | User royalty statistics |
| `GET /user/{id}/monthly` | Monthly breakdown |
| `GET /user/{id}/artists` | User's artists |
| `GET /user/{id}/assets` | User's assets |
| `GET /user/{id}/products` | User's products |
| `GET /user/{id}/aut` | User accounting data |
| `GET /user/invites` | Pending invitations |
| `POST /user/invites/{id}/resend` | Resend invitation |
| `POST /user/invites/{id}/cancel` | Cancel invitation |

### Artists — `/artist/`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /artist/{id}/assets` | Artist's tracks |
| `GET /artist/{id}/products` | Artist's releases |
| `GET /artist/{id}/splits` | Artist's splits |
| `GET /artist/{id}/splits/{type}` | Splits by type |
| `GET /artist/{id}/stats` | Artist analytics |
| `POST /artist/{id}/merge` | Merge duplicate artists |
| `POST /artist/bulksplit` | Bulk assign splits |

### Assets (Tracks) — `/asset`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /asset/{id}/artists` | Track's artists |
| `GET /asset/{id}/stats` | Track analytics |
| `POST /asset/{id}/setdefaultsplit` | Set default split |
| `GET /asset/{id}/media` | Track media files |
| `DELETE /asset/{id}/media/{mediaName}` | Delete media |
| `GET /asset/{id}/ddex-metadata` | DDEX metadata |
| `GET /asset/{id}/ddex-readiness` | DDEX readiness check |
| `GET /asset/{assetId}/works` | Musical works (ISWC) |
| `POST /asset/bulksplits` | Bulk assign splits |
| `DELETE /asset/bulk/deletesplit` | Bulk remove splits |
| `POST /asset/bulk/defaultsplit` | Bulk set default splits |

### Products (Releases) — `/product/`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /product/{id}/artists` | Release's artists |
| `GET /product/{id}/assets` | Release's tracks |
| `GET /product/{id}/stats` | Release analytics |
| `POST /product/{id}/setdefaultsplit` | Set default split |
| `GET /product/{id}/media` | Release artwork/media |
| `GET /product/{id}/delivery` | Delivery info |
| `GET /product/{id}/delivery/status` | Delivery status |
| `POST /product/batch-delivery` | Batch deliver releases |
| `GET /product/{id}/deliveries` | Delivery history |
| `POST /product/{id}/deliveries/{deliveryId}/retry` | Retry failed delivery |
| `GET /product/delivery-providers` | Available distributors |
| `GET /product/download/metadata` | Metadata export |

### Splits — `/split/`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `POST /split/default` | Create split from artist default |
| `POST /split/match` | Find splits matching criteria |
| `DELETE /split/bulk/catalog-splits` | Remove catalog-level splits |

Split types: `simple`, `conditional`, `temporal`

Example conditional split:

```json
{
  "entity_type": "artist",
  "entity_id": "uuid",
  "split_type": "conditional",
  "effective_date": "2025-01-01",
  "shares": [
    { "user": "user-uuid-1", "percentage": 60 },
    { "user": "user-uuid-2", "percentage": 40 }
  ],
  "conditions": {
    "territories": ["US", "GB", "CA"],
    "mode": "include",
    "sources": ["Spotify", "Apple Music"],
    "period_start": "2025-01-01",
    "period_end": "2025-12-31",
    "memo": "North America streaming deal"
  }
}
```

> Shares must sum to 100. `entity_type` is `artist`, `product`, or `asset`. Conditions are optional (omit for simple splits).

### Accounting — `/accounting/`

| Endpoint | Description |
|----------|-------------|
| `GET /accounting/{id}/stats` | User accounting stats |
| `GET /accounting/transactions` | Transaction list |
| `GET /accounting/transactions/summary` | Transaction summary |
| `GET /accounting/transactions/monthly` | Monthly transaction breakdown |
| `GET /accounting/getcurrentdue` | Current amount due per user |
| `GET /accounting/gettotaldue` | Total outstanding across workspace |
| `POST /accounting/refresh` | Recalculate accounting |
| `POST /accounting/refreshstats` | Refresh stats cache |
| `POST /accounting/users/{id}/recalculate` | Recalculate single user |
| `POST /accounting/tenant/recalculate` | Recalculate entire workspace |
| `GET /accounting/queue/status` | Processing queue status |

### Payments — `/payment/`

Standard CRUD (supports both JSON and `multipart/form-data` for receipt attachments):

| Endpoint | Description |
|----------|-------------|
| `GET /payment/summary` | Payment totals |
| `POST /payment/bulk` | Bulk create payments |

### Payment Requests — `/payment-request/`

| Endpoint | Description |
|----------|-------------|
| `GET /payment-request/` | List requests |
| `GET /payment-request/{id}` | Get request |
| `POST /payment-request/{id}/approve` | Approve request |
| `POST /payment-request/{id}/decline` | Decline request |

### Royalty Files — `/file/`

| Endpoint | Description |
|----------|-------------|
| `GET /file/royalty` | List uploaded royalty files |
| `GET /file/royalty/{id}` | Get file details |
| `DELETE /file/royalty/{id}` | Delete file |
| `POST /file/createroyalty` | Upload royalty file |
| `GET /file/upload-url` | Get presigned upload URL |
| `POST /file/confirm-upload-completion` | Confirm upload finished |
| `GET /file/detection/{sessionId}` | Poll auto-detection status |
| `POST /file/confirm-detection/{sessionId}` | Confirm detected format |
| `GET /file/auto-processing-config` | Auto-processing settings |
| `POST /file/enable-auto-processing` | Enable auto-processing |
| `GET /file/sources` | List royalty sources (DSPs) |
| `GET /file/{id}/download` | Download file |
| `GET /file/processing/{jobId}` | Processing job status |

### Labels — `/labels/`

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /labels/hierarchy` | Label tree structure |

### Releases — `/releases`

Full release lifecycle:

| Endpoint | Description |
|----------|-------------|
| `POST /releases` | Create release (draft) |
| `POST /releases/{id}/submit` | Submit for review |
| `POST /releases/{id}/review` | Start review |
| `POST /releases/{id}/approve` | Approve release |
| `POST /releases/{id}/reject` | Reject release |
| `POST /releases/{id}/tracks` | Add track to release |
| `GET /releases/{id}/tracks` | Get release tracklist |
| `GET /releases/{id}/stats` | Release statistics |

### Expenses & Revenue

Both follow standard CRUD at `/expense/` and `/revenue/` with summary endpoints.

### Notifications — `/notifications/`

| Endpoint | Description |
|----------|-------------|
| `GET /notifications/` | List notifications |
| `PATCH /notifications/{id}/read` | Mark as read |
| `PATCH /notifications/read-all` | Mark all as read |
| `GET /notifications/unread-count` | Unread count |

### Downloads / Reports — `/download/`

| Endpoint | Description |
|----------|-------------|
| `POST /download/generate` | Generate report download |
| `GET /download/{id}/status` | Check generation status |
| `GET /download/list` | List available downloads |

---

## Royalties & Analytics

Read-only analytics endpoints at `/royalty/`. All support the standard [filter params](#filtering-analytics-endpoints).

| Endpoint | Description |
|----------|-------------|
| `GET /royalty/` | Main summary |
| `GET /royalty/month` | Monthly trends |
| `GET /royalty/dsp` | By DSP/platform |
| `GET /royalty/country` | By country/territory |
| `GET /royalty/artist` | By artist |
| `GET /royalty/product` | By product/release |
| `GET /royalty/asset` | By track |
| `GET /royalty/tables` | By data source |
| `GET /royalty/saletype` | By sale type (stream, download, etc.) |
| `GET /royalty/aggregator` | By distributor |
| `GET /royalty/accountingperiod` | By accounting period |

### Analytics Response Shape

```json
{
  "Downloads": 1500,
  "Streams": 250000,
  "Royalty": 1234.56,
  "Count": 251500,
  "RatePer1K": 4.91,
  "RoyaltyPercentage": 45.2,
  "CountPercentage": 38.7,
  "PreviousRoyalty": 1100.00,
  "PreviousCount": 220000
}
```

> `PreviousRoyalty` and `PreviousCount` are only present when `includePreviousPeriod=true`.

---

## Webhooks

### Outbound Webhooks (Royalti to Your Endpoint)

Configure your webhook endpoint via tenant settings:

```bash
# Set webhook URL
PUT /tenant/settings/webhook-url
Body: { "webhookUrl": "https://yourapp.com/webhooks/royalti" }

# Enable/disable webhooks
PUT /tenant/settings/webhook-isActive
Body: { "isActive": true }

# Subscribe to event types
PUT /tenant/settings/webhook-enabledEvents
Body: { "enabledEvents": ["PAYMENT_COMPLETED", "ROYALTY_FILE_PROCESSED"] }
```

### Event Categories

**Financial:**
`PAYMENT_COMPLETED` | `PAYMENT_PROCESSING` | `PAYMENT_MADE_FAILED` | `PAYMENT_REQUEST_SENT` | `PAYMENT_REQUEST_APPROVED` | `PAYMENT_REQUEST_REJECTED` | `PAYMENT_DELETED` | `REVENUE_CREATED` | `REVENUE_UPDATED` | `REVENUE_DELETED` | `EXPENSE_CREATED` | `EXPENSE_UPDATED` | `EXPENSE_DELETED`

**Catalog:**
`ASSET_CREATED` | `ASSET_UPDATED` | `ASSET_DELETED` | `PRODUCT_CREATED` | `PRODUCT_UPDATED` | `PRODUCT_DELETED`

**Roster:**
`USER_CREATED` | `USER_UPDATED` | `USER_DELETED` | `USER_INVITATION_SENT` | `ARTIST_CREATED` | `ARTIST_UPDATED` | `ARTIST_DELETED`

**Royalty (opt-in):**
`ROYALTY_FILE_UPLOADED` | `ROYALTY_FILE_PROCESSED` | `ROYALTY_FILE_PROCESSING_FAILED`

**Split:**
`USER_ADDED_TO_SPLIT` | `USER_REMOVED_FROM_SPLIT`

### Webhook Payload

```json
{
  "id": "whd_{uuid}",
  "event": "PAYMENT_COMPLETED",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "version": "1.0",
  "tenant": {
    "id": "tenant-uuid",
    "name": "My Label",
    "domain": "mylabel.royalti.io"
  },
  "source": {
    "service": "royalti-api",
    "environment": "production",
    "traceId": "trace-uuid"
  },
  "data": {
    "event": {
      "id": "evt-uuid",
      "type": "PAYMENT_COMPLETED",
      "category": "financial",
      "timestamp": "2025-01-15T10:30:00.000Z",
      "importance": "high"
    },
    "actor": {
      "id": "user-uuid",
      "type": "user",
      "name": "John Doe",
      "email": "john@label.com"
    },
    "resource": {
      "type": "payment",
      "id": "payment-uuid",
      "url": "https://app.royalti.io/payments/payment-uuid",
      "displayName": "Payment #1234"
    },
    "attributes": {
      "amount": 500.00,
      "currency": "USD",
      "recipientId": "user-uuid"
    },
    "previous": {}
  },
  "delivery": {
    "attempt": 1,
    "maxAttempts": 3,
    "nextRetryAt": "2025-01-15T10:35:00.000Z"
  }
}
```

**Notes:**
- `previous` is only populated on `*_UPDATED` events (contains pre-update values)
- `delivery.maxAttempts` is 3 with exponential backoff
- Financial and split events include an HMAC signature header for verification

### Webhook Delivery Management

| Endpoint | Description |
|----------|-------------|
| `GET /webhook-deliveries` | List deliveries (filterable by status, event type, date) |
| `GET /webhook-deliveries/summary` | Delivery stats |
| `GET /webhook-deliveries/{id}` | Delivery details |
| `POST /webhook-deliveries/{id}/retry` | Retry failed delivery |

Delivery statuses: `pending` | `success` | `failed` | `timeout` | `cancelled`

---

## WebSocket Events

Real-time events via [Socket.io](https://socket.io/) for file processing progress.

### Connection

```javascript
import { io } from "socket.io-client";

const socket = io("wss://api.royalti.io", {
  auth: { token: "<access_token>" }
});
```

Events are user-scoped — only the user who uploaded the file receives processing events.

### File Processing Events

| Event | Description |
|-------|-------------|
| `file:processing:started` | File processing begins |
| `file:processing:progress` | Progress update (~10% intervals) |
| `file:processing:completed` | Processing finished successfully |
| `file:processing:failed` | Processing failed |

### Event Payload

```json
{
  "fileId": "file-uuid",
  "fileName": "spotify-2025-01.csv",
  "status": "processing",
  "progress": 75,
  "message": "Processing row 750 of 1000",
  "error": null,
  "metadata": {
    "rowsProcessed": 750,
    "source": "Spotify",
    "accountingPeriod": "2025-01",
    "salePeriod": "2025-01",
    "processingTimeMs": 45000
  }
}
```

---

## Data Models

### User (Global)

| Field | Type |
|-------|------|
| `id` | UUID |
| `email` | string |
| `isVerified` | boolean |
| `isAdmin` | boolean |
| `provider` | string |
| `googleId` | string |
| `linkedinId` | string |
| `facebookId` | string |

### TenantUser (Workspace-Scoped)

| Field | Type |
|-------|------|
| `id` | UUID |
| `UserId` | UUID (ref: User) |
| `TenantId` | UUID |
| `firstName` | string |
| `lastName` | string |
| `role` | `main_super_admin` \| `super_admin` \| `owner` \| `admin` \| `user` \| `guest` |
| `userType` | string |
| `paymentSettings` | object |
| `permissions` | JSONB |

### Artist

| Field | Type |
|-------|------|
| `id` | UUID |
| `artistName` | string |
| `signDate` | date |
| `label` | string |
| `publisher` | string |
| `copyright` | string |
| `externalId` | string |
| `artistImg` | string (URL) |
| `links` | object (spotify, youtube, instagram, ...) |
| `genres` | string[] |
| `realName` | string |
| `pseudonyms` | string |
| `biography` | string |
| `influences` | string[] |
| `instruments` | string[] |
| `activeYears` | string |
| `associatedActs` | string[] |
| `contributors` | object[] |

### Asset (Track)

| Field | Type |
|-------|------|
| `id` | UUID |
| `title` | string |
| `ISRC` | string |
| `type` | `Audio` \| `Video` \| `Ringtone` \| `YouTube` |
| `version` | string |
| `displayArtist` | string |
| `mainArtist` | string[] |
| `mainGenre` | string[] |
| `subGenre` | string[] |
| `explicit` | boolean |
| `language` | string |
| `tempo` | number |
| `key` | string |
| `mood` | string[] |
| `lyrics` | string |
| `label` | string |
| `copyright` | string |
| `publisher` | string |
| `copyrightOwner` | string |
| `recordingDate` | date |
| `recordingLocation` | string |
| `productionYear` | number |

### Product (Release)

| Field | Type |
|-------|------|
| `id` | UUID |
| `title` | string |
| `UPC` | string |
| `format` | `Single` \| `EP` \| `Album` \| `LP` |
| `release_date` | date |
| `takedown_date` | date |
| `label` | string |
| `display_artist` | string |
| `type` | `Audio` \| `Video` |
| `catalog_number` | string |
| `version` | string |
| `explicit` | boolean |
| `main_genre` | string |
| `sub_genre` | string |
| `status` | `active` \| `inactive` \| `pending` \| `Live` \| `Taken Down` \| `Scheduled` \| `Error` |
| `distribution` | string |
| `external_id` | string |

### Split

| Field | Type |
|-------|------|
| `id` | UUID |
| `entity_type` | `artist` \| `product` \| `asset` |
| `entity_id` | UUID |
| `split_type` | `simple` \| `conditional` \| `temporal` |
| `effective_date` | date |
| `shares` | `[{ user: UUID, percentage: number }]` |
| `conditions` | `{ territories[], mode, sources[], period_start, period_end, memo }` |

### Transaction

| Field | Type |
|-------|------|
| `id` | UUID |
| `type` | `payment` \| `royalty` \| `expense` \| `adjustment` |
| `amount` | number |
| `currency` | string |
| `status` | `pending` \| `processed` \| `failed` |
| `reference` | string |
| `metadata` | object |

---

## Integration Patterns

### Sync Catalog from External System

```
1. POST /artist/bulk          → Create artists
2. POST /product/bulk         → Create products
3. POST /asset/bulk           → Create assets/tracks
4. POST /split/               → Assign splits per entity
5. POST /accounting/refresh   → Recalculate accounting
```

### Upload and Process a Royalty File

```
1. GET  /file/upload-url               → Get presigned upload URL
2. PUT  <presigned_url>                 → Upload file to cloud storage
3. POST /file/confirm-upload-completion → Notify API upload is done
4. GET  /file/detection/{sessionId}     → Poll auto-detection
   OR subscribe to WebSocket events
5. POST /file/confirm-detection/{sessionId} → Confirm detected format
6. GET  /file/processing/{jobId}        → Poll processing status
   OR subscribe to file:processing:progress
7. POST /accounting/refresh             → Recalculate after processing
```

### Fetch Analytics Dashboard Data

```
1. GET /royalty/?start=2025-01-01&end=2025-12-31              → Overview
2. GET /royalty/month?start=2025-01-01&end=2025-12-31         → Monthly trend
3. GET /royalty/dsp?start=2025-01-01&end=2025-12-31           → Platform breakdown
4. GET /royalty/country?start=2025-01-01&end=2025-12-31       → Geographic breakdown
5. GET /royalty/asset?start=2025-01-01&end=2025-12-31&size=10 → Top tracks
```

### Set Up Webhook Listener

```
1. PUT /tenant/settings/webhook-url          → Set your endpoint
2. PUT /tenant/settings/webhook-enabledEvents → Choose event types
3. PUT /tenant/settings/webhook-isActive     → Enable
4. GET /webhook-deliveries                   → Monitor deliveries
5. POST /webhook-deliveries/{id}/retry       → Retry failures
```

---

## Full Documentation

For complete endpoint details, request/response schemas, and the interactive API explorer, visit our [developer documentation](https://docs.royalti.io).

---

## License

This API reference is provided by [Royalti.io](https://royalti.io). See [LICENSE](LICENSE) for details.
