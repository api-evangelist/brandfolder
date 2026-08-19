---
name: Find assets in Brandfolder
description: Locate the right digital assets in a Brandfolder by browsing the hierarchy or running a search, then read the delivery URL for a specific asset.
api: openapi/brandfolder-openapi-original.yml
operations:
  - list-brandfolders
  - opIdApiV4SectionsGet
  - opIdApiV4BrandfoldersAssetsByBrandfolderIdGet
  - opIdApiV4CollectionsGet
  - opIdApiV4CollectionsAssetsByCollectionIdGet
  - opIdApiV4SectionsAssetsBySectionIdGet
  - opIdApiV4LabelsAssetsByLabelIdGet
  - opIdApiV4AssetsByIdGet
generated: '2026-08-13'
method: generated
source: openapi/brandfolder-openapi-original.yml + https://developers.smartsheet.com/api/brandfolder/guides/recipes
---

# Find assets in Brandfolder

Base URL `https://brandfolder.com/api/v4`. Every request needs
`Authorization: Bearer {BF_API_KEY}`. The key is a *user* key — you will only
ever see what that user can see.

## 1. Get your bearings

Brandfolder IDs are opaque triplets like `oqgkkd-fr5iv4-443db` and carry no type
prefix, so you cannot guess one or tell an asset key from a section key by
looking at it. Always start by listing.

- `list-brandfolders` → `GET /brandfolders` returns every Brandfolder the key
  can reach. Take `data[].id`.
- `opIdApiV4SectionsGet` → `GET /brandfolders/{brandfolder_id}/sections` for the
  sections inside it. Match on `data[].attributes.name`, take `data[].id`.
- `opIdApiV4CollectionsGet` → `GET /collections` for collections across the
  account.

## 2. Search rather than page

`opIdApiV4BrandfoldersAssetsByBrandfolderIdGet` →
`GET /brandfolders/{brandfolder_id}/assets` accepts a `search` parameter that
uses the same syntax as the Brandfolder web UI search bar. URL-encode the value.

```
GET /brandfolders/{brandfolder_id}/assets?search=extension%3Apng
GET /brandfolders/{brandfolder_id}/assets?search=label%3A%22Social%22
```

The same `search` parameter works on
`opIdApiV4SectionsAssetsBySectionIdGet` (`/sections/{section_id}/assets`),
`opIdApiV4CollectionsAssetsByCollectionIdGet` (`/collections/{collection_id}/assets`)
and `opIdApiV4LabelsAssetsByLabelIdGet` (`/labels/{label_id}/assets`).

Searching is almost always the right move: paging the whole Brandfolder to
filter client-side burns request budget against an API that publishes no rate
limits, so you cannot tell how close you are to a 429.

## 3. Ask for the fields you actually need

The default asset payload is thin — `name`, `description`, `thumbnail_url`,
`approved`. Anything else must be requested:

```
?fields=cdn_url,updated_at
?include=brandfolder,section,attachments
```

`fields` adds attributes to each resource. `include` adds whole related records
to a top-level `included` array and fills `relationships` on each item. The docs
warn that both slow the response, so request only what you will use.

`cdn_url` is the delivery URL (`https://cdn.bfldr.com/...?auto=webp&format=png`).
It is not returned by default — if you need to place an asset somewhere, you
must ask for it.

## 4. Page and sort deterministically

- `page` is 1-based. `per` defaults to 100 and maxes at 3000.
- `meta.total_count` on the response is how you know when to stop — there is no
  cursor and no next-page link, so you compute page numbers yourself.
- `sort_by` accepts `name`, `score`, `position`, `updated_at`, `created_at`, and
  **must** be sent together with `order` (`ASC` or `DESC`). Either one alone is
  not a valid request.

Prefer `per=3000` on large listings over paging at the default 100.

## 5. Fetch one asset

`opIdApiV4AssetsByIdGet` → `GET /assets/{asset_id}`, with the same `fields` and
`include` parameters.

## Error handling

Branch on the HTTP status, not the body: Brandfolder's `default` error response
is a bare string on 72 of 73 operations, with no code and no structure.

- `400` — a misspelled query parameter, or one not valid for that endpoint.
- `403` — wrong/absent API key, **or the resource was deleted**, **or** the CDN
  URL belongs to an asset that is pending approval or unpublished. Do not read
  403 as "retry with a different token" without checking asset availability.
- `404` — bad resource ID or path.
- `429` — back off exponentially. There is no `Retry-After` header and no
  published limit, so use your own ceiling.

## Do not

- Do not construct an asset, section or collection ID. Always read it from a
  listing response.
- Do not assume `PATCH` exists. The API accepts `GET`, `PUT`, `POST`, `DELETE` only.
