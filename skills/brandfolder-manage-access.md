---
name: Grant and audit Brandfolder access
description: Invite users to an organization, Brandfolder or collection, then audit and revoke the permissions those invitations produced.
api: openapi/brandfolder-openapi-original.yml
operations:
  - opIdApiV4OrganizationsGet
  - opIdApiV4OrganizationsInvitationsByOrganizationIdGet
  - opIdApiV4OrganizationsInvitationsByOrganizationIdPost
  - opIdApiV4BrandfoldersInvitationsByBrandfolderIdGet
  - opIdApiV4BrandfoldersInvitationsByBrandfolderIdPost
  - opIdApiV4CollectionsInvitationsByCollectionIdGet
  - opIdApiV4CollectionsInvitationsByCollectionIdPost
  - opIdApiV4InvitationsByIdGet
  - opIdApiV4InvitationsByIdDelete
  - opIdApiV4OrganizationsUserPermissionsByOrganizationIdGet
  - opIdApiV4BrandfoldersUserPermissionsByBrandfolderIdGet
  - opIdApiV4CollectionsUserPermissionsByCollectionIdGet
  - opIdApiV4UserPermissionsByIdGet
  - opIdApiV4UserPermissionsByIdDelete
generated: '2026-08-13'
method: generated
source: openapi/brandfolder-openapi-original.yml
---

# Grant and audit Brandfolder access

Base URL `https://brandfolder.com/api/v4`, `Authorization: Bearer {BF_API_KEY}`.

Access in Brandfolder is two objects, not one:

- an **Invitation** is the offer — `email`, `permission_level`,
  `personal_message`, `invitation_url`
- a **UserPermission** is the resulting grant — created when the invitation is
  accepted, and the thing you must revoke to actually remove access

Deleting an invitation does not revoke an accepted grant. Audits that only read
invitations miss everyone who already joined.

## Invite

Choose the narrowest scope that does the job:

| Scope | Operation |
|---|---|
| Organization | `opIdApiV4OrganizationsInvitationsByOrganizationIdPost` → `POST /organizations/{organization_id}/invitations` |
| Brandfolder | `opIdApiV4BrandfoldersInvitationsByBrandfolderIdPost` → `POST /brandfolders/{brandfolder_id}/invitations` |
| Collection | `opIdApiV4CollectionsInvitationsByCollectionIdPost` → `POST /collections/{collection_id}/invitations` |
| Portal | `opIdApiV4PortalsInvitationsByPortalIdPost` → `POST /portals/{portal_id}/invitations` |
| Brandguide | `opIdApiV4BrandguidesInvitationsByBrandguideIdPost` → `POST /brandguides/{brandguide_id}/invitations` |

Note on the last two: the API can invite to Portals and Brandguides, but
publishes **no Portal or Brandguide schema and no listing operation**, so you
cannot discover a `portal_id` or `brandguide_id` through the API. You must
obtain those out of band from the web UI.

Read back with the matching `...InvitationsBy...Get` operation, or
`opIdApiV4InvitationsByIdGet` → `GET /invitations/{invitation_id}` for one.
Rescind an unaccepted offer with `opIdApiV4InvitationsByIdDelete` →
`DELETE /invitations/{invitation_id}`.

## Audit

Enumerate grants at each scope:

- `opIdApiV4OrganizationsUserPermissionsByOrganizationIdGet` →
  `GET /organizations/{organization_id}/user_permissions`
- `opIdApiV4BrandfoldersUserPermissionsByBrandfolderIdGet` →
  `GET /brandfolders/{brandfolder_id}/user_permissions`
- `opIdApiV4CollectionsUserPermissionsByCollectionIdGet` →
  `GET /collections/{collection_id}/user_permissions`

These are the only schemas in the API that declare a `relationships` block —
use `?include=user` to resolve `email`, `first_name`, `last_name` in one pass
rather than fetching users individually.

Start from `opIdApiV4OrganizationsGet` → `GET /organizations` and walk down;
there is no single "all permissions" endpoint.

## Revoke

`opIdApiV4UserPermissionsByIdDelete` → `DELETE /user_permissions/{user_permission_id}`

Confirm the target first with `opIdApiV4UserPermissionsByIdGet` →
`GET /user_permissions/{user_permission_id}`. This is destructive and there is
no undo, no soft delete and no restore operation.

## Why the API key itself matters here

The bearer key carries **no scopes**. It inherits the issuing user's permissions
verbatim, so a key created by an org admin can invite and revoke across the
whole organization. Least privilege is achieved by issuing the key from a
restricted user account — there is no way to narrow an existing key.

Treat any automation holding an admin-issued key as an admin. Rotation is
manual: generate a new key at `https://brandfolder.com/profile#integrations`;
Brandfolder publishes no rotation, expiry or revocation API.

## Errors

- `400` — missing `email` or `permission_level` in the invitation body.
- `403` — the key's user is not an admin at that scope, or the resource was deleted.
- `404` — wrong scope ID.
- `429` — back off; no `Retry-After` is returned.

Bulk invitations have no idempotency key — a retried invite is a second
invitation. List before you re-send.
