---
name: royalti-api
description: Royalti.io REST API v2.6 pattern reference for developers. Covers authentication, CRUD patterns, pagination, error handling, webhooks, WebSocket events, and data models. Use when helping developers integrate with the Royalti API (api.royalti.io) or when writing integration code, API documentation, or troubleshooting API issues.
license: MIT
compatibility: Designed for Claude Code, also works with Cursor, Codex CLI, Gemini CLI, and other Agent Skills compatible tools.
metadata:
  author: Royalti.io
  version: "1.0.0"
---

# Royalti API v2.6 — Developer Reference

Pattern reference for integrating with the Royalti.io REST API.

**Base URL:** `https://api.royalti.io`
**Current Version:** 2.6.4
**Architecture:** Multi-tenant (workspace-scoped)

---

## 1. Authentication

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

# Response:
# {
#   "refresh_token": "eyJ...",
#   "workspaces": [{ "id": "ws_abc", "name": "My Label", ... }]
# }

# Step 2: Exchange refresh token for access token (scoped to a workspace)
curl https://api.royalti.io/auth/authtoken?currentWorkspace=ws_abc \
  -H "Authorization: Bearer <refresh_token>"

# Response:
# { "data": { "access_token": "eyJ..." } }
```

### Using the Access Token

All subsequent requests include the access token:

```bash
curl https://api.royalti.io/artist/ \
  -H "Authorization: Bearer <access_token>"
```

### API Key Authentication

API keys can be used instead of JWT tokens. Pass them in the `Authorization` header:

```bash
# Workspace API key
curl https://api.royalti.io/artist/ \
  -H "Authorization: Bearer RWAK_abc123..."

# User API key
curl https://api.royalti.io/asset \
  -H "Authorization: Bearer RUAK_def456..."
```

### Other Auth Methods

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /auth/loginlink` | POST | Magic link login |
| `POST /auth/forgotpassword` | POST | Request password reset |
| `PATCH /auth/resetpassword?code=CODE` | PATCH | Apply password reset |
| `GET /auth/google` | GET | Google OAuth |
| `GET /auth/linkedin` | GET | LinkedIn OAuth |
| `GET /auth/facebook` | GET | Facebook OAuth |

### Rate Limiting

- Login endpoint: **20 requests per 3 minutes** per IP
- Other endpoints: No documented limits (subject to fair use)

### RBAC Roles (ascending privilege)

```
guest < user < admin < owner < super_admin < main_super_admin
```

---

## 2. Request Patterns

### Standard Headers

```
Authorization: Bearer <access_token | API_KEY>
Content-Type: application/json
```

### Pagination (all list endpoints)

| Param | Default | Description |
|-------|---------|-------------|
| `page` | `1` | Page number |
| `size` | `10`–`20` | Items per page (max `100`) |
| `sort` | `updatedAt` | Sort field |
| `order` | `desc` | `asc` or `desc` |

```bash
GET /artist/?page=2&size=25&sort=artistName&order=asc
```

### Filtering (analytics endpoints)

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
| `periodFilterType` | `accounting` or `sale` | Date filtering mode |
| `includePreviousPeriod` | boolean | Include comparison data |
| `table_name` | string | Filter by royalty file source table |

### Search

Many list endpoints support a `search` query param for text-based filtering:

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

## 3. Response Patterns

### Success Response

```json
{
  "status": "success",
  "message": "Operation successful",
  "data": { ... }
}
```

### Paginated List Response

```json
{
  "status": "success",
  "data": [ ... ],
  "totalItems": 142,
  "totalPages": 15,
  "currentPage": 1
}
```

### Summary Response (v2.6.4+)

Available on: `/artist/summary`, `/asset/summary`, `/product/summary`, `/user/summary`, `/split/summary`, `/payment/summary`, `/expense/summary`, `/revenue/summary`

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

### Error Response

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

### Common HTTP Status Codes

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

## 4. Core Resources

All resources follow the same CRUD pattern unless noted. Every resource is workspace-scoped (multi-tenant).

### Standard CRUD Pattern

```
GET    /{resource}/          # List (paginated)
POST   /{resource}/          # Create
GET    /{resource}/{id}      # Get by ID
PUT    /{resource}/{id}      # Update
DELETE /{resource}/{id}      # Delete
GET    /{resource}/summary   # Summary stats (v2.6.4+)
GET    /{resource}/download/csv  # CSV export
POST   /{resource}/bulk      # Bulk create
DELETE /{resource}/bulk/delete   # Bulk delete
```

### Users (`/user/`)

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

### Artists (`/artist/`)

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

Key fields: `artistName`, `signDate`, `label`, `publisher`, `copyright`, `externalId`, `artistImg`, `links` (spotify/youtube/instagram/etc.), `genres`, `realName`, `pseudonyms`

