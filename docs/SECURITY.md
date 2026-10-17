# Security Policy

## Reporting a vulnerability

Please **do not open a public issue** for security problems. Report privately through GitHub
*Security → Report a vulnerability* on this repository, or email the maintainers (address to be added
before the first release). We aim to acknowledge within 3 business days and to give a remediation
timeline within 10 business days.

## Security model (summary)

Full requirements: [REQUIREMENTS.md §8](REQUIREMENTS.md#8-security-requirements).

- **Pairing:** short-lived single-use code → signed site token; SaaS response verified against a pinned public key; token encrypted at rest (libsodium secretbox); revocable.
- **Inbound (SaaS → plugin):** Ed25519-signed, timestamped (±5 min), replay-protected requests; payload schema validation; HTML allow-list; drafts are always created as `draft`.
- **Outbound (plugin → SaaS):** HTTPS only, certificate verification on, bearer token + HMAC, host allow-list, SSRF guard on image downloads.
- **Admin:** capability check on every route/action, nonce on every state change, no public routes.
- **Data minimisation:** no user emails, passwords or customer data are sent to the SaaS.
- **Cloned staging protection:** pairing is bound to the site URL fingerprint.

## Key rotation (to be completed in milestone M1)

1. SaaS publishes the new public key; plugin accepts old + new for an overlap window via `SEOSAAS_SAAS_PUBLIC_KEY`.
2. `SEOSAAS_ENCRYPTION_KEY` rotation re-encrypts the stored token on next admin load.
3. Compromised site token: revoke at the SaaS, then re-pair.

## Dependency hygiene

Dependabot, `composer audit`, `npm audit`, GitHub secret scanning with push protection, and CodeQL for JS are enabled; a nightly CI job repeats the audits.
