# SEO SaaS — WordPress Plugin

WordPress plugin for an **AI-powered SEO SaaS platform** (first market: travel and tourism websites,
sold as a USD subscription). The plugin securely pairs a WordPress site with the central SaaS, builds
a content inventory, receives approved article drafts and publishes them, applies internal-link
suggestions with preview and one-click rollback, and records everything in an audit log.

> **Project status: requirements phase.** This repository currently holds the specification, diagrams
> and contributor workflow. Code is built milestone by milestone against GitHub issues
> (see [Roadmap](#roadmap)). Commands in [Scripts](#scripts) describe the target tooling and become
> available as milestone M0 lands.

## What it does — and does not do

| ✅ Does | ❌ Does not |
|---|---|
| Pair a site with the SaaS (signed token, revocable, encrypted at rest) | Talk to Google / Search Console (that lives only in the SaaS) |
| Inventory posts/pages and internal links; find orphan, broken and over-linked pages | Crawl the site over HTTP in v1 |
| Receive AI drafts as **WordPress drafts** and publish only after approval | Publish anything without the configured approval (default: manual for every item) |
| Preview, apply and **roll back** internal link suggestions | Add external links, link exchanges or partnership placements (rejected in v1) |
| Keep an append-only audit log (who, what, when) | Promise ranking improvements — ever |
| Provide a React admin UI (status, queue, links, audit, settings, diagnostics) | Handle payments or licensing UI (SaaS-side; see below) |

## Documentation map

| Document | Purpose |
|---|---|
| [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) | **Source of truth**: scope, roles, every functional/security requirement with ID and acceptance criteria |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Layers, folder structure, data model, lifecycle, conventions |
| [docs/API.md](docs/API.md) | REST endpoints, auth, envelope, error codes, rate limits |
| [docs/SUBSCRIPTION_AND_LICENSING.md](docs/SUBSCRIPTION_AND_LICENSING.md) | Proposed subscription/entitlement design (separate work item) |
| [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md) | Workflow: issues → branches → PRs, standards, checklist |
| [docs/SECURITY.md](docs/SECURITY.md) | Disclosure policy and security model |
| [docs/CHANGE_REQUESTS.md](docs/CHANGE_REQUESTS.md) | Register of requirement changes |
| [CHANGELOG.md](CHANGELOG.md) | Release notes |

## Architecture at a glance

```
 ┌───────────────────────────── WordPress site ─────────────────────────────┐
 │  wp-admin React app  ──REST──►  Api (thin controllers, schema, perms)    │
 │  (assets/admin)                       │                                    │
 │                                       ▼                                    │
 │            Domain: Pairing · Inventory · Publishing · Linking · Audit     │
 │                  (pure PHP, depends only on interfaces)                    │
 │                                       ▲ implemented by                     │
 │  Infrastructure: SaaS client · $wpdb repos · Action Scheduler queue ·      │
 │                  crypto · config · logger         Core: DI · hooks · migrations
 └───────────────▲──────────────────────────────────────────────┬─────────────┘
        signed push (optional)                       poll + report-back (HTTPS)
 ┌───────────────┴──────────────────────────────────────────────▼─────────────┐
 │  SaaS (separate project): plans, AI drafts, link suggestions, Search       │
 │  Console data, billing / licensing                                          │
 └────────────────────────────────────────────────────────────────────────────┘
```

Full diagram set (all in [`docs/diagrams/`](docs/diagrams/)):

| # | Diagram |
|---|---|
| 01 | [Layered architecture and SaaS boundary](docs/diagrams/01-layered-architecture.png) |
| 02 | [End-to-end plugin process flow](docs/diagrams/02-end-to-end-flow.png) |
| 03 | [Site pairing flow](docs/diagrams/03-pairing-flow.png) |
| 04 | [Content inventory scan flow](docs/diagrams/04-inventory-scan-flow.png) |
| 05 | [Draft intake, approval and publishing flow](docs/diagrams/05-draft-approval-publish-flow.png) |
| 06 | [Internal link suggestion, apply and rollback flow](docs/diagrams/06-link-suggestion-apply-rollback-flow.png) |
| 07 | [REST request pipeline](docs/diagrams/07-rest-request-pipeline.png) |
| 08 | [Database tables](docs/diagrams/08-database-tables.png) |
| 09 | [Branching and release flow](docs/diagrams/09-branching-and-release-flow.png) |
| 10 | [Activation, deactivation and uninstall flow](docs/diagrams/10-plugin-lifecycle-flow.png) |

![End-to-end flow](docs/diagrams/02-end-to-end-flow.png)

## Key design rules

1. **Human in control** — approve before change, preview before apply, rollback after apply.
2. **Business logic is separate from UI** — Domain has no WordPress calls; controllers stay thin.
3. **Never break the customer's site** — timeouts, retries with backoff, circuit breaker, errors logged not shown.
4. **Everything external sits behind an interface** — the SaaS client has a mock so the plugin is built before the SaaS exists.
5. **Secure by default** — capability checks on every route, nonces, signed requests, encrypted token, uninstall cleanup, multisite-safe.

## Prerequisites

PHP 8.1+ (with `sodium`), WordPress 6.4+, MySQL 5.7+/MariaDB 10.4+, Composer 2, Node 20+, Docker (for `wp-env`).

## Quick start (target workflow)

```bash
git clone https://github.com/alimarchal/wordpress-plugin.git && cd wordpress-plugin
composer install
npm install
cp .env.example .env     # development only; never commit .env
npm run env:start        # wp-env: local WordPress with the plugin mounted
npm run start            # watch & rebuild the React admin app
```

## Environment configuration

A distributed plugin cannot rely on a `.env` file on customer servers, so configuration is layered.
Highest priority wins: **`wp-config.php` constants → environment variables → `.env` (dev only) →
bundled `config/` defaults.** Only `ConfigRepository` reads these.

| Key | Development | Staging | Production |
|---|---|---|---|
| `SEOSAAS_ENV` | `development` | `staging` | `production` |
| `SEOSAAS_API_URL` | `http://localhost:8000/api` | staging API (HTTPS) | production API (HTTPS) |
| `SEOSAAS_SAAS_PUBLIC_KEY` | dev key | staging key | production key |
| `SEOSAAS_USE_MOCK_SAAS` | `true` | `false` | forced `false` |
| `SEOSAAS_LOG_LEVEL` | `debug` | `info` | `warning` |
| `SEOSAAS_ENCRYPTION_KEY` | optional | recommended | recommended |

Staging / production (`wp-config.php`):

```php
define( 'SEOSAAS_ENV', 'production' );
define( 'SEOSAAS_API_URL', 'https://api.example.com' );
define( 'SEOSAAS_SAAS_PUBLIC_KEY', '…' );
```

Details and the full key list: [docs/REQUIREMENTS.md §10](docs/REQUIREMENTS.md#10-environment-configuration).

## Scripts

| Command | Purpose |
|---|---|
| `composer lint` / `composer lint:fix` | PHPCS with WordPress Coding Standards |
| `composer analyse` | PHPStan level 6+ |
| `composer test`, `test:unit`, `test:integration` | PHPUnit |
| `composer quality` | Dead-code and duplication checks |
| `npm run build` / `npm run start` | `@wordpress/scripts` production build / watch |
| `npm run lint` / `npm test` | ESLint + Prettier / Jest + React Testing Library |
| `npm run env:start` | Local WordPress via `@wordpress/env` |
| `composer release:zip` | Production ZIP via `bin/build-zip.sh` and `.distignore` |

## REST API (summary)

Namespace `/wp-json/seosaas/v1/` · resources: `/pairing`, `/inventory`, `/drafts`, `/links`,
`/links/changes/{id}/rollback`, `/audit`, `/settings`, `/jobs`, `/diagnostics`. Every response uses
`{ success, data, meta, error:{ code, message } }`. Full reference: [docs/API.md](docs/API.md).

## Roadmap

| Milestone | Scope | Est. |
|---|---|---|
| M0 Foundation | Repo tooling, CI, folder structure, container, config, logger, migrator, wp-env, mock SaaS | 1.5 wk |
| M1 Pairing & security base | Pairing, crypto, signatures, permission framework, audit service | 1.5 wk |
| M2 Inventory | Chunked scans, job tracking, delta sync, orphan/broken detection | 2 wk |
| M3 Drafts & publishing | Draft intake, sanitiser, block conversion, SEO adapters, approval API | 2 wk |
| M4 Internal linking | Patcher, preview, apply, guardrails, rollback, bulk apply | 2 wk |
| M5 Admin UI | React screens, stores, services, a11y, i18n | 2–3 wk |
| M6 Hardening & release | Security review, compatibility matrix, docs, release ZIP, pilot on 3–5 sites | 1.5 wk |

Estimates are illustrative (~12–14 weeks for one developer incl. pilot). Milestones are tracked in
four GitHub tracking issues — see the [issue list](https://github.com/alimarchal/wordpress-plugin/issues):

1. Foundation, configuration, pairing and security base (M0–M1)
2. Content inventory, background jobs and audit log (M2)
3. Draft publishing and internal linking (M3–M4)
4. Admin React UI, CI/CD and release hardening (M5–M6)

## Testing

Unit tests for Domain (no WordPress, ≥ 85 % coverage), integration tests for every REST endpoint and
migration (WP test suite), security tests (permission matrix per role, XSS corpus, SSRF, signature
tamper), UI tests (stores, components, axe), E2E smoke on release (Playwright on `wp-env`).

## Release process

Branches `main` (releases) · `staging` (QA) · `develop` (integration). PRs run lint → analyse → test →
build. Merge to `staging` builds a QA ZIP; tag `vX.Y.Z` on `main` builds the production ZIP (no dev
dependencies, prefixed vendor code, checksum) and creates a GitHub Release. See
[diagram 09](docs/diagrams/09-branching-and-release-flow.png).

## Troubleshooting (initial)

| Symptom | Likely cause / action |
|---|---|
| "Needs re-pair" notice | Site URL changed (migration or staging clone) — disconnect and pair again |
| Pairing fails with clock warning | Server time differs from SaaS by > 5 min — fix NTP/host time |
| Scan stays "queued" | WP-Cron disabled or Action Scheduler not running — see Diagnostics and Site Health |
| Link apply returns 409 | Page changed after the suggestion was made — request a refreshed suggestion |
| Banner "SaaS unavailable" | Circuit breaker open after repeated failures — automatic retry; site is unaffected |

## Contributing

Work is driven by GitHub issues. Read [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md), claim a checklist
item on a tracking issue, and open a small PR. To change a requirement, open a *Change request* issue.

## Security

Report vulnerabilities privately — see [docs/SECURITY.md](docs/SECURITY.md).

## Data processing

The plugin shares only site metadata and content inventory with the paired SaaS (no user emails,
passwords or customer data). Consent is captured at pairing; the privacy-policy text is registered
with WordPress.

## Licence

To be decided (D-08). Recommendation: private ZIP distribution first with GPL-compatible code.
