---
name: Upload a file into Brandfolder
description: Get a temporary upload URL, push file bytes to it, then create the asset and attachment in a Brandfolder or collection.
api: openapi/brandfolder-openapi-original.yml
operations:
  - opIdStorageserviceUploadRequestsGet
  - opIdStorageserviceBfUploadRequestBucketPut
  - opIdStorageserviceBfUploadRequestPost
  - opIdStorageserviceBfUploadRequestPut
  - opIdApiV4CollectionsAssetsByBrandfolderIdPost
  - opIdApiV4CollectionsAssetsByCollectionIdPost
  - opIdApiV4AttachmentsByIdGet
generated: '2026-08-13'
method: generated
source: openapi/brandfolder-openapi-original.yml + https://developers.smartsheet.com/api/brandfolder/guides/recipes/uploading-media-to-brandfolder
---

# Upload a file into Brandfolder

Base URL `https://brandfolder.com/api/v4`, `Authorization: Bearer {BF_API_KEY}`,
`Content-Type: application/json`.

## The rule that shapes everything

**Brandfolder does not accept file bytes on the asset-creation endpoint.** Any
file you want to become an Attachment must already be hosted at a publicly
reachable URL when you create the asset. If you hold the bytes rather than a
URL, you must first put them somewhere Brandfolder can fetch them — and
Brandfolder gives you a temporary bucket for exactly that.

## 1. Get a temporary upload URL

`opIdStorageserviceUploadRequestsGet` → `GET /upload_requests`

Returns an upload target in Brandfolder's temporary storage.

## 2. Push the bytes

`opIdStorageserviceBfUploadRequestBucketPut` → `PUT /upload_url`

Write the file content to the URL returned in step 1. For large files use the
resumable pair instead:

- `opIdStorageserviceBfUploadRequestPost` → `POST /resumable_upload_url` to start
- `opIdStorageserviceBfUploadRequestPut` → `PUT /resumable_upload_url` to resume

## 3. Create the asset

Into a Brandfolder:

`opIdApiV4CollectionsAssetsByBrandfolderIdPost` →
`POST /brandfolders/{brandfolder_id}/assets`

Into a collection:

`opIdApiV4CollectionsAssetsByCollectionIdPost` →
`POST /collections/{collection_id}/assets`

The request body carries the asset attributes — `name`, `description`,
`thumbnail_url`, `approved`, optionally `availability_start` /
`availability_end`, and the `attachments` array whose entries reference the
public URL from step 2. Read the exact request schema from
`openapi/brandfolder-openapi-original.yml`; do not guess field names.

A new asset lands in a **section**, so know which section you want before you
start (`opIdApiV4SectionsGet` → `GET /brandfolders/{brandfolder_id}/sections`).

## 4. Confirm

`opIdApiV4AttachmentsByIdGet` → `GET /attachments/{attachment_id}` returns
`mimetype`, `extension`, `filename`, `size`, `width`, `height`, `url` and
`position` — the cheapest way to confirm Brandfolder actually fetched and
processed the file rather than just recording the URL.

## Retry safety — read this before you automate

**There is no idempotency contract.** Brandfolder documents no idempotency key,
and the OpenAPI declares no `Idempotency-Key` parameter on any of its 30 write
operations. A retried `POST /brandfolders/{id}/assets` creates a *second asset*.

If a create times out or returns 5xx:

1. Do **not** blindly retry.
2. List first — `GET /brandfolders/{brandfolder_id}/assets?search=<name>` — and
   check whether the asset already landed.
3. Only create again if it did not.

Keep your own client-side dedupe key (e.g. a content hash in `description` or a
custom field) if you are uploading in bulk; the API will not do it for you.

## Errors

- `400` — missing a required attribute in the request body, or a malformed payload.
- `403` — the key's user lacks write permission on that Brandfolder or collection.
- `404` — wrong `brandfolder_id` / `collection_id`.
- `429` — back off exponentially; no `Retry-After` is returned.

## Availability

`availability_start` and `availability_end` on the create request control when
the asset publishes and expires. An unpublished or pending-approval asset's
`cdn_url` returns 403/404 at the edge — availability is enforced on delivery,
not only in the API, so an asset can exist and still be undeliverable.
