# Contributing

Thanks for helping build the plugin. Work is driven by **GitHub issues**: every change starts as an
issue and ends as a pull request that links it.

## Where to start

1. Read the [README](../README.md) and [REQUIREMENTS.md](REQUIREMENTS.md).
2. Pick one of the milestone issues (M0–M6 are grouped into four tracking issues). Each lists requirement IDs (`FR-…`, `SEC-…`) and a checklist.
3. Comment on the issue to claim a checklist item, then open a PR for that item (small PRs, one concern each).

## Changing a requirement

Requirements are living. To add, change or remove one:

1. Open an issue with the **Change request** template.
2. Quote the affected ID(s), explain the reason and the impact on scope/time/security/data model.
3. Once agreed, add a row to [CHANGE_REQUESTS.md](CHANGE_REQUESTS.md) and update `REQUIREMENTS.md` in the same PR as the code.

Ask before assumptions that affect **security** or the **data model** — open an issue or comment first.

## Branches

| Branch | Purpose |
|---|---|
| `main` | Production releases only |
| `staging` | Pre-release QA |
| `develop` | Integration |
| `feature/*`, `fix/*`, `chore/*`, `release/*`, `hotfix/*` | Short-lived work, kebab-case (e.g. `feature/link-rollback`) |

PRs target `develop`. Branch protection: PR required, ≥ 1 review, status checks green (lint, analyse, test, build), no force-push.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/): `feat(linking): add rollback conflict detection`,
`fix(pairing): reject expired code`, `docs:`, `chore:`, `test:`, `refactor:`.

## Code standards (summary)

- PHP 8.1+, PSR-4, PSR-12, WPCS; prefix `seosaas_` for hooks/options/tables/capabilities.
- **Domain code never calls WordPress functions**; controllers are thin; repositories own SQL; everything external is behind an interface.
- Exceptions from the hierarchy only; catch at boundaries; no empty `catch`; no `error_log()` — use the Logger.
- All input validated then sanitised; all output escaped; all SQL through `$wpdb->prepare()`.
- No hardcoded URLs, keys or secrets — only `ConfigRepository`.
- No duplicate or dead code; extract on the third repetition; no commented-out code.
- Comments explain **why**, not how. PHPDoc on public classes/methods.
- React: `@wordpress/data` stores, components presentational, API calls only via `services/`, loading/empty/error states everywhere, camelCase JS, PascalCase components, kebab-case files.

## Checks to run before pushing

```bash
composer lint        # PHPCS (WPCS + PHPCompatibility)
composer analyse     # PHPStan level 6+
composer test        # PHPUnit (unit + integration)
npm run lint         # ESLint + Prettier
npm test             # Jest + React Testing Library
npm run build        # @wordpress/scripts
```

## Pull request checklist

- [ ] Linked requirement IDs (FR-/SEC-/NFR-) and tests added or updated
- [ ] Lint, static analysis and tests pass locally and in CI
- [ ] No secrets, URLs or environment values in code
- [ ] No dead or duplicated code introduced; helpers reused
- [ ] Errors handled via exception hierarchy; user messages safe and translatable
- [ ] New routes have schema, permission callback, rate-limit bucket and API docs
- [ ] Audit events added for new state-changing actions
- [ ] Comments explain why; README/docs updated if behaviour or setup changed
- [ ] Backwards compatibility and migration impact considered

## How to add a module or adapter

- **Module:** implement `ModuleInterface::register()`, add Domain service + ports, Infrastructure adapters, Api controller, and register it in `Core/Plugin.php`. Do not edit other modules.
- **SEO-plugin adapter:** add a class under `Infrastructure/WordPress/SeoPluginAdapters/` implementing the adapter interface; no Domain changes.

## Tests

Unit tests for Domain (no WordPress loaded, ≥ 85 % coverage target), integration tests for every REST endpoint and migration, security tests (permission matrix, XSS corpus, SSRF, signature tamper), UI tests for stores and components.
