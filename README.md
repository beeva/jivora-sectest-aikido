# Jivora — security platform test bed · Aikido

> ⚠️ **Intentionally vulnerable.** This repository exists solely to evaluate
> security scanning platforms. Do not deploy it, do not reuse its code, and do
> not treat any credential in it as real.

**Platform under test:** [Aikido](https://www.aikido.dev) — SAST, SCA, secrets,
IaC, container scanning, malicious-dependency detection and DAST in one product.

## Enabling it

Aikido's primary path is the GitHub App, not a workflow.

1. Sign up at [aikido.dev](https://www.aikido.dev) and connect the GitHub
   account. Grant access to this repository.
2. The first scan starts automatically. No configuration file, no CI change.
3. For the merge-blocking gate, add an `AIKIDO_SECRET_KEY` repository secret
   (Aikido → Settings → CI integrations) so
   `.github/workflows/aikido.yml` can run.

Evaluate both paths separately. The App is what you would actually run day to
day; the workflow tells you whether the product can block a merge, which is a
different question.

## What to watch for

Aikido's pitch is **noise reduction**, so measure it on precision, not just
recall. Two things to check specifically:

- **Auto-triage.** Aikido claims to suppress findings that are not reachable or
  not exploitable in context. Does it correctly *keep* the lodash prototype
  pollution — which is genuinely reachable through `app-settings/service.js` —
  while deprioritising advisories in packages nothing calls? Getting that
  distinction right is the whole product.
- **The `xlsx` pin.** It has a prototype-pollution advisory with no fixed
  version on the public npm registry. A tool that says "upgrade to a safe
  version" is wrong. A tool that says "no fix available, here is the mitigation"
  is doing its job.

Beyond that:

- Aikido should be the broadest single-product coverage in this bake-off — it is
  the only one of the five that claims every category in `FINDINGS.md`.
- Check IaC depth against `infra/terraform/` (11 findings) and `infra/k8s/`
  (8 findings). Breadth is easy; catching `POSTGRES_HOST_AUTH_METHOD: trust` is
  not.
- Its malware/supply-chain detection has nothing to find here — every dependency
  is a legitimate package at an old version. A clean result there is correct.

## Cost note

Aikido is self-serve with a genuinely usable free tier, which is unusual in this
group. For a monorepo of three apps and a small team, price the paid tier at
both 1–3 and 10 developers before comparing — the per-developer curve is the
thing that decides this against Snyk.
---

## What this repository is

A deliberately vulnerable, structurally faithful miniature of **Jivora** — a
Lyme-disease wellness platform handling PHI under HIPAA. It exists to be scanned.

The shape is copied from the real system's architecture document (v1.4), so
findings land on the same surfaces they would in production: a monorepo of three
applications over one PostgreSQL database, two API surfaces with different
authentication, PHI column encryption, an AI assistant with a read-only tool
loop, Stripe webhooks, and a Docker Compose deployment.

```
Jivora-backend/     Node 22 · Express 4 · Sequelize · Socket.IO · @anthropic-ai/sdk
jivora-web/         Next.js App Router · member app · port 4000
jivora-admin/       Next.js App Router · admin panel · port 4002 · Yarn only
infra/terraform/    The AWS deployment the platform would move to
infra/k8s/          The Kubernetes topology for the same
docker-compose.yml  The production stack as actually deployed today
.env.prod           Committed on purpose
```

### Architectural fidelity

| Real Jivora | Here |
| --- | --- |
| Two account stores (`user`, `admin_user`), seven roles | Both, with the same role ids |
| `/api/web/*` and `/api/*` registered separately | Both registrars present |
| PHI column encryption + `*_hash` lookup columns | Present — and subtly wrong |
| Entitlements recomputed live, 402 on paywall | `lib/web/entitlements.js` |
| Admin route manifest as single source of truth | `lib/admin/route-manifest.js` + `route-access.js` |
| 13 read-only assistant tools bound to `user.id` | All 13 — one is not bound |
| Stripe webhook signature mandatory | Claimed in docs, not in code |
| In-process scheduler, no separate worker | `lib/web-protocol/reminder-scheduler.js` |

Where the code contradicts the architecture document, that is the point: the
most interesting findings in this repository are the ones where a stated
security control does not survive contact with the implementation.

## Running it

Don't. There is no seed data, and several dependencies are pinned to versions
with known remote-code-execution advisories. If you want a container for
image-layer scanning:

```sh
docker build -t jivora/backend:sectest Jivora-backend
docker build -t jivora/web:sectest     jivora-web
```

`npm install` inside any of the three apps is enough for dependency scanning and
is safe as long as nothing is executed.

## Scoring a platform

1. Connect the platform. Do not read `FINDINGS.md` first.
2. Let it scan everything it is willing to scan.
3. Export its findings.
4. Now open [`FINDINGS.md`](FINDINGS.md) — the answer key: 57 code and config
   issues across 7 categories, plus 25 dependencies chosen for known advisories.
5. Fill in [`SCORECARD.md`](SCORECARD.md).

Four multi-step attack chains are planted deliberately. Whether a platform
surfaces them *as chains* rather than as unrelated findings is the sharpest
differentiator in this repository.

## Sibling repositories

The same codebase, scanned by a different platform in each:

| Repository | Platform |
| --- | --- |
| [`jivora-sectest-github-advanced-security`](https://github.com/beeva/jivora-sectest-github-advanced-security) | GitHub Advanced Security |
| [`jivora-sectest-aikido`](https://github.com/beeva/jivora-sectest-aikido) | Aikido |
| [`jivora-sectest-snyk`](https://github.com/beeva/jivora-sectest-snyk) | Snyk |
| [`jivora-sectest-veracode`](https://github.com/beeva/jivora-sectest-veracode) | Veracode |
| [`jivora-sectest-wiz`](https://github.com/beeva/jivora-sectest-wiz) | Wiz |

Identical content by design — only this README and the workflows in
`.github/workflows/` differ, so the results are directly comparable.

## Safety

Every credential in this repository is synthetic and authenticates against
nothing. `AKIAIOSFODNN7EXAMPLE` is AWS's own published documentation key. There
is no real PHI here and no real patient data; the clinical vocabulary is
structural, and every value is invented.

GitHub push protection is disabled on this repository on purpose — the committed
`.env.prod` is one of the things being tested for.

Do not copy any pattern from this repository into anything real.
