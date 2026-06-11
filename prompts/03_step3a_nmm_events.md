# Step 3a: nmm-events

Read v16 sections 8 (Cross-API event catalogue) and 9 (Consistency, idempotency, replay), and `nmm_build_plan_v14.md` sections "`nmm-events` repo" and the work order step 3. Then read the existing `nmm` repo's `app/services/outbox/`, `app/workers/`, `app/models/` for any outbox or dead-letter table, and any RQ worker code that publishes outbox rows or consumes them.

## Scope

`nmm-events` is the runtime cross-API event package. Outbox model, relay worker, event consumer worker, schema validator, dispatch decorator, dead-letter handling, retry. Published as a Python package on every tagged release.

The JSON Schema documents themselves live in a separate repo (`nmm-event-contracts`, step 3b). The `SchemaRegistry` in `nmm-events` takes an injectable loader callable at construction so 3a and 3b can develop in parallel. Application repos wire the production loader (`nmm_event_contracts.loader.load_all_schemas`) when they construct the registry.

This step ships:

- The runtime code (outbox model, relay, consumer, schema registry, dispatch, dead-letter, retry)
- CI for the package
- Refactored runtime code from the existing `nmm` repo, with the table shape diffed first

The actual schemas for v16 events are step 4a (in `nmm-event-contracts`); this step ships the runtime infrastructure.

## Outbox table shape: diff first

Before writing the `OutboxModel` mixin, read the existing nmm outbox table's columns, types, indexes, and constraints (in `app/models/outbox.py` and the relevant alembic migration). The mixin must accommodate what nmm has, not the other way around. If the existing table's column shape conflicts with what's documented in the build plan, raise the differences as a `DIFFS.md` at the repo root listing each column delta and proceed with the existing shape; do not change nmm's table in this PR.

The existing dead-letter table (if it exists in nmm) gets the same diff-first treatment for `DeadLetterEventsModel`.

## Module layout

```
nmm_events/
  __init__.py
  outbox/
    model.py          # OutboxModel SQLAlchemy mixin
    relay.py          # OutboxRelay worker
  consumer/
    consumer.py       # EventConsumer worker
    consumed_events.py # ConsumedEventsModel mixin
  schemas/
    registry.py       # SchemaRegistry, takes an injectable loader callable
  dispatch.py         # decorator pattern for registering handlers per routing key
  dead_letter.py      # DeadLetterEventsModel mixin and query helpers
  retry.py            # exponential backoff schedule (1s, 5s, 25s, 2m, 10m)
```

## OutboxModel

SQLAlchemy mixin built to match nmm's existing table per the diff-first step. Standard event columns: `id`, `aggregate_type`, `aggregate_id`, `event_id` (UUID), `event_type`, `schema_version`, `occurred_at`, `published_at` (nullable), `payload` (JSONB), `publisher` (api name), `routing_key`, `dead_lettered_at` (nullable), `discarded_at` (nullable), audit columns. Each consuming API extends the mixin with its tenant column where applicable.

Index on `(published_at IS NULL, dead_lettered_at IS NULL, discarded_at IS NULL)` for the relay's pending-row scan.

## OutboxRelay

RQ-based worker. Reads pending rows in `id` order per `aggregate_id` (per-aggregate ordering preserved). For each row: validates payload against the schema for the routing key via `SchemaRegistry`, publishes to RabbitMQ on the configurable topic exchange (default `nmm.events`), marks the row as published on success.

Bounded retry per row: 1s, 5s, 25s, 2m, 10m. After exhaustion the row is marked dead-lettered.

Configurable via env vars: RabbitMQ URL, exchange name, batch size, polling interval.

The relay reads tenant identity from the `nmm-tenancy` contextvar at publish time and includes it in the event envelope (`publisher_id` or `advertiser_id` field where applicable per the multi-tenancy contract).

## EventConsumer

RQ-based worker. Subscribes to declared routing keys. On receipt: validates incoming payload against the schema for the routing key, dispatches to the registered handler.

Idempotency: per-API `consumed_events` table (`id`, `publisher`, `event_id`, `consumed_at`, indexed by `(publisher, event_id)`, TTL 7 days). The consumer checks before dispatch and skips repeats.

Per-handler retry on exception per the same retry schedule. Dead-letter on exhaustion.

## SchemaRegistry: strict only

Builds an in-memory map from routing key to schema document keyed by `schema_version`. Exposes `validate(routing_key, payload)` raising on failure with the validation error inline.

The registry takes an injectable loader callable at construction:

```python
def SchemaRegistry(loader: Callable[[], dict[str, dict[int, dict]]]) -> SchemaRegistry: ...
```

In production, application repos pass `nmm_event_contracts.loader.load_all_schemas` (from step 3b's `nmm-event-contracts` package). In `nmm-events`'s own tests, the loader is a fixture that returns a small in-memory dict. This decouples 3a from 3b at build time so they can be developed in parallel; the runtime dependency (application repos pinning `nmm-event-contracts`) is set up in step 5 (retrofit) and steps 7a/7b (new app repos).

The registry is strict: a routing key with no schema, or a `schema_version` not present for a routing key, raises. There is no permissive mode and no escape hatch flag.

## dispatch.py

Decorator pattern:

```python
from nmm_events.dispatch import handler

@handler("delivery.campaign_booking.state_changed")
def on_booking_state_changed(event): ...
```

The consumer worker reads the registry of decorated functions at startup and subscribes to those routing keys.

## dead_letter.py

`DeadLetterEventsModel` mixin built to match nmm's existing table per the diff-first step. Columns: `id`, source `publisher`, original `event_id`, `routing_key`, `payload`, `last_error`, `retry_count`, `dead_lettered_at`. Query helpers: `list_pending`, `replay(id)` (re-enqueues), `discard(id)`.

## Outbox replay

Paginated read interface against an outbox table: list rows by routing key and event_id range. Application repos expose this via service-to-service-authenticated endpoints per v16 section 9.

## CI

On every PR: builds the package, runs unit and integration tests. A version bump is required on any change to `nmm_events/`. CI enforces by comparing `pyproject.toml` version against `main`.

## Tests

Unit: outbox writes, relay publishing happy path, relay retry on RabbitMQ failure, dead-letter on exhaustion, consumer dispatch, idempotency on repeat, retry on handler exception, schema validation failure stops publish, schema version mismatch on consume raises, missing schema raises.

Integration with a local RabbitMQ container and a placeholder schema in a test fixture: end-to-end publish-consume round trip, validating against the placeholder schema in strict mode.

## Publishing

Standard `pyproject.toml`. Published to the private package registry on every tagged release. Tag format `v<semver>`. Application repos pin by exact version.

## Out of scope

The JSON Schema documents themselves. The package infrastructure for `nmm-event-contracts`. Both are step 3b. The schemas for v16 events are step 4a.

## Acceptance

Tests pass. The package builds and publishes a 0.1.0 release. A hello-world consumer using the dispatch decorator receives an event published by a hello-world producer using `OutboxRelay` across a local RabbitMQ, validating against a test-fixture schema in strict mode.
