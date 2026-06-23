---
name: shopify-executor
description: Implement and verify Shopify themes, apps, extensions, and versioned Admin API changesets using development environments and human-gated releases.
version: 0.1.0
requires_tools:
  - enable_sandbox_tools
platforms:
  - linux
tags:
  - shopify
  - theme
  - app
  - extension
category: commerce-development
---

# Shopify executor

Use this skill for Shopify code or configuration changes.

## Classify before acting

- `theme`: Liquid, JSON templates, sections, blocks, CSS, JavaScript.
- `app`: hosted backend/frontend and Shopify app configuration.
- `extension`: Admin, checkout, customer account, POS, or theme app extension.
- `admin_data`: GraphQL mutation represented as a reviewable changeset.
- `headless`: Hydrogen/custom frontend and Storefront API.

Do not proceed without an explicit development store and target environment.
Never infer that a store or theme is safe for production writes.

## Git workflow

```bash
git clone --branch "<base-branch>" "<repo-url>" repo
cd repo
git checkout -b "feature/<branch-slug>"
```

Git is the source of truth. Detect Shopify/Git drift before overwriting remote
theme state.

## Theme verification

```bash
shopify theme check
npm run lint --if-present
npm test --if-present
```

Use Playwright only when requested. Preview with a development/unpublished
theme. Never publish or push live without human approval.

## App and extension verification

```bash
npm ci
npm run lint --if-present
npm run typecheck --if-present
npm test --if-present
shopify app build
```

`shopify app deploy` deploys app configuration/extensions, not the hosted web
application. Release and production hosting remain Saga/CI responsibilities.

## Admin API changes

Do not execute production mutations in the coding loop. Create:

```text
shopify-changes/<job-id>/
  plan.json
  before.json
  apply.graphql
  rollback.graphql
  verification.json
```

The executor applies an approved changeset later with audit logging.

## High-risk changes

Flag as high risk:

- checkout/payment/shipping/discount/validation;
- billing, authentication, API scopes, webhook verification;
- customer PII;
- production mutations;
- theme publish or app release;
- changes without a tested rollback.

## Secrets

Never commit app secrets, access tokens, Shopify CLI login state, production
database credentials, or `.env` values. Inject them through the job/session.
