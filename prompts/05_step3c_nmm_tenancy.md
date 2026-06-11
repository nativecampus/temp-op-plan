# Step 3c: nmm-tenancy

Read v16 sections 30 (Identity and authentication) and 31 (Authorisation and tenant scoping), and `nmm_build_plan_v14.md` section "`nmm-tenancy` repo" and work order step 3.

## Scope

Build `nmm-tenancy` as a Python package: contextvar, SQLAlchemy listener, exceptions, test fixtures. The `multi-tenancy-contract.md` document is step 4b and not part of this prompt; this step delivers the runtime package and an empty placeholder for the contract document.

## Module layout

```
nmm_tenancy/
  __init__.py
  contextvar.py     # the tenant_scope contextvar and helpers
  listener.py       # the SQLAlchemy before_compile event listener
  exceptions.py     # TenantScopeViolation
  fixtures.py       # the cross-tenant test fixture every consuming API runs
```

## contextvar.py

`tenant_scope: ContextVar[TenantScope]` with helpers `set_publisher(publisher_id)`, `set_advertiser(advertiser_ids)`, `set_native_bypass()`, `get_current()`, `clear()`.

`TenantScope` is a typed union: `PublisherScope`, `AdvertiserScope`, `NativeBypassScope`. Per v16 section 30:

- `PublisherScope`: `{"type": "publisher", "publisher_id": int}`
- `AdvertiserScope`: `{"type": "advertiser", "advertiser_ids": list[int]}`
- `NativeBypassScope`: `{"type": "native_bypass"}`

`set_advertiser` accepts a list because v16 section 30 specifies the JWT carries an `advertiser_ids` array claim: an advertiser_contact user can be primary or additional contact across multiple advertisers within an agency relationship. The list is preserved through the contextvar so the listener can use `ANY()` matching.

## listener.py

`register_tenant_listener(metadata, tenant_scoped_tables)` registers the SQLAlchemy `before_compile` event handler. Tenant-scoped tables are passed in by each consuming API as a list of `(table, tenant_column_name)` tuples.

The listener:

- For Selects against a registered tenant-scoped table where the contextvar holds `PublisherScope`: injects `<tenant_column> = :publisher_id` into the where clause
- For Selects with `AdvertiserScope`: injects `<tenant_column> = ANY(:advertiser_ids)` into the where clause
- For Updates and Deletes: checks the targeted row's tenant column against the contextvar. With `PublisherScope`, the value must equal `publisher_id`. With `AdvertiserScope`, the value must be in `advertiser_ids`. Mismatch raises `TenantScopeViolation`.
- For `NativeBypassScope`: skips injection on Selects and skips the check on Updates/Deletes

Tenant column types. Each consuming API's projection tables have a single integer tenant column (`publisher_id` on the SU MM API's tables, `advertiser_id` on the Advertiser API's tables). The consumer worker populates the column from the event payload's tenant identity (per v16 section 8 and step 4a's schemas). Multi-advertiser users querying see rows for any of their advertiser_ids; writes to a row require the row's `advertiser_id` to be in the user's claim list.

## exceptions.py

`TenantScopeViolation`. Application repos catch this in a FastAPI exception handler and return 404 per v16 section 31.

## fixtures.py

`cross_tenant_isolation_fixture(api_type)` returns a pytest fixture that creates two tenants of the appropriate type (two publishers or two advertisers) each with rows in every tenant-scoped table on the API. For `api_type='advertiser'` the fixture also creates a third user whose claim covers both advertisers (the multi-advertiser case) and asserts that user sees both, while users scoped to a single advertiser see only their own.

The standard test set every consuming API must pass:

- query-as-A returns only A's data on every list endpoint
- query-as-A with B's row ID returns 404 on every detail endpoint
- update-as-A targeting B's row returns 404
- delete-as-A targeting B's row returns 404
- query-as-multi-advertiser-user returns both A's and B's data on every list endpoint (advertiser API only)
- `native_bypass` user sees both

Each consuming API's repo will have `scripts/fixtures/cross_tenant_isolation.py` importing from this package.

## Placeholder for the contract document

Add `multi-tenancy-contract.md` at the repo root with the heading and a stub: `# Multi-tenancy contract` followed by `<!-- Step 4b fills this in. The contract specifies the listener pattern that the SU MM and Advertiser APIs implement identically. -->`. Step 4b writes the actual document.

## Tests

Unit: listener Select injection (publisher and advertiser scopes), Update mismatch raises, Delete mismatch raises, native_bypass skips, multi-advertiser ANY matching. Integration against a sample SQLAlchemy model with two tenants and one multi-advertiser user verifying isolation across the test set above.

## Publishing

Standard `pyproject.toml`. Published to the private package registry on every tagged release.

## Out of scope

The contract document content. That is step 4b.

## Acceptance

Tests pass. The fixture imported into a sample consuming API verifies cross-tenant isolation across all test cases including the multi-advertiser case. The package builds and publishes a 0.1.0 release.
