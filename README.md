# Webhook Receiver

A Next.js 14 service demo that receives, verifies, and stores webhook events from third-party providers.

---

## Stack

- **Next.js 14** (App Router, API Routes)
- **PostgreSQL** (via `pg`)
- **TypeScript**

---

## Setup

### Prerequisites

- Node 18+
- PostgreSQL running locally

### 1. Install dependencies

```bash
npm install
```

### 2. Create `.env.local`

```
DATABASE_URL=postgresql://postgres:password@localhost:5432/fluff_assessment
WEBHOOK_SECRET=fluff-secret-abc123
```

### 3. Run the schema

```bash
psql postgresql://postgres:password@localhost:5432/fluff_assessment -f sql/schema.sql
```

### 4. Start the dev server

bash npm run dev

Server runs on `http://localhost:3000`.

---

## Endpoints

### `POST /api/webhooks`

Receives an event. Verifies the `X-Signature` header using HMAC-SHA256 with the configured secret.

**Headers:**
- `Content-Type: application/json`
- `X-Signature: <hmac-sha256-hex>` (also accepts `sha256=<hex>` prefix format)

**Example — generate a valid signature and send an event:**

```bash
SECRET="fluff-secret-abc123"
BODY='{"type":"user.created","data":{"id":42,"email":"test@example.com"}}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST http://localhost:3000/api/webhooks \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -d "$BODY"
```

**Responses:**
- `201` — event stored successfully
- `401` — signature missing or invalid
- `400` — body is not valid JSON
- `500` — storage failed (event parked in `dead_letter_queue`)

---

### `GET /api/webhooks`

Lists received events, most recent first.

```bash
curl http://localhost:3000/api/webhooks
curl http://localhost:3000/api/webhooks?limit=10
```

Max limit: 200. Default: 50.

Test invalid Signature:
```bash
curl -X POST http://localhost:3000/api/webhooks \
  -H "Content-Type: application/json" \
  -H "X-Signature: badsignature" \
  -d '{"type":"user.created"}'
```
---

## How failures are handled

If the `INSERT` into `webhook_events` fails (e.g. DB connection lost), the raw body and signature are written to a `dead_letter_queue` table instead. The caller receives a `500` and should retry.

It preserves enough information to replay any event once the underlying issue is resolved — the `raw_body` and `signature` are both stored, so a recovery script can re-POST them to the endpoint or insert directly.

---

## Decisions made

**HMAC verification uses `timingSafeEqual`** — a regular string comparison (`===`) is vulnerable to timing attacks that can leak information about the secret. `timingSafeEqual` takes constant time regardless of where the strings diverge.

**Raw body is stored alongside the parsed payload** — the parsed JSONB is useful for querying; the raw body is the source of truth for signature verification and replaying events from the DLQ.

**Dead letter queue** — the requirement said "nothing gets lost if something goes wrong." A DLQ table is the simplest durable fallback that doesn't require additional infrastructure. The trade-off is that retries are manual or need a separate job.

**`event_type` extracted to a top-level column** — makes it practical to filter or aggregate events by type without querying into the JSONB. Falls back to `"unknown"` if the payload doesn't include a `type` field.

---

## What I'd do with more time

- **DLQ retry job** — a cron or background worker that periodically retries unresolved DLQ entries and marks them resolved on success.
- **Logging** — structured logging per event (received, verified, stored, failed) so you can trace any event through the system.
- **Filtering on GET** — filter by `event_type`, date range, or pagination cursor rather than just a limit.
- **Auth on GET** — the list endpoint is currently open; in production it should require a token.
