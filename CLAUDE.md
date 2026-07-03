# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

ERP-Government (eGov) — an Odoo 15.0 (OCB) ERP distribution for Thai government agencies, built by
NSTDA/Ecosoft. It is a **Doodba** (Docker Odoo Base) child project: core Odoo/OCB and third-party OCA
addons are aggregated at build/dev time from `odoo/custom/src/repos.yaml` (they are NOT committed to
this repo — only `odoo/custom/src/private/*` is version-controlled, see `.gitignore`). There is no
local (non-Docker) dev workflow.

## Commands

Per README.md, run/build directly with `docker-compose`:

```bash
git clone https://github.com/NSTDA/ERP-Government.git
cd ERP-Government
docker-compose -f egov-prod.yaml up -d          # pull & run the published ghcr.io/nstda/egov_odoo image
docker-compose -f egov-prod-build.yaml up -d    # build the image locally (from ./odoo + repos.yaml), then run
```

`egov-prod.yaml` does not build anything locally — it always runs the pre-built published image, so
local changes (private addon code, `repos.yaml` pins) have no effect unless you use
`egov-prod-build.yaml` instead.

Once running, the app is available at http://localhost/ or http://127.0.0.1/.

`.docker/*.env` (referenced by `egov-prod.yaml`/`egov-prod-build.yaml` via `env_file:`) ship with
placeholder values checked into git — replace them with real credentials before any real deployment.

This project also has a Doodba `tasks.py`/`invoke` tooling layer (module install/test, OCA addon
aggregation, etc.), but it is not this team's primary workflow — `docker-compose` per README.md above
is the source of truth for running/building.

## Architecture

**Layered addon model**, all resolved through Doodba's `repos.yaml`/`addons.yaml`:

1. **Core** — `ocb` (Odoo Community Backport) 15.0, with one cherry-picked upstream Odoo PR (see the
   `./odoo:` entry's `merges:` in `odoo/custom/src/repos.yaml`).
2. **Extra (OCA)** — ~30 official OCA repositories (`account-budgeting`, `agreement`, `contract`,
   `hr-expense`, `l10n-thailand`, `operating-unit`, `purchase-workflow`, `stock-logistics-*`,
   `mis-builder`, `queue`, etc.), each pinned to specific commits/PRs in `repos.yaml` with inline
   comments explaining the fix/feature. `odoo/custom/src/addons.yaml` lists exactly which addon
   directories from each repo are actually loaded. Modify pins there — never edit vendored code
   directly, since it's re-cloned by `invoke git-aggregate` and is git-ignored locally.
3. **Private (`odoo/custom/src/private/*`)** — the only addons owned by this repo. This is almost
   always where changes belong:
   - `egov_install` — thin addon whose sole purpose is its long `depends` list; installing it pulls in
     the full stack of OCA modules that make up the eGov baseline (asset/operating-unit structure,
     budgeting, procurement, tier validation, Thai localization, etc.). Start here to understand what
     "the eGov system" is composed of.
   - `egov_manual_install` — `auto_install` addon adding a settings toggle; separate from `egov_install`.
   - `egov_config` — master-data/config loader (CSV/XML data files: chart of accounts hookup, operating
     units, users, automations). Depends on `egov_install` + `egov_coa`.
   - `egov_coa` — Thai government Chart of Accounts (via CSV templates + `post_init_hook`).
   - `egov_purchase`, `egov_purchase_request`, `egov_sale`, `egov_hr_expense` — thin
     integration/customization layers wiring OCA `tier_validation` (approval workflow) modules and
     Thai localization (`l10n_th_gov_*`) modules into server actions/views for each business flow.
   - `egov_tier_validation`, `egov_user_role` — cross-cutting approval-workflow and access-control glue
     across operating units (multi-org/multi-department scoping is a recurring theme — see the
     `*_operating_unit*` OCA addons this repo depends on throughout).
   - `egov_budget_*_excel_import_export` — Excel import/export for budget allocation/control, on top of
     OCA's `excel_import_export` + `budget_allocation`/`budget_control_*`.
   - `egov_z_migration` — one-off data migration views/logic (name-prefixed `z` to sort last / run last
     in dependency graphs).
   - `om_credit_limit`, `locust_test` — third-party/ops utility addons (credit-limit warnings; load
     testing harness), not eGov-specific business logic.

   **When adding a new addon under `odoo/custom/src/private/`, add a one-line bullet here** describing
   its purpose and which OCA/core modules it wires together — keep this list in sync with the folder.

**Key implication for changes:** business logic in this codebase is mostly thin — most heavy lifting
(budget control, tier/approval validation, operating-unit-based multi-org scoping, Thai localization)
lives in pinned OCA addons declared in `repos.yaml`, not in `private/`. When a bug or feature looks
like it should live in an OCA module, check whether it's more appropriate to bump/re-pin that module's
commit in `repos.yaml` (with a comment explaining why, matching existing style) versus patching it via
a private addon that inherits/extends it. After changing a pin, rebuild to pick it up:
`docker-compose -f egov-prod-build.yaml build` (the image's default `AGGREGATE=true` build arg means
the Doodba onbuild base re-runs git-aggregate against `repos.yaml` during the image build).

## Conventions

- `repos.yaml` merge entries are annotated with a trailing `# [15.0][TYPE] description` comment citing
  the OCA PR/change being pulled in — keep this convention when adding/updating pins.
- Linting/formatting is enforced via `pre-commit` (see `.pre-commit-config.yaml`), which includes
  `pylint-odoo`.
