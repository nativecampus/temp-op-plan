# Step 2: nmm-base

Depends on step 1.

Read base_app's `docs/` directory in full (architecture.md, coding_standards.md, testing-guide.md, openapi-export.md, frontends.md), and `native-ui`'s README and `dist/index.d.ts`. Then `nmm_build_plan_v14.md` work order step 2.

## Scope

Fork `base_app` to a new repo `nmm-base`. Add NMM-specific scaffold-time content. `nmm-base` is a scaffold template, not a runtime dependency: repos are scaffolded once via `install.sh nmm_su_mm` and `install.sh nmm_advertiser` and then evolve independently.

`pyproject.toml` does not list `nmm-events`, `nmm-event-contracts`, `nmm-tenancy`, or `nmm-auth`. Those packages are step 3 and added by each application repo after scaffolding.

## Vite SPA scaffold pre-wiring

In `frontends/_template/`:

- Add `native-ui` as a dependency in `package.json` at version `2.1.0` plus its peer dependencies (`@mui/lab`, `@mui/material`, `@mui/system`, `@mui/x-date-pickers`, `react@>=19.0.0`, `react-dom@>=19.0.0`, `@emotion/react`, `@emotion/styled`)
- `src/App.tsx` wraps the app in `NativeProvider` with the default theme
- `src/native-ui-theme-example.tsx` showing `createCustomTheme` usage for white-label SPAs (the Publisher Widget will use this; non-widget SPAs use the default theme)
- `docs/native-ui.md` listing the available components (selective MUI re-exports, `DataCard`, `Icon`, `StatusBadge`, `NativeProvider`, `createCustomTheme`) and pointing at native-ui's README

## NMM-specific docs

- `docs/CLAUDE.md`: NMM agent conventions. The harness pattern (`python manage.py harness`), architectural patterns, cross-API event flow expectations, the OpenAPI-as-export rule, the multi-tenancy listener pattern (forward reference to `nmm-tenancy`), the JWT validation pattern (forward reference to `nmm-auth`). Reference `nmm_functional_spec_v16.md` as the canonical system spec and `nmm_build_plan_v14.md` as the construction plan.
- `docs/architecture-patterns.md`: outbox transactionality, projection tables, event consumers, multi-tenancy listener, JWT validation, OpenAPI as exported artefact.
- `docs/harness-conventions.md`: the `python manage.py harness` pattern as it exists in nmm today, carried forward.
- `docs/cross-api-events.md`: publish/consume pattern, routing key convention `<api>.<aggregate_type>.<event_type>`, schema versioning rules per v16 section 8, runtime dependency on `nmm-events`, schema package dependency on `nmm-event-contracts`.

## install.sh

The script that scaffolds a new repo. Same shape as base_app's `install.sh`. Takes one snake_case argument, clones `nmm-base`, removes the `.git` link, runs `python -m scripts.init_project <name>`. The `init_project.py` (inherited from base_app) handles renames; nmm-base's version adds renames for the NMM-specific tokens in the new docs and templates.

## Acceptance

`install.sh test_app` produces a runnable repo with:

- The directory layout
- The Vite SPA scaffold under `frontends/_template/` installable with `npm install`
- The OpenAPI export script working against a hello-world FastAPI app
- The NMM docs in place with token renames applied
- A clean git history ready for first commit
