# Step 4a: Event schemas in nmm-event-contracts

Depends on step 3b (`nmm-event-contracts` package infrastructure published).

Read v16 section 8 (Cross-API event catalogue) in full, plus every section it references for payload shapes (sections 7, 10, 13, 14, 15, 16, 17, 17a, 17b, 17c, 17d, 17e, 19, 19a, 20, 21, 23, 23a, 26, 27, 32, 39 at minimum). Then `nmm_build_plan_v14.md` section "Contract artefacts" subsection on `nmm-event-contracts repo`, including the full enumerated list of routing keys.

## Scope

Write the JSON Schema 2020-12 documents for every event listed in the build plan's enumerated routing key list. Populate `registry.json` with one entry per routing key. Delete the placeholder schema added in step 3b.

## Schema shape

Every schema specifies:

- The standard event envelope: `event_id` (UUID), `event_type`, `aggregate_type`, `aggregate_id`, `schema_version`, `occurred_at` (ISO 8601 UTC), `published_at` (ISO 8601 UTC), `publisher` (one of `delivery`, `su_mm`, `advertiser`)
- Tenant identity in the envelope where applicable: `publisher_id` (int) or `advertiser_id` (int) so consumers can populate projection tenant columns without trusting an HTTP request handler. Per the multi-tenancy contract.
- The `payload` object for that event type, with field-level types, required fields, constraints (min/max length, format, enum)

Payload field shapes come from the relevant v16 section. Where v16 underspecifies a payload field, write the schema to match the canonical entity shape from the relevant section (section 7 for booking and item shapes, section 17 for approvals using the `(entity_type, from_state, to_state)` model, section 20 for communications, etc) and append an entry to `UNDERSPECIFIED.md` at the repo root.

## UNDERSPECIFIED.md

A markdown file at the repo root. One section per underspecification:

- The schema file path
- The field name
- What v16 says (or doesn't say)
- What was assumed
- A suggested resolution

The agent does not block on review. Underspecifications are resolved in merge review before tagging the release.

## registry.json

One entry per routing key:

```json
{
  "routing_key": "delivery.campaign_booking.state_changed",
  "current_schema_version": 1,
  "publisher_api": "delivery",
  "consumer_apis": ["su_mm", "advertiser"],
  "consumer_action": "Update campaign_bookings_projection per v16 section 7; recompute outstanding-tasks view per v16 section 19; trigger notification where the recipient is in scope per v16 section 19's notification trigger mapping"
}
```

The `consumer_action` value is a human-readable description of what each consuming API does on receipt. v16 section 8's per-event paragraph and the build plan v14's "Notification triggers per consuming API" section together specify this.

## Routing keys

The enumerated list in `nmm_build_plan_v14.md` section "Contract artefacts" subsection on `nmm-event-contracts repo` is the source. The list includes the Publisher Widget enquiry events `advertiser.enquiry.received` and `advertiser.enquiry.email_verified`. v14 is authoritative for the schema list.

Verify the v14 list against v16 section 8 before starting. If anything in the v14 list is missing from v16 section 8, write the schema (v14 is authoritative) and append the v16 gap to `MISSING_FROM_V16.md` at the repo root for v16 to be updated in follow-up.

For events that depend on entity shapes (e.g. `delivery.campaign_booking.created` needs the booking shape), define a reusable `$defs` block in the relevant schema file. Cross-schema references via `$ref` to `_shared/` are acceptable for entity shapes referenced by multiple events.

## Approval rules

Approval-related event payloads (`delivery.approval.requested`, `granted`, `rejected`, `amendments_requested`, `overridden`) reference the rule key per v16 section 17: `(entity_type, from_state, to_state)`. Schema fields reflect this triple. Do not use any earlier `approval_type` model. The approval payload also carries the optional `against_artefact_version_id` (per v16 section 17b's auto-supersession semantics) and, for `overridden`, the `override_reason` and `overridden_by_user_id`.

## Entity surfaces from v16 cross-cutting patterns

Events for the cross-cutting patterns introduced in v16 sections 17a–e need their own schemas:

- Workflow templates (v16 section 17a): `*_workflow_template.created`, `.version_saved`, `.activated`, `.deactivated` carry the template payload (id, code, type, predicate, version number, summary, stages list with stage definition fields per section 17a).
- Stages (v16 section 17a): `*_stage.opened`, `.completed` (payload includes `completed_against_artefact_versions`, a map from consumed artefact type code to the version current at completion time), `.skipped`, `.reset` (emitted for each stage reset by the amends reset cascade) carry the per-stage payload (parent record id, stage_code, owner_user_id, sla deadline, transition cause, `fires_fsm_transition` binding where set). The parent record's `*.created` event carries the full instantiated stage list inline so consumers can project the stage structure without a separate stage-instantiated event. Stage definitions in those payloads carry `fires_fsm_transition` (a transition triple or empty), required artefact types (consumed), `produces_artefact_types`, `internal_sign_off_pools`, and an optional `approval` configuration `{approver_role, circumstances}` for stage-level sign-offs (per v16 section 17a). Amends publication events on the parent record (per v16 section 17a's round-cap evaluation): `*.amends_published` carries `subject_artefact_type`, `subject_artefact_version_id`, `round_number`, and the cap value at publish time; `*.amends_publish_refused` and `*.amends_over_cap_approval_raised` fire on the corresponding outcomes.
- Internal sign-off ticks (v16 section 17a): `*_stage_signoff.ticked` carries parent record id, stage_code, pool label (operator-defined string), ticking user_id. Sign-offs are workflow-level only; they do not produce approval rows and the schema does not reference the rule engine.
- Artefacts (v16 section 17b): `*_artefact.created`, `.superseded` carry artefact_type_code, version, structured content, attachment_id, supersession reference.
- Attachments (v16 section 17c): `*_attachment.added`, `.removed` carry filename, mime_type, byte_size, storage_url (which may be a platform reference or external URL per v16 section 17c), provenance_type.
- External links (v16 section 17d): `*_external_link.added`, `.removed` carry kind_code, url, label.

The pattern applies to design requests (v16 section 39) and campaign plans (v16 section 23a). Schemas in `_shared/` cover the reusable shapes (stage transition, artefact version, attachment metadata, external link reference, override outcome, nudge metadata).

## Tests

For each schema: a valid example payload validates, an invalid example payload (missing required field, wrong type, out-of-range value) fails validation with the expected error.

For `registry.json`: every routing key listed has a corresponding schema file at the declared current version. Every schema file under `schemas/` has a corresponding registry entry. The consistency check from step 3b's CI catches divergence on every PR.

## Out of scope

Any new strict-mode flag. The registry has been strict-only since step 3a's `SchemaRegistry`.

## Acceptance

A new tagged release of `nmm-event-contracts` ships with all schemas. The validator passes on every schema. `UNDERSPECIFIED.md` and `MISSING_FROM_V16.md` (if any) are reviewed before the release tag. The Python package's loader returns a payload schema for every routing key in `registry.json`. The TypeScript types package exposes a typed payload for every routing key. `nmm-events`'s `SchemaRegistry`, depending on this released version, validates a sample payload for every routing key.
