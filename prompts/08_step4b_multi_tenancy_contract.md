# Step 4b: multi-tenancy-contract.md in nmm-tenancy

Depends on step 3c (`nmm-tenancy` package published).

Read v16 sections 30 (Identity and authentication) and 31 (Authorisation and tenant scoping). Then `nmm_build_plan_v14.md` section "Contract artefacts" subsection on `multi-tenancy-contract.md`, and the existing placeholder file at the `nmm-tenancy` repo root.

## Scope

Replace the placeholder in `nmm-tenancy` with the full contract document. The document is the canonical specification of the listener pattern that the SU MM and Advertiser APIs implement identically.

## Document structure

The document covers, in order:

The contextvar. `tenant_scope` set by the JWT validation dependency on every request. The three values:

- `{"type": "publisher", "publisher_id": <int>}` for publisher_staff users on the SU MM API
- `{"type": "advertiser", "advertiser_ids": [<int>, ...]}` for advertiser_contact users on the Advertiser API (an array because users may be primary or additional contacts across multiple advertisers)
- `{"type": "native_bypass"}` for native_staff users invoking these APIs (admin support paths)

Tenant-scoped tables. The full lists from v14's SU MM API and Advertiser API sections. Every table named, with its tenant column. The single integer column (`publisher_id` or `advertiser_id`) is populated by the consumer worker from event payload tenant identity, never by an HTTP request handler.

The listener behaviour. SQLAlchemy `before_compile` event listener intercepts every Select, Update, and Delete against any tenant-scoped table.

- For Selects with `PublisherScope`: injects `<tenant_column> = :publisher_id` into the where clause
- For Selects with `AdvertiserScope`: injects `<tenant_column> = ANY(:advertiser_ids)` into the where clause
- For Updates and Deletes: with `PublisherScope`, the row's tenant column must equal `publisher_id`; with `AdvertiserScope`, the row's tenant column must be in `advertiser_ids`. Mismatch raises `TenantScopeViolation`.
- For `NativeBypassScope`: skips injection on Selects and skips the check on Updates/Deletes

HTTP response shape. Out-of-tenant Selects return 404 (the tenant filter excludes the row, so the query returns nothing, which the endpoint translates to 404). Out-of-tenant Updates and Deletes raise `TenantScopeViolation` which the endpoint translates to 404. Never 403, because the existence of out-of-tenant data must not be visible.

The required test set. Every consuming API runs the cross-tenant isolation fixture from `nmm_tenancy.fixtures`:

- Two publishers (or two advertisers) created with rows in every tenant-scoped table
- Query-as-A returns only A's data on every list endpoint
- Query-as-A with B's row ID returns 404 on every detail endpoint
- Update-as-A targeting B's row returns 404
- Delete-as-A targeting B's row returns 404
- Multi-advertiser user (advertiser API only) sees both A's and B's data
- A native_bypass user sees both

The fixture lives in each application repo at `scripts/fixtures/cross_tenant_isolation.py`, importing from `nmm_tenancy.fixtures`. It runs as part of the application repo's harness.

The contextvar shim for the Delivery API. The Delivery API has no tenant-scoped tables but depends on `nmm-tenancy` for the contextvar shim that the `nmm-events` consumer worker uses to set tenant identity on outbound events. Document the explicit `set_native_bypass()` call pattern for HTTP requests and RQ jobs.

The flow at request entry. JWT validation dependency reads claims, computes scope, sets contextvar. Every database operation in that request runs under the listener. Endpoint exception handler translates `TenantScopeViolation` to 404.

The flow at outbound event publish. The relay reads the contextvar at publish time and includes the tenant identity in the event payload metadata (per the schemas in step 4a). Consumers use the included tenant identity to populate their projection rows' tenant column without trusting any HTTP request handler to do so.

## Out of scope

The package code (already in step 3c). The fixtures (already in step 3c).

## Acceptance

The document is reviewable as a self-contained specification: an implementing agent reads it and can produce a conforming listener-using API without reading v16. A new tagged release of `nmm-tenancy` ships with the document.
