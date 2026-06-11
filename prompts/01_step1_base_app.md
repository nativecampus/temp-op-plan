# Step 1: base_app

Read base_app's `CLAUDE.md`, `docs/`, `scripts/init_project.py`, `install.sh`, `manage.py`, and `package.json`. Then read `nmm_build_plan_v14.md` work order step 1.

## Scope

Add an OpenAPI export script with a CI gate. Add a Vite SPA scaffold pattern under `frontends/`. Both are general FastAPI conventions and live in `base_app` so future projects scaffolded from it inherit them.

## OpenAPI export script

`scripts/export_openapi.py` runs the FastAPI app in test mode, calls `app.openapi()`, writes the result to `<api-name>.openapi.json` at the repo root.

The api-name and the FastAPI app import path are configurable via env vars (`OPENAPI_APP_IMPORT`, `OPENAPI_OUTPUT_FILENAME`) so each repo derived from base_app names its file correctly without editing the script. `init_project.py` writes sensible defaults for these env vars into `.env.example`.

CI step in the existing workflow: runs the script and fails the PR if the committed file diverges from the generated output.

`docs/openapi-export.md`: the convention, the script, how to regenerate locally, the CI gate.

## Vite SPA scaffold

`frontends/` directory with a README describing the pattern. SPAs go under `frontends/<spa-name>/`. The repo-root OpenAPI document is the contract. The SPA generates its TypeScript client from that document in CI.

Template at `frontends/_template/`:

- `package.json` with vite, react, typescript, openapi-typescript as dev dependencies
- `vite.config.ts`, `tsconfig.json`, `index.html`, `src/main.tsx`, `src/App.tsx`
- openapi-typescript config reading `<api-name>.openapi.json` at the repo root, writing types to `src/api/types.ts`
- npm scripts: `dev`, `build`, `generate-types`
- Prism mock-server config for development against the OpenAPI document before the API exists

`docs/frontends.md`: the convention, the template, the scripts, the prism pattern.

## Init script

`scripts/init_project.py` extended to rename the OpenAPI output filename token in `.env.example` and to update the frontends template's `package.json` name field with the new project's kebab-case name.

## Out of scope

Anything NMM-specific. `native-ui` pre-wiring. NMM docs. Those belong in `nmm-base` (step 2).

## Acceptance

The OpenAPI script runs against base_app's own FastAPI app and produces a baseline file. The CI gate is wired and passes against the baseline. `cd frontends/_template && npm install && npm run dev` works. `python -m scripts.init_project test_project` produces a runnable repo whose OpenAPI export and SPA scaffold both work end to end.
