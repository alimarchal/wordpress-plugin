# Subscription and Licensing — Suggested Approach

> **Status: proposal, not yet in scope.** The SaaS API calls and licensing are being handled as a
> separate piece of work. This document records the recommended design so the plugin is built
> *ready* for it (via an interface and a mock) without implementing billing in this repository.
> Pricing below is illustrative and taken from the Product Blueprint.

## 1. Business model recap

Subscription (USD) sold by the SaaS. First market: travel and tourism websites.

| Plan | Articles / month | Tracked keywords | Indicative price |
|---|---|---|---|
| Starter | 5 | 10 | $49 / month |
| Growth | 10 | 25 | $99 / month |
| Pro | 15 | 50 | $149 / month |

All plans include Search Console integration, internal linking and monthly reports. Final packaging,
article-credit definition, rollover and revision rules are open product decisions (see Blueprint §12).

## 2. Principle: the plugin enforces *entitlements*, the SaaS owns *billing*

- The plugin never handles payment details, invoices or Google data.
- The **pairing token is the licence**: one paired site = one SaaS site record. There is no separate licence-key text box in v1, which removes key sharing and key-leak risk.
- Quotas (articles/month, keywords, site count) are counted **by the SaaS**. The plugin only reads the result.

## 3. Proposed entitlement contract

A single read-only call from plugin to SaaS (cached), exposed to Domain through an interface:

```php
interface EntitlementProviderInterface
{
    public function current(): Entitlement; // plan, status, limits, validUntil, features[]
}
```

Suggested `GET /sites/{id}/entitlement` response (signed by the SaaS, verified with the pinned public key — same mechanism as pairing):

```json
{
  "plan": "growth",
  "status": "active",
  "articles_remaining": 7,
  "period_ends_at": "2026-11-30T00:00:00Z",
  "features": ["drafts", "link_suggestions", "inventory_sync"],
  "valid_until": "2026-10-20T12:00:00Z",
  "signature": "…"
}
```

- Delivered also on every normal poll (`GET /sites/{id}/tasks`) so no extra request is needed.
- Cached with `valid_until` (suggested 72 h). A SaaS outage must **not** lock the customer out — the cache is honoured until it expires.
- `MockEntitlementProvider` for dev/CI; the mock obeys the same rule as the SaaS mock: forbidden when `SEOSAAS_ENV=production`.

## 4. Status behaviour (suggested)

| SaaS status | Plugin behaviour |
|---|---|
| `active` | Everything enabled |
| `past_due` (grace, suggested 7 days) | Everything works; admin notice asks to update billing |
| `quota_exhausted` | No new drafts delivered by SaaS; existing drafts, approvals and links continue |
| `canceled` / `suspended` | Stop pulling new drafts/suggestions. **Never** delete or unpublish content. Rollback, audit log, export and disconnect stay available |
| `unpaired` / `site_limit_exceeded` | Show *Needs re-pair* / upgrade notice; no outbound sync |

Design rule: a billing problem must never trap customer content or remove the customer's ability to
undo changes (rollback) or leave (disconnect/uninstall).

## 5. Where it plugs into this codebase

| Concern | Placement |
|---|---|
| `EntitlementProviderInterface`, `Entitlement` value object | `src/Domain/Contracts`, `src/Domain/Pairing` |
| HTTP + mock implementations | `src/Infrastructure/Saas` |
| Gate checks (`canReceiveDrafts()`) | Domain services, consulted before accepting inbound drafts/suggestions |
| UI | Dashboard "plan / quota" card (read from entitlement); upgrade link opens the SaaS billing page — **no payment form inside WordPress** |
| Audit | `SECURITY_EVENT` on signature failure; `SETTINGS_CHANGED` is not used for plan changes (they happen at the SaaS) |
| Config | `SEOSAAS_FEATURE_ENTITLEMENTS` flag so the mock can be used until the SaaS is ready |

## 6. Recommendations for the SaaS-side (outside this repo)

1. **Payments:** use a Merchant-of-Record provider (e.g. Paddle or Lemon Squeezy) or Stripe Billing with Stripe Tax; MoR removes most international sales-tax work for a small team. Decide based on which countries you must collect from (open decision in Blueprint §12).
2. **Webhook-driven state:** the billing provider webhooks update the subscription row; the plugin only ever sees the resulting entitlement.
3. **Site seats:** limit paired sites per plan on the SaaS (pairing code issuance), not in the plugin.
4. **Article credit definition:** decide what counts as one credit and what counts as a revision *before* the SaaS counts usage, then expose `articles_remaining` accordingly.
5. **Trial:** time-boxed trial plan = just another `plan` value; no special plugin code.

## 7. Open questions to settle before building

- Do unused article credits roll over?
- Are revisions free up to N per article?
- Annual billing and discounts?
- Which payment gateways/countries at launch?
- Grace period length and what "suspended" shows to the site owner?
- Is the agency/white-label tier a separate SKU?

Track outcomes as Change Requests in [CHANGE_REQUESTS.md](CHANGE_REQUESTS.md).
