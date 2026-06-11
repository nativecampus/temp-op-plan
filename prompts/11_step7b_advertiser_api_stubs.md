# Step 7b: Advertiser API route stubs, schemas, and OpenAPI export in nmm-advertiser

Depends on step 6 (`nmm-advertiser` scaffolded from `nmm-base` via `install.sh nmm_advertiser`), step 3a/3b (post-4a release of `nmm-events` and `nmm-event-contracts` with real schemas), step 3c (`nmm-tenancy`), step 3d (`nmm-auth`).

Read v16 in full, with particular attention to sections 6 (Entity ownership), 7 (Booking and item record split), 8 (Cross-API event catalogue), 16 (Field-level completeness), 17 (Approvals and approval rules, in the `(entity_type, from_state, to_state)` form), 17a-e (templates, artefacts, attachments, external links, people-on-record), 19 (Notifications), 23a (Campaign plans), 31 (Authorisation and tenant scoping), 39 (Design requests), 41 (Frontends). Then `nmm_build_plan_v14.md` sections "Advertiser API" (full endpoint surface, including Publisher Widget public endpoints) and "Contract artefacts" subsection on `advertiser-api.openapi.json`.

## Pre-flight: verify the build plan's endpoint enumeration is complete

Same pattern as step 7a. Scan v16 for every endpoint the Advertiser API needs to serve (including widget public endpoints). If v16 implies endpoints not in the build plan, list them in `MISSING_ENDPOINTS.md` at the repo root and stop. Do not silently invent endpoints; do not silently omit.

If the enumeration is complete, proceed.

## Scope

Write the FastAPI route stubs and Pydantic request/response schemas for every endpoint enumerated in the build plan's Advertiser API section, including the Publisher Widget public endpoints. Run the OpenAPI export script. Commit the resulting `advertiser-api.openapi.json` at the repo root.

The OpenAPI document is not hand-written. It is exported from FastAPI.

## Out of scope

Endpoint implementations. Every route stub returns HTTP 501 with body `{"error": "not_implemented"}`. Database models, services, business logic, the listener wiring, the consumer wiring, the Advertiser Portal SPA, the Publisher Widget script bundle. Those are step 8.

The captcha provider integration and the edge rate-limit configuration. The widget endpoints' route handlers declare placeholder dependencies for these (`captcha_required`, `rate_limited_by_publisher_slug`); step 8 implements them. The build plan's "Publisher Widget public endpoints" subsection specifies that rate limiting is at the edge by `publisher_slug` and that captcha applies on enquiry submission. Step 8 reads that and implements; this step only declares the FastAPI dependencies as placeholders so the OpenAPI document marks the security/throttling appropriately.

## Authenticated endpoints

For every endpoint in the build plan's "Advertiser API" → "Endpoints" subsections (Advertiser, Advertiser contacts, Bookings, Advertiser-supplied content, Advertiser-level documents, Campaign plans (read access plus advertiser approval grant/reject/amends-requested per v16 section 23a), Design requests (read access plus advertiser approval where the advertiser is sign-off contact per v16 section 39), Advertiser approvals, Self-serve booking, Notifications, Messaging hub):

A FastAPI route handler with:

- The HTTP method and path matching the build plan
- Path, query, and body parameters typed via Pydantic models
- A response model typed via Pydantic
- Auth dependency from `nmm-auth`. Default: `require_user("nmm-advertiser", {"advertiser_contact"})`. For endpoints that admit Native staff bypass per v16 section 31 (e.g. advertiser data admin paths called from the Delivery Manager): `require_user("nmm-advertiser", {"advertiser_contact", "native_staff"})`. Read v16 section 31 to determine which endpoints admit both; default to advertiser_contact only when v16 doesn't explicitly call out a Native admin path.
- A docstring citing the relevant v16 section
- Body returns 501 as above

Approval-related schemas use the `(entity_type, from_state, to_state)` model per v16 section 17.

