# Safha POS Platform (skeleton)

Folder structure for rebuilding Safha POS as a multi-tenant SaaS:
**.NET 10 modular monolith (clean architecture)** + **React (TypeScript) apps**.
No code yet — every empty folder holds a `.gitkeep`.

One project per layer, with one folder per business module inside each layer.
That keeps the build small (5 projects + 5 test projects) while each module
stays separate.

```text
SafhaPos.Platform/
├── src/
│   ├── server/                         .NET backend
│   │   ├── SafhaPos.Domain/            business rules, references nothing
│   │   │   ├── Common/                 base entity, Money, Result
│   │   │   └── <Module>/               entities, value objects, events of that module
│   │   ├── SafhaPos.Application/       use cases, references Domain only
│   │   │   ├── Common/                 Abstractions (interfaces), Behaviors (validation, logging)
│   │   │   ├── <Module>/Commands       actions that change data
│   │   │   ├── <Module>/Queries        reads
│   │   │   └── Reports/ Notifications/ Integrations/
│   │   ├── SafhaPos.Infrastructure/    implements Application interfaces
│   │   │   ├── Persistence/            Master + Tenant databases (EF Core / MySQL), configurations, migrations
│   │   │   ├── MultiTenancy/           find the restaurant from the domain, open its database
│   │   │   ├── Messaging/ Caching/ Storage/ Auth/
│   │   │   └── Integrations/           Zatca, NamiPay, MyFatoorah, Deliverect, QuickBooks, WhatsApp, Sms, AiGateway
│   │   ├── SafhaPos.Api/               HTTP API + SignalR
│   │   │   ├── Endpoints/<Module>/     one folder of endpoints per module
│   │   │   └── Realtime/ Middleware/ Configuration/
│   │   └── SafhaPos.Worker/            background jobs (ZATCA, Deliverect, QuickBooks, billing)
│   └── client/                         React frontend
│       ├── apps/
│       │   ├── cashier-pos             cashier screen (keyboard-first, works through network blips)
│       │   ├── back-office             menu, stock, reports, settings
│       │   ├── kitchen-display         kitchen / waiter screens
│       │   ├── online-ordering         customer online menu
│       │   └── platform-admin          SaaS owner: companies, plans, subscriptions
│       └── packages/                   shared: ui, api-client, auth, i18n (EN/AR), utils, config
├── tests/
│   ├── SafhaPos.Domain.Tests           rule tests
│   ├── SafhaPos.Application.Tests      use-case tests
│   ├── SafhaPos.Infrastructure.Tests   database / integration tests
│   ├── SafhaPos.Api.Tests              endpoint tests
│   └── SafhaPos.ArchitectureTests      enforces the layer and module rules below
├── deploy/                             docker, nginx, scripts
├── docs/                               architecture, decisions, API, migration notes from PHP
└── tools/data-migration/               scripts to move data from the PHP databases
```

```text
Api ─────┐
Worker ──┼──► Application ──► Domain
Infrastructure ┘   (implements Application interfaces)
```

## Modules

The same module names appear as folders in Domain, Application and Api.

| Module | Covers |
|---|---|
| Identity | users, roles, permissions, login, outlet selection |
| Tenancy | companies, subdomain / custom domain, tenant databases |
| Billing | plans, trials, subscriptions, feature limits, AI credits |
| Catalog | categories, items, modifiers, combos, outlet prices |
| Floor | areas, tables, reservations |
| Orders | place / hold / split / send to kitchen / invoice / refund |
| Payments | cash register, card, NamiPay, MyFatoorah, loyalty |
| Zatca | Saudi e-invoicing (runs after the sale, from a queue) |
| Kitchen | kitchen tickets and live status |
| Inventory | ingredients, recipes, purchases, stock movement, costing |
| Accounting | chart of accounts, vouchers, profit and loss |
| Customers | customers, dues, loyalty points |
| Reports | dashboards and reports (Application queries + Api endpoints only) |

Outside calls live in `Infrastructure/Integrations` (Deliverect, QuickBooks,
WhatsApp, SMS, payments, ZATCA, and `AiGateway`, the client for the existing
pos-ai-service gateway `POST /v1/ai/generate`).

## Ground rules

- Dependencies point inwards: Api / Worker / Infrastructure → Application → Domain. Domain references nothing.
- A module does not use another module's folders directly; it goes through Application interfaces or events. `SafhaPos.ArchitectureTests` checks this.
- Master database (companies, plans) + one database per restaurant — same as today.
- Stateless API (token login), money as `decimal`, times in UTC, idempotency key on order submit and payment.
- Scale by running more API / Worker copies behind a load balancer with Redis — not by adding projects.
- Split a module into its own projects only if it becomes too large or needs separate deployment.
- Stays outside this repo: pos-ai-service (Ask AI + AI gateway), local print agent, WhatsApp device service.
- Migrate one flow at a time (login → cashier path → kitchen → mobile API → back office), reading the same MySQL as the PHP app during the switch.
