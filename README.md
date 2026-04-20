# KACE-PromptKart

**Public Shopify Hydrogen storefront for selling AI prompt packs.**

Customer-facing storefront deployed on top of `<PROMPTKART_STORE>.myshopify.com`. Every business decision (price visibility, discounts, reward balance, cross-sells) is delegated to the `KACECommerceEngine` middleware — the storefront is a thin rendering layer driven by rule-engine actions.

---

## Placeholders used in this README

| Placeholder | Description | Example |
| --- | --- | --- |
| `<GITHUB_HANDLE>` | GitHub account the repos live under | `your-github-handle` |
| `<PROMPTKART_STORE>` | PromptKart dev store handle | `promptkart-dev` |
| `<KACE_ENGINE_URL>` | URL of KACECommerceEngine in the current env | `http://localhost:8080` in dev |
| `<CUSTOMER_ACCOUNT_API_CLIENT_ID>` | Client ID for Shopify Customer Account API on this store | (from Dev Dashboard) |
| `<CUSTOMER_ACCOUNT_API_ISSUER_URL>` | OIDC issuer URL for this store's customer auth | `https://shopify.com/<store-id>` |

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. Treat it as a standalone acronym (like IKEA or NASA) — don't re-expand it in every sentence. Historically the letters came from **K**ishore **A**pps **C**ommerce **E**ngine; today it's the suite's brand name.

