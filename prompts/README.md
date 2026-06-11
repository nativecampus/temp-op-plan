# NMM build plan prompts

Prompts for Claude Code agents executing `nmm_build_plan_v14.md`. Run in numeric order.

Inputs every prompt assumes are present:

- `nmm_functional_spec_v16.md` (system spec)
- `nmm_build_plan_v14.md` (construction plan)

Step 6 (scaffold the new application repos) is two `install.sh` shell commands you run yourself, not an agent task. There is no prompt for step 6.

## Sequencing

```
01 base_app
   └─ 02 nmm-base ───────────────────────────────────────────┐
                                                              │
03 nmm-events runtime ─┐                                      │
                       ├─ 07 event schemas ──┐                │
04 nmm-event-contracts ┘                     │                │
                                              ├─ 09 retrofit nmm
05 nmm-tenancy package ─── 08 tenancy contract                │
                                                              │
06 nmm-auth                                                   │
                                                              │
                                          (step 6: install.sh)┘
                                                ├─ 10 SU MM API stubs
                                                └─ 11 Advertiser API stubs
                                                     └─ (step 8: parallel app work)
```

03, 04, 05, 06 run in parallel. 07 depends on 04. 08 depends on 05. 09 depends on 03, 04, 05, 06, and 07 (so the SchemaRegistry has real schemas to validate against in strict mode and the contextvar shim is in place). 10 and 11 depend on the two `install.sh` runs and on 03/04/05/06/07/08 being released.

## Prompt to v14 work order step mapping

| Prompt | v14 step | What it does |
|---|---|---|
| `01_step1_base_app.md` | 1 | base_app: OpenAPI export script + Vite SPA scaffold |
| `02_step2_nmm_base.md` | 2 | Fork base_app to nmm-base with NMM-specific scaffold |
| `03_step3a_nmm_events.md` | 3 | nmm-events runtime package (outbox, relay, consumer, dispatch). Outbox shape diffs against existing nmm. SchemaRegistry strict only. |
| `04_step3b_nmm_event_contracts.md` | 3 | nmm-event-contracts package infrastructure (schema directory, registry, Python + TypeScript packaging) |
| `05_step3c_nmm_tenancy.md` | 3 | nmm-tenancy listener package |
| `06_step3d_nmm_auth.md` | 3 | nmm-auth JWT validation. Does NOT own permission evaluation |
| `07_step4a_event_schemas.md` | 4 | JSON Schemas for every routing key in nmm-event-contracts |
| `08_step4b_multi_tenancy_contract.md` | 4 | multi-tenancy-contract.md content in nmm-tenancy |
| `09_step5_retrofit_nmm.md` | 5 | Retrofit nmm to consume the four shared packages |
| (no prompt) | 6 | `install.sh nmm_su_mm` and `install.sh nmm_advertiser` (you run these) |
| `10_step7a_su_mm_api_stubs.md` | 7 | FastAPI route stubs + Pydantic schemas in nmm-su-mm; OpenAPI export |
| `11_step7b_advertiser_api_stubs.md` | 7 | Same for nmm-advertiser, including Publisher Widget endpoints |
| (not yet written) | 8 | Parallel application work in three repos: implementations behind every stub, listener wiring, consumer wiring, projections, computed views, SPAs |

## Step 8 is not in this folder

Step 8 is the bulk of the work and needs decomposition into many focused prompts (per API per concern). Not yet written.
