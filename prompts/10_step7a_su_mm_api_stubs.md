# Step 7a: SU MM API route stubs, schemas, and OpenAPI export in nmm-su-mm

Depends on step 6 (`nmm-su-mm` scaffolded from `nmm-base` via `install.sh nmm_su_mm`), step 3a/3b (post-4a release of `nmm-events` and `nmm-event-contracts` with real schemas), step 3c (`nmm-tenancy`), step 3d (`nmm-auth`).

Read v16 in full, with particular attention to sections 6 (Entity ownership), 7 (Booking and item record split), 8 (Cross-API event catalogue), 13 (Acceptance, venue acceptance, right of query), 16 (Field-level completeness), 17 (Approvals and approval rules, in the `(entity_type, from_state, to_state)` form), 19 (Notifications), 31 (Authorisation and tenant scoping), 33-37 (inventory tiers, Tier 2, Active Weeks, asset service-level evidence, fair calendar), 41 (Frontends). Then `nmm_build_plan_v14.md` sections "SU MM API" (full endpoint surface) and "Contract artefacts" subsection on `su-mm-api.openapi.json`.

## Pre-flight: verify the build plan's endpoint enumeration is complete

Before writing route stubs, scan v16 for every endpoint the SU MM API needs to serve and compare against the build plan's "SU MM API" → "Endpoints" subsections. If v16 implies endpoints not in the build plan (e.g. missing CRUD on a canonical entity, missing transition endpoints), list them in `MISSING_ENDPOINTS.md` at the repo root and stop. Do not silently invent endpoints; do not silently omit ones v16 implies.

If the enumeration is complete, proceed.

## Scope

Write the FastAPI route stubs and Pydantic request/response schemas for every endpoint enumerated in the build plan's SU MM API section. Run the OpenAPI export script (inherited from `nmm-base` via `base_app`). Commit the resulting `su-mm-api.openapi.json` at the repo root.

The OpenAPI document is not hand-written. It is exported from FastAPI. The source of truth is the FastAPI route signatures and Pydantic schemas; the `.json` file is the build artefact.

## Out of scope

Endpoint implementations. Every route stub returns HTTP 501 with body `{"error": "not_implemented"}`. Database models, services, business logic, the listener wiring, the consumer wiring, the SPA. Those are step 8.

## What to write

For every endpoint in the build plan's "SU MM API" → "Endpoints" subsections (Inventory, Fairs, Bookings, Acceptance, Evidence, Stats, SU content approvals, Design requests where SU is approver per v16 section 39 (read access plus approval grant/reject/amends-requested), Campaign plans visibility (per v16 section 23a), Publisher staff users, Publisher contacts, Active Weeks, Notifications, Messaging hub):

A FastAPI route handler with:

- The HTTP method and path matching the build plan
- Path, query, and body parameters typed via Pydantic models
- A response model typed via Pydantic
- An auth dependency from `nmm-auth`. Default for endpoints serving publisher contacts only: `require_user("nmm-su-mm", {"publisher_staff"})`. For endpoints that admit Native staff bypass per v16 section 31 (e.g. inventory tier classification editable from the Delivery Manager admin UI calling SU MM endpoints, retained advertisers admin, fair calendar entries reviewable cross-publisher): `require_user("nmm-su-mm", {"publisher_staff", "native_staff"})`. Read v16 section 31 to determine which endpoints admit both; default to publisher_staff only when v16 doesn't explicitly call out a Native admin path.
- A docstring summarising the endpoint's purpose, citing the relevant v16 section
- Body returns `JSONResponse(status_code=501, content={"error": "not_implemented"})`

For every Pydantic schema:

- Field types matching v16's specification of the entity
- Required vs optional fields per v16
- Constraints (min/max length, format, enum) where v16 specifies them
- Field docstrings citing the relevant v16 section

Approval-related schemas use the `(entity_type, from_state, to_state)` model per v16 section 17. Do not use any earlier `approval_type` model.

Where v16 underspecifies a field, match the canonical entity shape from the relevant section and append to `UNDERSPECIFIED.md` at the repo root.

## Required modules

Stub the application skeleton enough to make `app.openapi()` produce a valid document:

- `app/main.py`: FastAPI app instantiated with title `nmm-su-mm`, version from pyproject, the routers below mounted, an exception handler translating `nmm_tenancy.exceptions.TenantScopeViolation` to 404
- `app/routers/`: one router file per endpoint group (inventory.py, fairs.py, bookings.py, evidence.py, stats.py, approvals.py, design_requests.py, campaign_plans.py, users.py, contacts.py, active_weeks.py, notifications.py, messaging.py, publishers.py)
- `app/schemas/`: Pydantic schemas grouped by entity (booking.py, item.py, evidence.py, stats.py, etc)
- `app/dependencies.py`: thin module that imports `require_user` and `require_internal_service` from `nmm-auth` and re-exports with the SU MM audience pre-bound, plus helpers for the two common allowed_user_types sets

No services, no models beyond what the schemas need, no listener wiring beyond a placeholder. The listener gets wired in step 8.

## OpenAPI export

Run `python scripts/export_openapi.py` (inherited from `base_app` via `nmm-base`). Output: `su-mm-api.openapi.json` at the repo root. CI gate (also inherited) passes against the committed file.

## Tests

For every router: a test that hits each endpoint with a valid auth header and asserts a 501 response. A test that hits with no auth and asserts 401. A test that hits with the wrong audience and asserts 401. For endpoints that admit both publisher_staff and native_staff, a test for each user_type asserting 501 (not 401).

A test that loads `su-mm-api.openapi.json` and asserts it contains every routing path enumerated in the build plan.

## Acceptance

`MISSING_ENDPOINTS.md` is empty or absent. The FastAPI app starts. `python scripts/export_openapi.py` produces a valid `su-mm-api.openapi.json` containing every endpoint from the build plan. The CI gate passes. Every endpoint returns 501 under valid auth, 401 otherwise. `UNDERSPECIFIED.md` is reviewed before the document is treated as final. The exported document is the contract the SU Media Manager SPA's TypeScript client generates from in step 8.
