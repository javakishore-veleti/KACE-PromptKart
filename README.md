# KACE-PromptKart

**Public Hydrogen storefront for selling AI prompt packs on `promptkart-dev.myshopify.com`.**

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. It is a standalone acronym
(treat it like IKEA or NASA — don't re-expand it in every sentence). Historically
the letters came from **K**ishore **A**pps **C**ommerce **E**ngine, but today KACE
is the name of a 4-service suite built on Shopify by
[**Kishore Veleti**](https://github.com/javakishore-veleti) — Shopify Partner org
*Kishore Applications* (Partner ID `4868609`, org id `214691442`).

**The 4 repos in the KACE suite:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/javakishore-veleti/KACECommerceEngine) | TypeScript + Fastify middleware — rule engine facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [**KACE-PromptKart**](https://github.com/javakishore-veleti/KACE-PromptKart) *(this repo)* | Hydrogen public storefront selling AI prompt packs. |
| [KACE-StudyDesk](https://github.com/javakishore-veleti/KACE-StudyDesk) | Hydrogen public storefront selling micro-courses. |
| [KACE-Extended-Rules-Sidecar](https://github.com/javakishore-veleti/KACE-Extended-Rules-Sidecar) | Spring Boot + Drools JVM sidecar. |

---

## What is `KACE-PromptKart` specifically?

A **public Shopify Hydrogen storefront** (React + Remix + Shopify Storefront API) for
selling **AI prompt packs** to customers. Dev URL: `promptkart-dev.myshopify.com`.
Production domain TBD.

### Core UX pattern — guest-vs-logged-in gating

- **Guests** see product title, images, and a teaser description. They do **not** see
  price, variant options, or a buy button.
- **Logged-in customers** (via Shopify Customer Account API / OAuth+OIDC) see the full
  price — including any rule-driven discount — plus the buy button, reward balance, etc.

The gating logic is **not hardcoded**. It is expressed as a `kace_reward_rule`
MetaObject authored in Shopify admin and evaluated by `KACECommerceEngine`. The
storefront simply renders whatever actions come back (e.g. `HIDE_FIELD price`,
`SHOW_CTA "Login to see price & buy"`).

### Responsibilities of this repo

- React/Remix/Hydrogen storefront UI.
- Shopify Storefront API client for public catalog + cart + checkout.
- Shopify Customer Account API client for customer login.
- Thin HTTP client to `KACECommerceEngine` for every business decision.

### What this repo is NOT

- Not a Shopify merchant-installed admin app (no Polaris, no App Bridge, no OAuth
  merchant install).
- Does not call Shopify Admin GraphQL directly. All privileged calls go through
  `KACECommerceEngine`.
- Does not implement SessionStorage. That lives in `KACECommerceEngine`.

### Status

⏳ **Pre-scaffold.** Repo has a placeholder README, MIT license, and `.gitignore`.
Actual Hydrogen scaffolding + integration code ships in a later phase (see the
project-wide Development Plan in `Shopify_Middleware/Development_Plan.xlsx`).

---

## Planned tech stack

| Layer | Choice |
| --- | --- |
| Language | TypeScript (strict) |
| Framework | Hydrogen (Remix-based) |
| Shopify libs | `@shopify/hydrogen`, `@shopify/hydrogen-react`, `@shopify/customer-account-api-client` |
| UI | React + Tailwind CSS |
| Deployment (future) | Shopify Oxygen or Cloudflare Workers |
| Local dev | `shopify hydrogen dev` on port `:3000` |

---

## Planned directory layout

```
app/
├── routes/                # Hydrogen/Remix routes (home, products, cart, account)
├── components/            # ProductCard, ProductDetail, PriceGate, DiscountPill, ...
└── lib/                   # server-side helpers (layered like the backend services)
    ├── api/loaders/
    ├── service/           # interfaces + impl/ workflows
    ├── tasks/             # reusable tasks (FetchCustomerToken, CallKaceEvaluateRules, ...)
    ├── core/              # framework-free render-model domain + action-appliers
    ├── dao/               # storefront + KACE HTTP
    ├── dtos/ constants/ utils/ config/
```

Same `api/service/core/dao/dtos/constants/utils/config + tasks/workflow` layout as
the backend services, so all repos read as mirror images.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs (not in this repo)

Full design lives in the parent planning folder (`Shopify_Middleware/`):

- `README_KACE-PromptKart.md` — full architecture of this storefront
- `README_KACECommerceEngine.md` — the middleware this storefront calls
- `Development_Plan.xlsx` — phase-by-phase roadmap
- `diagrams/` — architecture + mindmap images