**The 4 repos:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/<GITHUB_HANDLE>/KACECommerceEngine) | TS + Fastify middleware. Rule facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [**KACE-PromptKart**](https://github.com/<GITHUB_HANDLE>/KACE-PromptKart) *(this repo)* | Hydrogen public storefront — AI prompt packs. |
| [KACE-StudyDesk](https://github.com/<GITHUB_HANDLE>/KACE-StudyDesk) | Hydrogen public storefront — micro-courses. |
| [KACE-Extended-Rules-Sidecar](https://github.com/<GITHUB_HANDLE>/KACE-Extended-Rules-Sidecar) | Java + Spring Boot + Drools JVM sidecar. |

---

## Architecture

### Where this storefront sits

```
Customer browser
        │
        ▼
┌──────────────────────────────────────────────┐
│  KACE-PromptKart  (Hydrogen + Remix, :3000)  │
│                                              │
│  • Shopify Storefront API                    │
│    (public token — catalog, cart, checkout)  │
│  • Shopify Customer Account API              │
│    (OAuth/OIDC — customer login)             │
│                                              │
│  On every page load / cart update:           │
│  ──────────────────────────────────────────  │
│  POST <KACE_ENGINE_URL>/api/v1/rules/evaluate│
│       Authorization: Bearer <customer JWT>   │
│  POST <KACE_ENGINE_URL>/api/v1/rewards/apply │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────┐
     │  KACECommerceEngine      │
     │  (Fastify TS middleware) │
     └──────────────────────────┘
                 │
                 ▼
         Shopify Admin GraphQL
     (never called by this storefront directly)
```

Core properties:

- **Thin rendering layer.** No business logic in components. The storefront renders a `RenderModel` produced by the loader, which was populated from KACE's rule-engine actions.
- **Never calls Admin GraphQL directly.** All Shopify Admin-tier operations go through `KACECommerceEngine`.
- **Customer-facing only.** No merchant admin UI (merchants use Shopify's built-in admin).

### Core UX pattern — guest-vs-logged-in gating

- **Guests** see product title, images, and a teaser description. They do **not** see price, variant options, or a buy button.
- **Logged-in customers** (via Shopify Customer Account API) see the full price — including any rule-driven discount — plus the buy button and reward balance.

The gating is **not hardcoded** in React. It's expressed as a `kace_reward_rule` MetaObject authored in the Shopify admin and evaluated by `KACECommerceEngine`. (For how KACE's own `SessionStorage` gets invoked when this storefront authenticates a customer, see **[README_How_SessionStorage_Invoked.md](https://github.com/<GITHUB_HANDLE>/KACECommerceEngine/blob/main/README_How_SessionStorage_Invoked.md)** in KACECommerceEngine.)

```jsonc
// trigger: product.view
{
  "conditions": { "all": [
    { "fact": "customer.isAuthenticated", "operator": "==", "value": false }
  ]},
  "actions": [
    { "type": "HIDE_FIELD", "params": { "field": "price" } },
    { "type": "HIDE_FIELD", "params": { "field": "buyButton" } },
    { "type": "SHOW_CTA",   "params": { "text": "Login to see price & buy", "link": "/account/login" } }
  ]
}
```

The storefront's loader receives the action list and an `ActionApplierRegistry` turns each into a mutation on the `RenderModel`. `<ProductDetail>` then reads the model and renders accordingly.

### Customer Account API login flow

```
Guest clicks "Sign in"
      ↓
Hydrogen → Shopify /authentication/<store-id>/oauth/authorize?...
      ↓
Customer logs in (Shopify-hosted UI)
      ↓
Redirect back to /account/callback?code=...&state=...
      ↓
Hydrogen exchanges code for access + ID tokens
      ↓
Tokens stored in Hydrogen session (HttpOnly cookie)
      ↓
Every subsequent KACE call includes Authorization: Bearer <access token>
```

`@shopify/customer-account-api-client` handles the OAuth dance. The env vars the storefront needs are `<CUSTOMER_ACCOUNT_API_CLIENT_ID>` + `<CUSTOMER_ACCOUNT_API_ISSUER_URL>`, both captured from the Dev Dashboard for the PromptKart app.

### Rendering pipeline (loader → workflow → tasks → RenderModel)

```
/products/$handle.tsx
   loader(args)
      │
      ▼
   RenderProductDetailWorkflow
      │  reusable tasks (src/tasks/):
      │    FetchCustomerTokenTask
      │    FetchProductFromStorefrontApiTask
      │    CallKaceEvaluateRulesTask       ← trigger=product.view
      │    ApplyActionsToRenderModelTask   ← converts actions → RenderModel
      │    BuildLoaderResponseTask
      ▼
   { product, renderModel } → <ProductDetail product={product} render={renderModel} />
```

React components are pure given `renderModel`. No conditionals driven by hardcoded business logic.

---

## Planned directory layout

```
app/
├── root.tsx  entry.server.tsx  entry.client.tsx  tailwind.css
│
├── routes/                          Hydrogen/Remix routes
│   ├── _index.tsx                   home
│   ├── products.$handle.tsx         product detail (calls KACE in loader)
│   ├── collections.$handle.tsx
│   ├── cart.tsx
│   ├── account/
│   │   ├── login.tsx                Customer Account API redirect
│   │   ├── callback.tsx             OAuth callback
│   │   ├── logout.tsx
│   │   └── _index.tsx               "My Prompts" + reward balance
│   └── api/kace.$action.tsx         server-side proxy to KACE (optional)
│
├── components/
│   ├── ProductCard, ProductDetail, PriceGate, DiscountPill
│   ├── Cart, RewardBalance
│
└── lib/                             server-side helpers (mirrors backend layout)
    ├── api/loaders/                 product.loader.ts, account.loader.ts
    ├── service/                     interfaces + impl/
    │   ├── kace.service.ts
    │   ├── customer-account.service.ts
    │   └── impl/workflows/
    │       ├── RenderProductDetailWorkflow/…
    │       └── ApplyCartRewardsWorkflow/…
    ├── tasks/                       reusable
    │   ├── FetchCustomerTokenTask.ts
    │   ├── CallKaceEvaluateRulesTask.ts
    │   ├── CallKaceApplyRewardsTask.ts
    │   ├── ApplyActionsToRenderModelTask.ts
    │   └── BuildLoaderResponseTask.ts
    ├── core/                        framework-free
    │   ├── workflow/                Workflow, Task, WorkflowContext
    │   ├── domain/                  RenderModel, KaceAction, Customer
    │   └── action-appliers/         HideFieldApplier, ShowCtaApplier, ApplyDiscountApplier, Registry
    ├── dao/                         storefront.dao.ts, kace.dao.ts + impl/
    ├── dtos/                        request/ + response/
    ├── constants/ utils/
    └── config/                      env.ts, session.ts
```

Same `api/service/core/dao/dtos/constants/utils/config + tasks/workflow` layout as `KACECommerceEngine`, so all repos read as mirror images.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Language | TypeScript (strict) |
| Framework | Hydrogen (Remix-based) |
| Shopify libs | `@shopify/hydrogen`, `@shopify/hydrogen-react`, `@shopify/customer-account-api-client` |
| UI | React + Tailwind CSS |
| Deployment (future) | Shopify Oxygen or Cloudflare Workers |
| Local dev | `shopify hydrogen dev` on port `:3000` |
| Tests (future) | Vitest (unit) + Playwright (storefront smoke) |

---

## Status

⏳ **Pre-scaffold.** Repo currently holds this README, an MIT license, and a `.gitignore`. Hydrogen scaffolding + KACE integration ships in a later phase.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs

Full design docs for this storefront (and the full KACE suite) live in the parent planning folder alongside this project — architecture, per-service READMEs, diagrams, development-plan spreadsheet, and conversation history.