### Assets / Tracks (`/asset`)

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

Key fields: `title`, `ISRC`, `type` (Audio|Video|Ringtone|YouTube), `version`, `displayArtist`, `mainArtist[]`, `mainGenre[]`, `subGenre[]`, `explicit`, `language`, `tempo`, `key`, `mood[]`, `lyrics`, `label`, `copyright`, `publisher`

### Products / Releases (`/product/`)

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

Key fields: `title`, `UPC`, `format` (Single|EP|Album|LP), `release_date`, `takedown_date`, `label`, `display_artist`, `type` (Audio|Video), `catalog_number`, `status` (active|inactive|pending|Live|Taken Down|Scheduled|Error), `distribution`, `external_id`

### Splits (`/split/`)

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `POST /split/default` | Create split from artist default |
| `POST /split/match` | Find splits matching criteria |
| `DELETE /split/bulk/catalog-splits` | Remove catalog-level splits |

Split types: `simple`, `conditional`, `temporal`

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

**Rules:** Shares must sum to 100. `entity_type` is `artist`, `product`, or `asset`. Conditions are optional (omit for simple splits).

### Royalties / Analytics (`/royalty/`)

Read-only analytics endpoints. All support the standard filter params (start, end, dsp, country, etc.).

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

Analytics response shape:

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

`PreviousRoyalty` and `PreviousCount` are only present when `includePreviousPeriod=true`.

### Accounting (`/accounting/`)

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

### Payments (`/payment/`)

Standard CRUD. Supports both JSON and `multipart/form-data` (for attaching receipts).

| Endpoint | Description |
|----------|-------------|
| `GET /payment/summary` | Payment totals |
| `POST /payment/bulk` | Bulk create payments |

### Payment Requests (`/payment-request/`)

| Endpoint | Description |
|----------|-------------|
| `GET /payment-request/` | List requests |
| `GET /payment-request/{id}` | Get request |
| `POST /payment-request/{id}/approve` | Approve request |
| `POST /payment-request/{id}/decline` | Decline request |

### Royalty Files (`/file/`)

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

### Labels (`/labels/`)

Standard CRUD plus:

| Endpoint | Description |
|----------|-------------|
| `GET /labels/hierarchy` | Label tree structure |

### Releases (`/releases`)

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

### Notifications (`/notifications/`)

| Endpoint | Description |
|----------|-------------|
| `GET /notifications/` | List notifications |
| `PATCH /notifications/{id}/read` | Mark as read |
| `PATCH /notifications/read-all` | Mark all as read |
| `GET /notifications/unread-count` | Unread count |

### Downloads / Reports (`/download/`)

| Endpoint | Description |
|----------|-------------|
| `POST /download/generate` | Generate report download |
| `GET /download/{id}/status` | Check generation status |
| `GET /download/list` | List available downloads |

---

## 5. Webhooks

### Outbound Webhooks (Royalti → Your Endpoint)

Configure your webhook endpoint via tenant settings:

```bash
# Set webhook URL
PUT /tenant/settings/webhook-url
{ "webhookUrl": "https://yourapp.com/webhooks/royalti" }

# Enable/disable webhooks
PUT /tenant/settings/webhook-isActive
{ "isActive": true }

# Subscribe to event types
PUT /tenant/settings/webhook-enabledEvents
{ "enabledEvents": ["PAYMENT_COMPLETED", "ROYALTY_FILE_PROCESSED"] }
```

### Event Categories

**Financial Events:**
- `PAYMENT_COMPLETED`, `PAYMENT_PROCESSING`, `PAYMENT_MADE_FAILED`
- `PAYMENT_REQUEST_SENT`, `PAYMENT_REQUEST_APPROVED`, `PAYMENT_REQUEST_REJECTED`
- `PAYMENT_DELETED`
- `REVENUE_CREATED`, `REVENUE_UPDATED`, `REVENUE_DELETED`
- `EXPENSE_CREATED`, `EXPENSE_UPDATED`, `EXPENSE_DELETED`

**Catalog Events:**
- `ASSET_CREATED`, `ASSET_UPDATED`, `ASSET_DELETED`
- `PRODUCT_CREATED`, `PRODUCT_UPDATED`, `PRODUCT_DELETED`

**Roster Events:**
- `USER_CREATED`, `USER_UPDATED`, `USER_DELETED`
- `USER_INVITATION_SENT`
- `ARTIST_CREATED`, `ARTIST_UPDATED`, `ARTIST_DELETED`

**Royalty Events (opt-in):**
- `ROYALTY_FILE_UPLOADED`
- `ROYALTY_FILE_PROCESSED`
- `ROYALTY_FILE_PROCESSING_FAILED`

