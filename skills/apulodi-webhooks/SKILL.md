---
name: apulodi-webhooks
license: MIT
description: >
  Receive and handle APULODI webhooks safely. Covers verifying webhook
  signatures (HMAC-SHA256 over "{t}.{body}" with a 5-minute replay window),
  handling at-least-once delivery with event-id deduplication, responding
  within the timeout, and the full event catalog including file.uploaded,
  file.processed, file.deleted, and delivery failure events. Use when
  building a webhook receiver for APULODI or reacting to file lifecycle
  events in an application.
---

# APULODI webhooks

Webhooks push APULODI events (uploads, transforms, deletions, delivery
failures) to your HTTPS endpoint. Configure endpoints in the dashboard with
one or more event types; each endpoint gets a signing secret shown once at
creation — store it server-side only.

## Delivery format

```http
POST /your-endpoint
Content-Type: application/json
APULODI-Signature: t=1760000000,v1=5f8a...
```

The body is the event envelope:

```json
{
  "id": "evt_8f92...",
  "type": "file.processed",
  "createdAt": "2026-10-04T12:00:00.000Z",
  "data": { "file": { "id": "file_..." }, "variant": { } }
}
```

`data` contains the relevant resource: `file` always; `variant` for
transform events.

## Signature verification (required — always do this)

The `APULODI-Signature` header is `t=<unix-seconds>,v1=<hex>`, where
`v1` is HMAC-SHA256 of the byte string `"{t}.{rawBody}"` keyed with the
endpoint secret. Verify against the **raw request body** — re-serialized
JSON will not match.

Algorithm:

1. Parse `t` and `v1` from the header (comma-separated `k=v` pairs).
2. Reject if `t` is missing, unparseable, or older than **300 seconds**
   (replay protection). Compare clocks in UTC — use timezone-aware
   timestamps, not local-time `mktime`.
3. Compute `HMAC_SHA256(secret, "{t}.{rawBody}")` and compare to `v1`
   with a **constant-time** comparison.
4. Reject anything that fails before parsing/acting on the body.

**TypeScript:**

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyWebhookSignature(
  rawBody: string, header: string, secret: string, tolerance = 300
): boolean {
  const parts = Object.fromEntries(
    header.split(",").map((p) => p.split("=") as [string, string])
  );
  const t = Number(parts.t);
  if (!t || Math.abs(Date.now() / 1000 - t) > tolerance) return false;
  const expected = createHmac("sha256", secret)
    .update(`${t}.${rawBody}`).digest("hex");
  const a = Buffer.from(expected), b = Buffer.from(parts.v1 ?? "");
  return a.length === b.length && timingSafeEqual(a, b);
}
```

**Python (the `apulodi` SDK ships this):**

```python
from apulodi import verify_webhook_signature

ok = verify_webhook_signature(
    request_body,                       # raw bytes, exactly as received
    request.headers["APULODI-Signature"],
    webhook_secret,
)
if not ok:
    return ("invalid signature", 401)
```

**Never** verify against parsed-then-re-dumped JSON, and never log bodies
that may contain secrets.


## Receiver best practices

- **Deduplicate.** Delivery is at-least-once; the same event can arrive
  more than once. Track processed `event.id`s (cache or DB unique
  constraint) and skip repeats.
- **Respond fast.** Return `2xx` within ~10 seconds. Do the work
  asynchronously (queue/background job) if it is heavy.
- **Return 4xx for permanently bad requests** (bad signature = 401, stale
  timestamp = 400) so APULODI stops retrying them; transient failures may
  return 5xx to trigger retry.
- **Read `type` before `data`.** The event set grows over time; ignore
  unknown types rather than crashing.
- **Re-fetch when unsure.** Events are hints; load current state via the
  API (`GET /files/{id}`) before acting on stale beliefs.

## Event catalog

| Event | Meaning / typical reaction |
| --- | --- |
| `file.uploaded` | A file completed its upload. Update records, kick off pipelines |
| `file.processed` | A transform variant finished (contains `variant`). Flip UIs from placeholder to variant |
| `file.updated` | Metadata/path/visibility changed. Invalidate caches |
| `file.deleted` | File removed from storage. **Delete your mirrored records** — the standard cleanup hook |
| `file.restored` | Soft-deleted file restored |
| `file.replaced` | Contents replaced (new version). Refresh renditions |
| `upload.failed` | An upload failed before completing. Clean pending state |
| `webhook.delivery.failed` | APULODI could not deliver one of your webhooks — alert/inspect your endpoint |
| `webhook.delivery.recovered` | A previously failing endpoint delivered again |

Treat unknown event types as no-ops with 200.

## Receivers

**TypeScript (Express-style):**

```typescript
app.post("/apulodi-webhook", express.raw({ type: "*/*" }), (req, res) => {
  const raw = req.body.toString("utf8");
  if (!verifyWebhookSignature(raw, req.header("APULODI-Signature") ?? "", secret)) {
    return res.status(401).send("invalid signature");
  }
  const event = JSON.parse(raw);
  if (seen.has(event.id)) return res.sendStatus(200);   // dedup
  seen.add(event.id);
  if (event.type === "file.deleted") void deleteLocalRecord(event.data.file.id);
  res.sendStatus(200);
});
```

**Python (Flask):**

```python
@app.post("/apulodi-webhook")
def apulodi_webhook():
    raw = request.get_data()  # raw bytes — do not parse first
    if not verify_webhook_signature(
        raw, request.headers.get("APULODI-Signature", ""), WEBHOOK_SECRET
    ):
        return "invalid signature", 401
    event = json.loads(raw)
    if event["type"] == "file.deleted":
        db.delete(event["data"]["file"]["id"])
    return "", 200
```

## Debugging deliveries

Inspect **deliveries** per endpoint and **redeliver** an event after
fixing your receiver:

```python
deliveries = apulodi.webhooks.deliveries(endpoint_id)
apulodi.webhooks.redeliver(endpoint_id, delivery_id)
```

```typescript
const deliveries = await apulodi.webhooks.deliveries(endpointId);
await apulodi.webhooks.redeliver(endpointId, deliveryId);
```

Read the endpoint secret from configuration (env) — never hardcode it —
and always verify the signature before parsing or acting on a body.
