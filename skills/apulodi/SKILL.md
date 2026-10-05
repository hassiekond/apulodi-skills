---
name: apulodi
license: MIT
description: >
  Build integrations with APULODI, a developer-first file and media
  infrastructure platform. Use when an application needs to upload, store,
  organize, transform, or deliver files and media — image thumbnails and
  conversions, video transcoding and posters, audio conversion, signed
  download URLs, folders, file versioning, webhooks, or usage reporting —
  through the APULODI REST API or its official TypeScript and Python SDKs.
  Covers authentication, the direct-to-storage upload flow, multipart
  uploads, media transforms, and webhook receivers.
---

# APULODI file and media infrastructure

APULODI is a developer-first file infrastructure platform. Developers create an
API key for a project, upload files directly to storage via presigned URLs
(APULODI never proxies file bytes), transform media server-side, and deliver
downloads through short-lived signed URLs.

## Read the live docs

The live docs are the source of truth for anything not covered here:

- API reference: https://docs.apulodi.dev/api
- TypeScript SDK: https://docs.apulodi.dev/docs/sdk
- Python SDK: https://docs.apulodi.dev/docs/sdk/python

## Authentication

All API requests authenticate with a **project-scoped API key** sent as a
Bearer token. Keys are created in the APULODI dashboard per project and look
like `apk_test_...` (test) or `apk_live_...` (live).

```http
Authorization: Bearer apk_test_1a2b3c...
```

**Critical rules:**

- API keys are **server-side credentials**. Never expose them in browser
  code, mobile bundles, or client-side storage.
- The key determines the project. Every request is scoped to that project —
  a key can only see and manipulate files belonging to its own project.
- Never log full keys. On authentication failure the API returns
  `401` with an error envelope.

## Base URL

```text
https://app.apulodi.dev/api/v1
```

All paths in the sub-skills are relative to this base.

## Response format

Success responses use a `data` envelope:

```json
{ "data": { "id": "file_8f92...", "..." : "..." } }
```

List endpoints add cursor pagination:

```json
{
  "data": [ ... ],
  "pagination": { "hasMore": true, "nextCursor": "file_..." }
}
```

Errors use a consistent envelope with appropriate HTTP status:

```json
{
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "The requested file could not be found.",
    "details": {}
  }
}
```

Common codes: `VALIDATION_ERROR` (400/422), `UNAUTHORIZED` (401),
`FORBIDDEN` (403), `FILE_NOT_FOUND` (404), `PAYLOAD_TOO_LARGE` (413),
`CONFLICT` (409), `INTERNAL_ERROR` (500). Treat `5xx` as retryable with
exponential backoff; `4xx` as caller error.

## Idempotency

Upload-initiation accepts an `Idempotency-Key` header. Send a unique key
(e.g. a UUID) per logical upload so network retries do not create duplicate
pending files:

```http
Idempotency-Key: 0c3d6a5e-...
```

## Key behaviors to know before integrating

1. **Uploads are 3 steps.** Initiate → direct PUT to a presigned URL →
   complete. See `apulodi-file-uploads`.
2. **Transforms are async.** `POST .../transform` usually returns **202** with
   a variant in `pending`; poll or await the `file.processed` webhook until
   `ready`. See `apulodi-media-transforms`.
3. **Downloads are signed.** File bytes are only reachable via short-lived
   presigned URLs; there is no public object access.
4. **Deletes fire webhooks.** Deletion removes storage bytes and emits
   `file.deleted`; receivers should clean up their own records. See
   `apulodi-webhooks`.

## SDKs

Prefer the official SDKs over raw HTTP — they handle the upload handshake,
multipart part slicing, ETag bookkeeping, polling, and webhook verification.

TypeScript (npm):

```bash
npm install @apulodi/sdk
```

```typescript
import { Apulodi } from "@apulodi/sdk";

const apulodi = new Apulodi({ apiKey: process.env.APULODI_API_KEY! });

const file = await apulodi.files.upload(buffer, {
  fileName: "photo.jpg",
  contentType: "image/jpeg",
  path: "listings",
});
```

Python (PyPI, Python 3.9+, sole dependency `httpx`):

```bash
pip install apulodi
```

```python
import os
from apulodi import Apulodi

apulodi = Apulodi(api_key=os.environ["APULODI_API_KEY"])

file = apulodi.files.upload(
    open("photo.jpg", "rb"),
    file_name="photo.jpg",
    content_type="image/jpeg",
    path="listings",
)
```

## Sub-skills

Load the relevant sub-skill for the task at hand:

| Skill | Load when the task involves |
| --- | --- |
| `apulodi-file-uploads` | Uploading files (simple or large/multipart), listing/searching files, metadata updates, copy/rename/move, delete/restore, download URLs |
| `apulodi-media-transforms` | Thumbnails, image resizing/format conversion, video transcoding or posters, audio conversion, variant management |
| `apulodi-webhooks` | Receiving APULODI events, verifying webhook signatures, handling delivery retries, reacting to upload/transform/delete lifecycle events |
| `apulodi-management` | Creating or listing organizations/projects, inviting or managing team members, creating/revoking API keys (session-authenticated dashboard APIs, not API-key auth) |
