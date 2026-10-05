---
name: apulodi-media-transforms
license: MIT
description: >
  Transform media server-side with the APULODI API: image resizing and
  format conversion (WebP, AVIF, JPEG, PNG), video transcoding and poster
  frames, and audio conversion. Use when a task needs thumbnails, gallery
  renditions, optimized delivery formats, or video/audio processing on
  uploaded files, or needs to manage variants and their statuses.
---

# APULODI media transforms

All endpoints are relative to `https://app.apulodi.dev/api/v1` and require
`Authorization: Bearer <api-key>`. See the `apulodi` skill for auth and
envelopes.

**Mental model:** a transform creates a **variant** — a derived rendition of
a file, tracked separately from the original. Transforms are **asynchronous**:
the create call returns quickly with the variant in `pending`, and processing
finishes shortly after. Your code must handle the async contract.

## Creating a transform

```http
POST /files/{fileId}/transform
Content-Type: application/json

{ "kind": "image", "width": 640, "format": "webp", "quality": 80 }
```

Response: `200` with the variant already `ready`, or **`202` with the
variant `pending`** — the common case. Do not treat `pending` as failure.

### Image transforms

- `width` / `height`: integers **16–4096** (at least one required)
- `fit`: `cover` | `contain` | `inside` (how the image fills the box)
- `format`: `webp` | `jpeg` | `png` | `avif`
- `quality`: 1–100 (meaningful for lossy formats)

```json
{ "kind": "image", "width": 1600, "format": "webp", "quality": 80, "fit": "cover" }
```

### Video transforms

- `format`: `mp4` | `webm`, with `height` (e.g. 720 for 720p), **or**
  `poster: true` to extract a poster frame image.

```json
{ "kind": "video", "format": "mp4", "height": 720 }
{ "kind": "video", "poster": true }
```

### Audio transforms

- `format`: `mp3` | `m4a` | `ogg` | `wav`

```json
{ "kind": "audio", "format": "mp3" }
```

## The async contract (important)

A fresh variant is `pending`; it becomes `ready` after processing (usually
seconds). Two ways to await:

1. **Poll** `GET /files/{fileId}/variants` until the target variant's
   `status === "ready"`.
2. **Webhook** — listen for `file.processed`. Preferred for server apps.

**SDKs handle polling for you:**

```typescript
const variant = await apulodi.files.transform(file.id, {
  kind: "image", width: 640, format: "webp",
});
const ready = await apulodi.files.waitForVariant(file.id, variant.id, {
  timeoutMs: 30000,
});
```

```python
variant = apulodi.files.transform(
    file["id"], kind="image", width=640, format="webp"
)
ready = apulodi.files.wait_for_variant(file["id"], variant["id"], timeout=30)
```

**curl (create, then poll):**

```bash
curl -X POST "$BASE/files/file_123/transform" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"kind":"image","width":640,"format":"webp","quality":80}'

curl "$BASE/files/file_123/variants" -H "Authorization: Bearer $KEY"
```

## Delivering variants

Variant bytes are served through **signed URLs** (same as files):

```http
POST /files/{fileId}/variants/{variantId}/download-url
```

```python
url = apulodi.files.variant_download_url(file_id, variant_id)
```

```typescript
const { url } = await apulodi.files.variantDownloadUrl(fileId, variantId);
```

For `visibility: public` files, immutable CDN URLs are available — suitable
for cache-busting `<img>`/`<video>` tags without a signed round-trip.

## Patterns that work well

- **Thumbnail at upload time.** Transform right after the upload completes
  and `waitForVariant` while the user is still on the form; store the ready
  variant id so list pages need zero transforms on render. This avoids the
  classic "placeholder until refresh" bug.
- **Idempotent transforms.** Re-requesting the same transform is safe;
  reuse the existing ready variant instead of transforming again.
- **Progressive renditions.** A 640 WebP for grids, 1600 WebP for detail
  views, `poster` + 720p MP4 for video pages — all from one original.
- **Timeout degradation.** Wrap waits in a timeout and fall back to the
  original file or a placeholder rather than failing the page.

## Troubleshooting

- **Image never appears** → you rendered before the variant was `ready`, or
  you only checked the create call's status code. Use `waitForVariant` or
  the `file.processed` webhook.
- **422 on create** → check dimension bounds (16–4096) and enum values.
- **Variant 404 on download-url** → variant ids belong to their file; scope
  lookups by file id and don't cache variant ids across files.