**Split Events:**
- `USER_ADDED_TO_SPLIT`
- `USER_REMOVED_FROM_SPLIT`

### Webhook Payload Structure

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

- `previous` is only populated on `*_UPDATED` events (contains pre-update values)
- `delivery.maxAttempts` is 3 with exponential backoff
- Financial and split events include HMAC signature for verification

### HMAC Signature Validation

Financial and split webhook deliveries include an HMAC signature header for verification. Validate the signature before processing the payload.

### Webhook Delivery Management

| Endpoint | Description |
|----------|-------------|
| `GET /webhook-deliveries` | List deliveries (filterable by status, event type, date) |
| `GET /webhook-deliveries/summary` | Delivery stats |
| `GET /webhook-deliveries/{id}` | Delivery details |
| `POST /webhook-deliveries/{id}/retry` | Retry failed delivery |

Delivery statuses: `pending`, `success`, `failed`, `timeout`, `cancelled`

---

## 6. WebSocket Events (Socket.io)

Real-time events for file processing progress.

### Connection

```javascript
import { io } from "socket.io-client";

const socket = io("wss://api.royalti.io", {
  auth: { token: "<access_token>" }
});
```

Events are user-scoped — only the user who uploaded the file receives processing events.

### File Processing Events

| Event | When |
|-------|------|
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

## 7. Key Data Models

### User (Global)

```
id, email, isVerified, isAdmin, provider, googleId, linkedinId, facebookId
```

### TenantUser (Workspace-Scoped)

```
id, UserId, TenantId, firstName, lastName,
role: main_super_admin | super_admin | owner | admin | user | guest,
userType, paymentSettings, permissions (JSONB)
```

### Artist

```
id (UUID), artistName, signDate, label, publisher, copyright,
externalId, artistImg, links {spotify, youtube, instagram, ...},
genres[], realName, pseudonyms, biography, influences,
instruments, activeYears, associatedActs, contributors
```

### Asset (Track)

```
id (UUID), title, ISRC, type (Audio|Video|Ringtone|YouTube),
version, displayArtist, mainArtist[], mainGenre[], subGenre[],
explicit, language, tempo, key, mood[], lyrics,
label, copyright, publisher, copyrightOwner,
recordingDate, recordingLocation, productionYear,
enableDDEX, focusTrack, ddexMetadata, resourceReference
```

### Product (Release)

```
id (UUID), title, UPC, format (Single|EP|Album|LP),
release_date, takedown_date, label, display_artist,
type (Audio|Video), catalog_number, version, explicit,
main_genre, sub_genre, status (active|inactive|pending|Live|Taken Down|Scheduled|Error),
distribution, external_id
```

### Split

```
id, entity_type (artist|product|asset), entity_id,
split_type (simple|conditional|temporal),
effective_date,
shares: [{ user: UUID, percentage: 0-100 }],
conditions: { territories[], mode, sources[], period_start, period_end, memo }
```

### Transaction

```
id (UUID), type (payment|royalty|expense|adjustment),
amount, currency, status (pending|processed|failed),
reference, metadata
```

---

## 8. Common Integration Patterns

### Sync Catalog from External System

```
1. POST /artist/bulk          → Create artists
2. POST /product/bulk         → Create products
3. POST /asset/bulk           → Create assets/tracks
4. POST /split/               → Assign splits per entity
5. POST /accounting/refresh   → Recalculate accounting
```

### Upload and Process Royalty File

```
1. GET  /file/upload-url               → Get presigned upload URL
2. PUT  <presigned_url>                 → Upload file to cloud storage
3. POST /file/confirm-upload-completion → Notify API upload is done
4. GET  /file/detection/{sessionId}     → Poll auto-detection (or listen via WebSocket)
5. POST /file/confirm-detection/{sessionId} → Confirm detected format
6. GET  /file/processing/{jobId}        → Poll processing status
   OR subscribe to WebSocket: file:processing:progress
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

## 9. Code Examples

Language-specific integration examples are available in the `references/` directory:

- `references/examples-javascript.md` — Node.js/TypeScript client, auth, CRUD, file upload, WebSocket, Express webhook receiver
- `references/examples-python.md` — Python client, auth, pagination, file upload, Flask webhook receiver, socketio
- `references/examples-php.md` — PHP client, auth, pagination, file upload, Laravel and plain PHP webhook receivers

Load these files when the developer is working in a specific language.

## 10. Further Reading

- **API Explorer:** [apidocs.royalti.io](https://apidocs.royalti.io)
- **MCP Server:** [royalti.io/mcp](https://royalti.io/mcp) — AI assistant integration for direct API interaction
- **MCP Setup Guide:** [royalti.io/help/setting-up-the-royalti-mcp-server](https://royalti.io/help/setting-up-the-royalti-mcp-server)
