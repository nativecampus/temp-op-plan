# Step 3d: nmm-auth

Read v16 sections 30 (Identity and authentication) and 31 (Authorisation and tenant scoping), and the existing `nmm` repo's JWT validation code under `app/dependencies.py` and any related auth services. Then `nmm_build_plan_v14.md` section "`nmm-auth` repo" and work order step 3.

## Scope

Build `nmm-auth` as a Python package: JWT validation, custom claim block parsing, dependency factories, role taxonomies. The Delivery API, SU MM API, and Advertiser API all depend on it.

`nmm-auth` does not own permission evaluation. The role permission matrix per v16 section 31 lives in each application repo because it sits on the role-permissions admin tables, not in code. Step 5's retrofit of `nmm` does not move existing permission-checking code into this package; it only swaps the JWT primitive and `require_user`.

## Module layout

```
nmm_auth/
  __init__.py
  jwt.py            # JWT validation primitive (signature, expiry, audience)
  claims.py         # Custom claim block parsing
  dependencies.py   # require_user, require_internal_service, require_auth0_action_secret factories
  taxonomies.py     # role taxonomies per audience
  exceptions.py     # AuthError
```

## JWT validation

`validate_jwt(token, audience)`: validates signature against Auth0's public key (cached, refresh on key rotation), checks expiry, checks audience matches. Returns the decoded claims. Raises `AuthError` on any failure.

## Custom claim block

The JWT carries a custom claim block under a configurable claim name (default `https://native.fm/nmm`). Block contents per v16 section 30:

- `auth0_user_id`
- `email`
- `user_type`: one of `native_staff`, `publisher_staff`, `advertiser_contact`
- `roles`: array for native_staff, single role string for publisher_staff and advertiser_contact
- `tenant_scope`: `publisher_id` (int) for publisher_staff, `advertiser_ids` (array of int) for advertiser_contact, absent for native_staff
- `unrestricted_access`: boolean (native_staff only)

`parse_claims(decoded_jwt)` returns a typed `Claims` object. Raises `AuthError` if the custom claim block is missing or malformed.

## require_user dependency factory

```python
def require_user(audience: str, allowed_user_types: set[str]) -> Callable: ...
```

Returns a FastAPI dependency that:

1. Reads the bearer token from the `Authorization` header
2. Validates the JWT against the audience
3. Parses the claims
4. Checks `user_type` is in `allowed_user_types`
5. Looks up the user record in the API's local database (the canonical user record on the Delivery API, the projection on the others). Raises `AuthError` if absent or inactive.
6. Sets the `tenant_scope` contextvar from `nmm-tenancy` based on `user_type` and claims (publisher scope for publisher_staff, advertiser scope for advertiser_contact, native_bypass for native_staff)
7. Returns a `CurrentUser` object with the parsed claims and the user record

The dependency does not check role permissions. Each application repo declares per-route permission requirements and evaluates them against the `CurrentUser` using its own permission service, which reads the role-permissions admin tables.

## require_internal_service dependency

```python
def require_internal_service(audience: str = "nmm-internal") -> Callable: ...
```

Validates a service-to-service JWT (no user context, no tenant scope). Used by the cross-API outbox replay endpoints per v16 section 9.

## require_auth0_action_secret dependency

Validates the `X-Auth0-Action-Secret` header against an env var. Used by `GET /internal/users/{auth0_user_id}/claims`, called by Auth0 at token issuance.

## taxonomies.py

Role taxonomies per audience:

- `nmm-delivery`: native_staff roles per v16 section 3 (`campaign_delivery`, `campaign_planning`, `logistics_and_warehousing`, `experiential_pm`, `creative_production`, `staffing`, `sales_and_account_management`, `tech_nexus`, `engineering`, `operational_support`)
- `nmm-su-mm`: publisher_staff roles per v16 section 31 (`account_owner`, `commercial_manager`, `marketing_delivery_contact`, `read_only`)
- `nmm-advertiser`: advertiser_contact roles per v16 section 31 (`primary_contact`, `additional_contact`)

Each audience also accepts `native_staff` as a valid `user_type` because Native staff invoke all three APIs in admin support paths. The taxonomy validates the role string against the appropriate set based on `user_type`, not audience.

Unknown roles raise `AuthError`.

## Tests

Unit: signature validation, expiry, audience match, claim block parsing happy path, claim block parsing malformed (every malformation in turn), `require_user` rejecting wrong audience, rejecting wrong user_type, rejecting unknown role, accepting valid JWT, setting the contextvar correctly per scope type including the native_bypass case for a native_staff user calling SU MM or Advertiser API. Service-to-service JWT validation. Auth0 Action secret validation.

## Publishing

Standard `pyproject.toml`. Published to the private package registry on every tagged release.

## Acceptance

Tests pass. The package builds and publishes a 0.1.0 release. A sample FastAPI app using `require_user("nmm-su-mm", {"publisher_staff", "native_staff"})` accepts both a publisher_staff token and a native_staff token, rejects an advertiser_contact token, and sets the tenant_scope contextvar to publisher scope or native_bypass appropriately.
