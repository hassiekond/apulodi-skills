---
name: apulodi-file-uploads
license: MIT
description: >
  Upload, manage, and deliver files through the APULODI API. Covers the
  3-step direct upload flow (initiate, presigned PUT, complete), multipart
  uploads for large files, listing and searching files, metadata updates,
  rename/copy/restore, deletion, and signed download URLs. Use when a task
  needs to get files into or out of APULODI storage, organize them into
  folders, or manage their lifecycle.
---

# APULODI file uploads and file management

All endpoints below are relative to the base URL
`https://app.apulodi.dev/api/v1` and require
`Authorization: Bearer <api-key>`. See the `apulodi` skill for auth,
envelopes, and idempotency rules first.

**Core principle:** APULODI never proxies file bytes. Uploads go directly to
storage through presigned PUT URLs; downloads go directly from storage
through presigned GET URLs. The API handles authorization, metadata, and
lifecycle.

## The 3-step upload flow

### Step 1 — initiate

```http
POST /files/upload
Content-Type: application/json
Idempotency-Key: <unique-uuid>        # recommended for safe retries

{
  "fileName": "cover.jpg",
  "contentType": "image/jpeg",
  "size": 48291,
  "path": "listings",
  "metadata": { "listingId": "lst_1" }
}
```

- `path` (optional) is a logical folder like `users/avatars`. Traversal
  (`..`), absolute paths, and backslashes are rejected. Filenames are
  normalized server-side; the client cannot influence the internal object key.
- `size` (optional but recommended) and `contentType` are validated at
  completion against the real stored object.

Response: `{ "file": {...}, "upload": { "url", "method", "headers",
"expiresAt" } }`. The file is created in `pending` status.

### Step 2 — direct PUT to storage

`PUT` the raw bytes to `upload.url` using **exactly** the returned headers
(typically the `Content-Type` from step 1). Do not add or alter headers —
the signature covers them.

**TypeScript:**

```typescript
const res = await fetch(upload.url, {
  method: "PUT",
  headers: upload.headers,
  body: fileBytes,
});
if (!res.ok) throw new Error(`Storage upload failed: ${res.status}`);
```

**curl:**

```bash
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @cover.jpg
```

### Step 3 — complete

```http
POST /files/{fileId}/complete
```

The server **verifies the object actually exists in storage** (and its
metadata) before marking the file `uploaded`. Never mark an upload done on
client say-so. Status flow: `pending → uploaded`; failures `failed`;
deletion (soft) `deleted`.

**SDK one-liner (handles all steps; multipart above ~8 MiB automatically):**

```typescript
const file = await apulodi.files.upload(buffer, {
  fileName: "cover.jpg",
  contentType: "image/jpeg",
  path: "listings",
});
```

```python
file = apulodi.files.upload(
    open("cover.jpg", "rb"),
    file_name="cover.jpg",
    content_type="image/jpeg",
    path="listings",
)
```

## Validation limits

- Size: 1 byte up to the project maximum (default 100 MiB,
  env-configurable). Over the limit → `PAYLOAD_TOO_LARGE` (413).

## Listing files

```http
GET /files?limit=20&path=listings&contentType=image/jpeg&status=uploaded&search=cover&sort=createdAt&order=desc
```

- Filters: `path`, `contentType`, `status`, `search`, `createdAt` range.
- Sorting: `sort` ∈ `createdAt` | `updatedAt` | `filename` | `size`,
  `order` ∈ `asc` | `desc` (validated server-side; never interpolated SQL).
- Cursor pagination: when `pagination.hasMore` is true, pass
  `pagination.nextCursor` as `cursor` on the next request.

```typescript
for await (const file of apulodi.files.iterate({ path: "listings" })) {
  console.log(file.fileName, file.size, file.status);
}
```

```python
for file in apulodi.files.iterate(path="listings"):
    print(file["fileName"], file["size"], file["status"])
```

## File lifecycle operations

| Operation | HTTP | Notes |
| --- | --- | --- |
| Get metadata | `GET /files/{id}` | Includes `path`, `metadata`, `status` |
| Update | `PATCH /files/{id}` | `fileName`, `path` (move), `visibility` (`public`/`private`), `metadata` |
| Copy | `POST /files/{id}/copy` | Body `{"path": "backups"}`; new file id, original untouched |
| Delete (soft) | `DELETE /files/{id}` | Removes storage bytes; status `deleted`; fires `file.deleted` |
| Restore | `POST /files/{id}/restore` | Where supported; status back to `uploaded` |
| Download URL | `POST /files/{id}/download-url` | Short-lived presigned GET `url` + `expiresAt` (default ~5 min) |

`visibility: public` enables immutable CDN URLs; `private` (default)
allows only signed, expiring access.

**Python:**

```python
apulodi.files.update(file_id, path="listings/2026", metadata={"city": "Lilongwe"})
url = apulodi.files.download_url(file_id)["url"]
apulodi.files.delete(file_id)                      # fires file.deleted
```

**curl:**

```bash
curl -X PATCH "$BASE/files/file_123" -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"path": "listings/2026"}'

curl -X POST "$BASE/files/file_123/download-url" -H "Authorization: Bearer $KEY"
```

## Multipart uploads (large files)

For large files or unreliable networks. Same auth and envelopes.

1. **Initiate:** `POST /uploads/multipart` — same body as single upload.
   Returns the `file` plus a session including `partSize` (e.g. 8 MiB).
2. **Presign parts:** `POST /uploads/multipart/{sessionId}/parts` with
   `{"partNumbers": [1, 2, 3]}` → one presigned PUT URL per part. PUT each
   chunk of exactly `partSize` bytes (last may be smaller); record each
   returned `ETag`, **stripping surrounding quotes**.
3. **Complete:** `POST /uploads/multipart/{sessionId}/complete` with the
   ordered `[{"partNumber": 1, "etag": "..."}, ...]`. Server verifies and
   marks the file `uploaded`.
4. **Abort:** `DELETE /uploads/multipart/{sessionId}` on abandonment or
   failure — leftover parts cost storage.

**Python (explicit control):**

```python
session = apulodi.uploads.create_multipart(
    "tour.mp4", content_type="video/mp4", size=len(data)
)
parts = []
for i, chunk in enumerate(chunks(data, session["partSize"]), start=1):
    etag = apulodi.uploads.upload_part(session["id"], i, chunk)
    parts.append({"partNumber": i, "etag": etag})
file = apulodi.uploads.complete(session["id"], parts)
```

The TypeScript SDK slices and uploads parts automatically via
`apulodi.files.upload` with a stream/blob larger than the threshold, and
regenerates expired part URLs.

## Common integration mistakes

- **Proxying bytes through your backend** — pass the presigned URL to the
  client; never relay file bodies through your server.
- **Skipping `complete`** — a successful PUT does not finish the upload.
- **Altering presigned headers** — any change fails storage with 403.
- **Forgetting `Idempotency-Key`** on retryable initiations — retries
  create duplicate pending files.
- **Trusting client-declared size/type** — the server re-verifies at
  completion, but validate early for good errors.
- **Assuming delete is instantly reconciled** — await `file.deleted`
  (see `apulodi-webhooks`) before treating storage as clean.