## Publisher Widget public endpoints

Per the build plan's "Publisher Widget public endpoints" subsection:

- `GET /widget/{publisher_slug}/inventory`
- `GET /widget/{publisher_slug}/products`
- `GET /widget/{publisher_slug}/styling`
- `POST /widget/{publisher_slug}/enquiries`
- `POST /widget/{publisher_slug}/enquiries/{id}/verify-email`

These have no JWT, no `require_user`. They take placeholder dependencies for rate-limit (all five) and captcha (the two POSTs only):

```python
def rate_limited_by_publisher_slug(publisher_slug: str) -> None:
    """Placeholder. Step 8 implements edge rate limiting per the build plan's Publisher Widget public endpoints subsection."""
    pass

def captcha_required(captcha_token: str = Header(...)) -> None:
    """Placeholder. Step 8 implements captcha verification per the build plan's Publisher Widget public endpoints subsection."""
    pass
```

The OpenAPI document marks these endpoints with no security requirement (no JWT) but documents the captcha header parameter on the two POSTs.

The widget enquiry endpoints emit `advertiser.enquiry.received` and `advertiser.enquiry.email_verified` events on success per step 4a's schemas. The stubs do not emit; step 8 wires the outbox writes.

The self-serve booking submit endpoint (`POST /self-serve-bookings/{id}/submit`) per v16 section 6 and the build plan synchronously calls HubSpot's REST API. The stub returns 501; step 8 implements the synchronous call.

## Pydantic schemas

For every entity surfaced by these endpoints:

- Field types matching v16's specification
- Required vs optional fields per v16
- Constraints per v16
- Field docstrings citing the relevant v16 section

The advertiser content satellite (v16 section 7), the advertiser-level document store (section 41), the self-serve booking state (section 41), the enquiries store (section 41 and the build plan's Advertiser API section), and every projection table the API exposes.

Where v16 underspecifies a field, match the canonical entity shape and append to `UNDERSPECIFIED.md`.

## Required modules

- `app/main.py`: FastAPI app titled `nmm-advertiser`, with the `TenantScopeViolation` exception handler
- `app/routers/`: one router per endpoint group, plus `widget.py` for the public widget endpoints
- `app/schemas/`: Pydantic schemas grouped by entity
- `app/dependencies.py`: thin module re-exporting `nmm-auth` dependencies pre-bound to the advertiser audience, plus the widget rate-limit and captcha placeholder dependencies

## OpenAPI export

`python scripts/export_openapi.py` produces `advertiser-api.openapi.json` at the repo root. The CI gate passes against the committed file.

The widget endpoints appear in the same OpenAPI document, tagged appropriately so the Publisher Widget can generate against the `widget` tag and the Advertiser Portal SPA can generate against everything else.

## Tests

For every authenticated router: a test that hits each endpoint with a valid auth header and asserts 501. A test that hits with no auth and asserts 401. A test that hits with the wrong audience and asserts 401. For endpoints that admit both advertiser_contact and native_staff, a test for each user_type asserting 501.

For widget routes: a test that hits each endpoint without auth and asserts 501 (not 401, because they are unauthenticated). A test that confirms the OpenAPI document marks them with no security requirement. A test that the captcha header is documented on the two POSTs.

A test that loads `advertiser-api.openapi.json` and asserts it contains every routing path enumerated in the build plan, including all widget paths.

## Acceptance

`MISSING_ENDPOINTS.md` is empty or absent. The FastAPI app starts. `python scripts/export_openapi.py` produces a valid `advertiser-api.openapi.json` containing every endpoint and every widget endpoint from the build plan. The CI gate passes. Every authenticated endpoint returns 501 under valid auth, 401 otherwise. Every widget endpoint returns 501 without auth. `UNDERSPECIFIED.md` is reviewed before the document is treated as final. The exported document is the contract both the Advertiser Portal SPA and the Publisher Widget generate clients from in step 8.
